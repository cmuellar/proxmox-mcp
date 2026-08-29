# Vaultwarden Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. **Task 4 requires a manual pause for user action (Master-Passwort-Anlage) — never automate account creation, see Task 4 for why.**

**Goal:** Vaultwarden läuft in einer eigenen LXC, erreichbar unter `https://vault.muellar.org` über den bereits fertigen Caddy-Reverse-Proxy, mit genau einem Account (cmuellar) und dauerhaft gesperrter Registrierung.

**Architecture:** Neue LXC `vaultwarden` (VMID 110), Docker-Compose mit `vaultwarden/server`, Klartext-HTTP intern. Caddys bestehende `vault.muellar.org`-Route wird von einem Platzhalter auf `reverse_proxy` zu dieser LXC umgestellt.

**Tech Stack:** Docker Compose, `vaultwarden/server` (offizielles Image).

## Global Constraints

- Proxmox-Node: `proxmox`. Storage: `local-lvm`. Bridge: `vmbr0`. Template: `local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst`.
- Neue LXC: `--unprivileged 1 --features nesting=1,keyctl=1 --net0 name=eth0,bridge=vmbr0,ip=dhcp --onboot 1`, Docker CE per `get.docker.com`-APT-Repo, identisch zum bisherigen Vorgehen (LXCs 105–109).
- Nächste freie VMID: **110** — verifiziert per `pct list` (101–109 belegt).
- Secrets als Klartext-Env-Var in der `docker-compose.yml`, konsistent mit allen bisherigen Diensten in diesem Projekt.
- Spec: `docs/superpowers/specs/2026-08-29-vaultwarden-design.md`.
- Bestehende Caddy-Konfiguration (`/opt/caddy/Caddyfile` in LXC 109) — aktueller Inhalt:
  ```
  {
  	acme_ca https://ca.muellar.org/acme/acme-1/directory
  	acme_ca_root /certs/root_ca.crt
  }

  vault.muellar.org {
  	respond "Platzhalter-Backend: Caddy + step-ca funktionieren." 200
  }
  ```
- **Niemals selbst ein Master-Passwort für den Vaultwarden-Account setzen oder das Signup-Formular ausfüllen** — das ist Kredentialeingabe in ein Sicherheits-Formular, bleibt cmuellars eigener Schritt (siehe Memory `feedback-vaultwarden-keine-eingabe`).

---

### Task 1: LXC 110 (`vaultwarden`) anlegen + Docker installieren

**Files:** keine lokalen Dateien.

**Interfaces:**
- Produziert: laufender Debian-12-LXC 110 mit Docker CE, IP-Adresse (`<LXC110_IP>`).

- [ ] **Step 1: LXC erstellen**

```bash
pct create 110 local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
  --hostname vaultwarden \
  --unprivileged 1 \
  --features nesting=1,keyctl=1 \
  --cores 1 \
  --memory 1024 \
  --swap 512 \
  --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --onboot 1
pct start 110
```

- [ ] **Step 2: Verifizieren**

```bash
pct status 110
```
Erwartet: `status: running`

- [ ] **Step 3: Docker CE installieren**

```bash
pct exec 110 -- bash -c "
apt-get update && apt-get install -y ca-certificates curl && \
install -m 0755 -d /etc/apt/keyrings && \
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc && \
chmod a+r /etc/apt/keyrings/docker.asc && \
echo 'deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian bookworm stable' > /etc/apt/sources.list.d/docker.list && \
apt-get update && \
apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
"
```

- [ ] **Step 4: IP-Adresse ermitteln, Verzeichnis anlegen**

```bash
pct exec 110 -- hostname -I
pct exec 110 -- mkdir -p /opt/vaultwarden/data
```
Notiere die IP als `<LXC110_IP>` — wird in Task 3 (Caddy) gebraucht.

---

### Task 2: Vaultwarden starten und verifizieren

**Files:**
- Create: `/opt/vaultwarden/docker-compose.yml` (in LXC 110)

**Interfaces:**
- Konsumiert: LXC 110 aus Task 1.
- Produziert: Vaultwarden erreichbar unter `http://<LXC110_IP>:80` (Login-Seite, Web-Vault) und `/admin` (mit `ADMIN_TOKEN`) — Basis für Task 3 (Caddy-Anbindung) und Task 4 (Account-Anlage).

- [ ] **Step 1: `ADMIN_TOKEN` generieren**

```bash
pct exec 110 -- python3 -c "import secrets; print(secrets.token_urlsafe(32))"
```
**Ausgabe notieren** als `<ADMIN_TOKEN>`.

- [ ] **Step 2: `docker-compose.yml` schreiben**

```bash
cat > /tmp/docker-compose.yml << 'EOF'
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    volumes:
      - ./data:/data
    environment:
      - DOMAIN=https://vault.muellar.org
      - WEBSOCKET_ENABLED=true
      - SIGNUPS_ALLOWED=true
      - ADMIN_TOKEN=<ADMIN_TOKEN>
    ports:
      - "80:80"
    restart: unless-stopped
EOF
pct push 110 /tmp/docker-compose.yml /opt/vaultwarden/docker-compose.yml
rm /tmp/docker-compose.yml
```
`<ADMIN_TOKEN>` durch den tatsächlichen Wert aus Step 1 ersetzen.

- [ ] **Step 3: Stack starten**

```bash
pct exec 110 -- bash -c "cd /opt/vaultwarden && docker compose up -d"
```

- [ ] **Step 4: Verifizieren — Login-Seite erreichbar**

```bash
sleep 5
pct exec 110 -- curl -s -o /dev/null -w '%{http_code}\n' http://localhost/
```
Erwartet: `200`. Bei Fehlern zuerst `docker logs vaultwarden` prüfen — insbesondere auf Berechtigungsfehler beim Schreiben nach `/data` (das offizielle `vaultwarden/server`-Image läuft standardmäßig als `root`, sollte also kein Problem wie bei `step-ca` machen; falls doch, `chown` auf die vom Log gemeldete UID/GID anwenden, gleiches Muster wie beim CA-Vorprojekt).

- [ ] **Step 5: Verifizieren — Admin-Panel**

```bash
pct exec 110 -- curl -s -o /dev/null -w '%{http_code}\n' http://localhost/admin/login
```
Erwartet: `200`

---

### Task 3: Caddy auf Vaultwarden umstellen

**Files:**
- Modify: `/opt/caddy/Caddyfile` (in LXC 109)

**Interfaces:**
- Konsumiert: `<LXC110_IP>` aus Task 1.
- Produziert: `https://vault.muellar.org` zeigt auf Vaultwarden statt auf den Platzhalter.

- [ ] **Step 1: `Caddyfile` aktualisieren**

```bash
cat > /tmp/Caddyfile << 'EOF'
{
	acme_ca https://ca.muellar.org/acme/acme-1/directory
	acme_ca_root /certs/root_ca.crt
}

vault.muellar.org {
	reverse_proxy <LXC110_IP>:80
}
EOF
pct push 109 /tmp/Caddyfile /opt/caddy/Caddyfile
rm /tmp/Caddyfile
```
`<LXC110_IP>` durch die tatsächliche IP aus Task 1 ersetzen.

- [ ] **Step 2: Caddy neu laden**

```bash
pct exec 109 -- bash -c "cd /opt/caddy && docker compose restart caddy"
```

- [ ] **Step 3: Verifizieren**

```bash
sleep 5
pct exec 109 -- curl -s --cacert /opt/caddy/certs/root_ca.crt https://vault.muellar.org/ | grep -o "<title>[^<]*</title>"
```
Erwartet: ein `<title>`-Tag mit einem Vaultwarden-typischen Titel (z.B. enthält "Vaultwarden" oder "Bitwarden") — bestätigt, dass Caddy jetzt zu Vaultwarden statt zum Platzhalter durchreicht, mit weiterhin gültigem Zertifikat.

---

### Task 4: Account anlegen (manueller Schritt durch cmuellar)

**Files:** keine.

**Interfaces:**
- Konsumiert: `https://vault.muellar.org` aus Task 3.
- Produziert: ein Vaultwarden-Account für cmuellar — Voraussetzung für Task 5 (Registrierung sperren) und den eigentlichen Nutzen von Vaultwarden.

**Warum das ein manueller Schritt ist:** Die Account-Anlage verlangt ein
Master-Passwort — genau die Art von Sicherheits-Eingabe, die laut
Session-Regel (Memory `feedback-vaultwarden-keine-eingabe`) nie
automatisiert oder von mir selbst ausgefüllt werden darf, auch nicht beim
eigenen neuen Tresor.

- [ ] **Step 1: cmuellar bitten, den Account anzulegen**

Nachricht an cmuellar: "`https://vault.muellar.org` öffnen, 'Create Account'
wählen, E-Mail + Master-Passwort selbst eintragen. Bescheid geben, wenn
fertig — danach sperre ich die Registrierung."

- [ ] **Step 2: Warten auf Bestätigung**

Erst mit Task 5 fortfahren, nachdem cmuellar die Account-Anlage bestätigt
hat. Optionaler Verifikationsschritt in der Zwischenzeit (kein Login-Versuch,
nur Existenzprüfung über das Admin-Panel):

```bash
pct exec 110 -- curl -s -H "Cookie: VW_ADMIN=<ADMIN_TOKEN>" http://localhost/admin/users/overview 2>&1 | grep -o "cmuellar\|@[a-zA-Z0-9.]*" | head -5
```
(Nur informativ — das Admin-Cookie-Format kann je nach Vaultwarden-Version
abweichen; im Zweifel cmuellars Bestätigung im Chat als alleinige Grundlage
nehmen, nicht auf dieses Kommando verlassen.)

---

### Task 5: Registrierung dauerhaft sperren

**Files:**
- Modify: `/opt/vaultwarden/docker-compose.yml` (in LXC 110)

**Interfaces:**
- Konsumiert: bestätigten Account aus Task 4.
- Produziert: `SIGNUPS_ALLOWED=false`, verifiziert über einen abgelehnten Registrierungsversuch.

- [ ] **Step 1: `docker-compose.yml` aktualisieren**

```bash
cat > /tmp/docker-compose.yml << 'EOF'
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    volumes:
      - ./data:/data
    environment:
      - DOMAIN=https://vault.muellar.org
      - WEBSOCKET_ENABLED=true
      - SIGNUPS_ALLOWED=false
      - ADMIN_TOKEN=<ADMIN_TOKEN>
    ports:
      - "80:80"
    restart: unless-stopped
EOF
pct push 110 /tmp/docker-compose.yml /opt/vaultwarden/docker-compose.yml
rm /tmp/docker-compose.yml
pct exec 110 -- bash -c "cd /opt/vaultwarden && docker compose up -d"
```
`<ADMIN_TOKEN>` durch denselben Wert wie in Task 2 ersetzen (unverändert).

- [ ] **Step 2: Verifizieren — Registrierung wird abgelehnt**

```bash
sleep 5
pct exec 110 -- curl -s -X POST http://localhost/api/accounts/register \
  -H "Content-Type: application/json" \
  -d '{"email":"test-sollte-abgelehnt-werden@example.com","masterPasswordHash":"x","key":"x"}'
```
Erwartet: eine Fehlermeldung (z.B. `"Signups are not allowed"` oder
vergleichbar je nach Vaultwarden-Version), **kein** Erfolg — bestätigt, dass
nach diesem Zeitpunkt kein neuer Account mehr angelegt werden kann.

---

### Task 6: Vaultwarden-Daten ins bestehende Backup aufnehmen

**Files:**
- Modify: `/usr/local/sbin/paperless-backup.sh` (auf dem Proxmox-Host)

**Interfaces:**
- Konsumiert: `/opt/vaultwarden/data` in LXC 110 (Task 2).

- [ ] **Step 1: Aktuellen Skriptinhalt sichern**

```bash
cp /usr/local/sbin/paperless-backup.sh /root/paperless-backup.sh.vor-vaultwarden-backup
```

- [ ] **Step 2: `GAESTE`-Liste um LXC 110 erweitern**

```bash
sed -i 's/^GAESTE="100 101 102 108"$/GAESTE="100 101 102 108 110"/' /usr/local/sbin/paperless-backup.sh
grep -n 'GAESTE=' /usr/local/sbin/paperless-backup.sh
```
Erwartet: `GAESTE="100 101 102 108 110"`

- [ ] **Step 3: Neuen Sicherungsblock einfügen**

Zeilennummer der `# --- Gaeste sichern ---`-Markierung ermitteln (kann sich
durch den step-ca-Block aus dem Vorprojekt verschoben haben):

```bash
grep -n '# --- Gaeste sichern' /usr/local/sbin/paperless-backup.sh
```

Den folgenden Block in eine temporäre Datei schreiben und **eine Zeile vor**
der ermittelten Zeilennummer einfügen (mit `sed -i '<N-1>r /tmp/vw_block.txt' ...`,
`<N-1>` = ermittelte Zeilennummer minus 1 — analog zum step-ca-Vorgehen im
CA-Vorprojekt, dort mit Zeile 128 → Einfügen nach Zeile 127):

```bash
cat > /tmp/vw_block.txt << 'BLOCKEOF'
# --- Vaultwarden-Daten sichern ------------------------------------------
# /opt/vaultwarden/data enthaelt die SQLite-Datenbank mit allen Secrets -
# die kritischste Sicherung von allen. Gleiches Muster wie beim
# step-ca-Schluessel: pct exec + tar-Stream direkt auf die NAS.
VAULTWARDEN_SPIEGEL="$MNT/vaultwarden"
mkdir -p "$VAULTWARDEN_SPIEGEL"
log "sichere Vaultwarden-Daten"
pct exec 110 -- tar czf - -C /opt/vaultwarden/data . > "$VAULTWARDEN_SPIEGEL/vaultwarden-$STAND.tar.gz" 2>>"$LOG" \
  && log "Vaultwarden gesichert: $(du -h "$VAULTWARDEN_SPIEGEL/vaultwarden-$STAND.tar.gz" | cut -f1)" \
  || log "WARNUNG: Sichern von Vaultwarden fehlgeschlagen"

ls -1t "$VAULTWARDEN_SPIEGEL"/vaultwarden-*.tar.gz 2>/dev/null | tail -n +$((BEHALTEN+1)) \
  | while read -r f; do log "entferne alte Vaultwarden-Sicherung: $(basename "$f")"; rm -f "$f"; done

BLOCKEOF
```

Danach mit `sed -i '<N-1>r /tmp/vw_block.txt' /usr/local/sbin/paperless-backup.sh`
einfügen (`<N-1>` durch den tatsächlichen Wert ersetzen) und
`rm /tmp/vw_block.txt`.

- [ ] **Step 4: Einfügeposition verifizieren**

```bash
grep -n "Vaultwarden-Daten sichern\|# --- Gaeste sichern" /usr/local/sbin/paperless-backup.sh
```
Erwartet: der Vaultwarden-Block-Kommentar erscheint in einer früheren
Zeile als `# --- Gaeste sichern ---` (nicht danach — sonst falsch
eingefügt, siehe die entsprechende Falle im CA-Vorprojekt-Plan).

- [ ] **Step 5: Syntax-Check**

```bash
bash -n /usr/local/sbin/paperless-backup.sh && echo "Syntax OK"
```
Erwartet: `Syntax OK`

- [ ] **Step 6: Trockentest nur des neuen Blocks**

```bash
pct exec 110 -- tar czf - -C /opt/vaultwarden/data . | wc -c
```
Erwartet: eine Byte-Zahl > 0.

---

## Hinweis zum Admin-Token

Das in Task 2 generierte `<ADMIN_TOKEN>` nach Abschluss aller Tasks an
cmuellar mitteilen (Klartext im Chat, wie bei den vorherigen Projekten) —
wird nur für den Zugriff auf `/admin` gebraucht. Gehört wie alle anderen in
dieser Session generierten Secrets perspektivisch in Vaultwarden selbst
(jetzt, wo es läuft) — das wäre der naheliegende erste Eintrag.
