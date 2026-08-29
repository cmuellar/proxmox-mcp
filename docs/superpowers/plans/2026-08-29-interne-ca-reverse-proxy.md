# Interne CA (step-ca) + Reverse-Proxy (Caddy) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Eine interne, ACME-fähige Zertifizierungsstelle (`ca.muellar.org`) plus Reverse-Proxy (`caddy`), die künftige interne HTTPS-Dienste (zuerst Vaultwarden) automatisch mit vertrauenswürdigen Zertifikaten versorgt — ohne Browser-Warnungen, ohne öffentliches Internet.

**Architecture:** Zwei neue unprivilegierte Debian-12-LXCs auf dem Proxmox-Host `proxmox`: `step-ca` (Smallstep CA + ACME-Server, Docker) und `caddy` (Reverse-Proxy, Docker, bezieht Zertifikate automatisch per ACME von `step-ca`). DNS für beide über statische UniFi-Gateway-Records. Root-Zertifikat wird für Windows als `.crt`-Datei und für Apple-Geräte als `.mobileconfig`-Profil aufbereitet.

**Tech Stack:** Docker Compose, `smallstep/step-ca`, `caddy` (offizielles Image), UniFi-Network-API (DNS), Python 3 (`.mobileconfig`-Erstellung via `plistlib`).

## Global Constraints

- Proxmox-Node: `proxmox`. Storage: `local-lvm`. Bridge: `vmbr0`. Template: `local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst`.
- Neue LXCs: `--unprivileged 1 --features nesting=1,keyctl=1 --net0 name=eth0,bridge=vmbr0,ip=dhcp --onboot 1`, Docker CE per `get.docker.com`-APT-Repo (`https://download.docker.com/linux/debian bookworm stable`), identisch zum Vorgehen bei den LXCs 105–107.
- Nächste freie VMIDs: **108** (`step-ca`), **109** (`caddy`) — verifiziert per `pct list` (101–107 belegt).
- Secrets als Klartext-Env-Var in der jeweiligen `docker-compose.yml`, konsistent mit allen bisherigen MCP-/Dienst-LXCs in diesem Projekt — keine Ausnahme für die CA.
- Spec: `docs/superpowers/specs/2026-08-29-interne-ca-reverse-proxy-design.md`.
- Bestehende Dienste (Grafana, `paperless-mcp`, `unifi-mcp`, `ha-mcp`) werden NICHT angefasst — nur neue Container.

---

### Task 1: LXC 108 (`step-ca`) anlegen + Docker installieren

**Files:** keine lokalen Dateien — Proxmox-Host-Kommandos über `mcp__proxmox__ssh_execute` / `pct_exec`.

**Interfaces:**
- Produziert: laufender Debian-12-LXC 108 mit Docker CE, IP-Adresse (`<LXC108_IP>`, per DHCP — in Step 4 ermitteln).

- [ ] **Step 1: LXC erstellen**

```bash
pct create 108 local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
  --hostname step-ca \
  --unprivileged 1 \
  --features nesting=1,keyctl=1 \
  --cores 1 \
  --memory 1024 \
  --swap 512 \
  --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --onboot 1
pct start 108
```

- [ ] **Step 2: Verifizieren**

```bash
pct status 108
```
Erwartet: `status: running`

- [ ] **Step 3: Docker CE installieren**

```bash
pct exec 108 -- bash -c "
apt-get update && apt-get install -y ca-certificates curl && \
install -m 0755 -d /etc/apt/keyrings && \
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc && \
chmod a+r /etc/apt/keyrings/docker.asc && \
echo 'deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian bookworm stable' > /etc/apt/sources.list.d/docker.list && \
apt-get update && \
apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
"
```

- [ ] **Step 4: Docker-Version und IP-Adresse ermitteln, Compose-Verzeichnis anlegen**

```bash
pct exec 108 -- docker --version
pct exec 108 -- hostname -I
pct exec 108 -- mkdir -p /opt/step-ca/data
```
Notiere die IP als `<LXC108_IP>` — wird in Task 6 (DNS) gebraucht.

---

### Task 2: `step-ca` initialisieren mit ACME-Provisioner

**Files:**
- Create: `/opt/step-ca/docker-compose.yml` (in LXC 108)

**Interfaces:**
- Konsumiert: LXC 108 aus Task 1.
- Produziert: `step-ca` erreichbar unter `https://<LXC108_IP>:443`, ACME-Directory unter `/acme/acme/directory`, Root-Zertifikat unter `/home/step/certs/root_ca.crt` im Container (persistiert unter `/opt/step-ca/data` in der LXC) — Basis für Task 4 (Caddy vertraut diesem Root) und Task 7/8 (Verteilung).

- [ ] **Step 1: CA-Passwort generieren**

```bash
pct exec 108 -- python3 -c "import secrets; print(secrets.token_urlsafe(24))"
```
**Ausgabe notieren** als `<STEPCA_PASSWORD>` — wird in Step 2 gebraucht und muss cmuellar am Ende mitgeteilt werden (siehe Hinweis am Ende dieses Plans).

- [ ] **Step 2: `docker-compose.yml` schreiben**

Nutzt die offiziellen `DOCKER_STEPCA_INIT_*`-Umgebungsvariablen des `smallstep/step-ca`-Images für automatische, nicht-interaktive Ersteinrichtung (inkl. ACME-Provisioner) beim ersten Start:

```bash
cat > /tmp/docker-compose.yml << 'EOF'
services:
  step-ca:
    image: smallstep/step-ca:latest
    container_name: step-ca
    volumes:
      - ./data:/home/step
    environment:
      - DOCKER_STEPCA_INIT_NAME=muellar.org Interne CA
      - DOCKER_STEPCA_INIT_DNS_NAMES=ca.muellar.org,<LXC108_IP>
      - DOCKER_STEPCA_INIT_ADDRESS=:443
      - DOCKER_STEPCA_INIT_PROVISIONER_NAME=acme
      - DOCKER_STEPCA_INIT_PASSWORD=<STEPCA_PASSWORD>
      - DOCKER_STEPCA_INIT_ACME=true
    ports:
      - "443:443"
    restart: unless-stopped
EOF
pct push 108 /tmp/docker-compose.yml /opt/step-ca/docker-compose.yml
rm /tmp/docker-compose.yml
```
`<LXC108_IP>` und `<STEPCA_PASSWORD>` durch die tatsächlichen Werte aus Task 1 Step 4 bzw. Task 2 Step 1 ersetzen.

- [ ] **Step 3: Stack starten**

```bash
pct exec 108 -- bash -c "cd /opt/step-ca && docker compose up -d"
```

- [ ] **Step 4: Verifizieren — Health-Check und ACME-Directory**

```bash
sleep 5
pct exec 108 -- curl -sk https://localhost/health
echo
pct exec 108 -- curl -sk https://localhost/acme/acme/directory
```
Erwartet: erste Zeile `{"status":"ok"}`, zweite Zeile ein JSON-Objekt mit Schlüsseln wie `newNonce`, `newAccount`, `newOrder` (Standard-ACME-Endpunkte). Bei Fehlern: `docker logs step-ca` prüfen — die genauen `DOCKER_STEPCA_INIT_*`-Variablennamen stammen aus der aktuellen Doku des `smallstep/step-ca`-Images zum Zeitpunkt der Planerstellung; bei Abweichung dort nachschlagen und diesen Schritt entsprechend anpassen (dieser Verifikationsschritt ist bewusst die Absicherung dafür).

- [ ] **Step 5: Root-Zertifikat prüfen**

```bash
pct exec 108 -- cat /opt/step-ca/data/certs/root_ca.crt
```
Erwartet: ein PEM-Zertifikat (`-----BEGIN CERTIFICATE-----...`). Pfad und Existenz sind Voraussetzung für Task 4/7/8.

---

### Task 3: LXC 109 (`caddy`) anlegen + Docker installieren

**Files:** keine lokalen Dateien.

**Interfaces:**
- Produziert: laufender Debian-12-LXC 109 mit Docker CE, IP-Adresse (`<LXC109_IP>`).

- [ ] **Step 1: LXC erstellen**

```bash
pct create 109 local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
  --hostname caddy \
  --unprivileged 1 \
  --features nesting=1,keyctl=1 \
  --cores 1 \
  --memory 1024 \
  --swap 512 \
  --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --onboot 1
pct start 109
```

- [ ] **Step 2: Verifizieren**

```bash
pct status 109
```
Erwartet: `status: running`

- [ ] **Step 3: Docker CE installieren** (identisches Kommando wie Task 1 Step 3, mit `pct exec 109`)

```bash
pct exec 109 -- bash -c "
apt-get update && apt-get install -y ca-certificates curl && \
install -m 0755 -d /etc/apt/keyrings && \
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc && \
chmod a+r /etc/apt/keyrings/docker.asc && \
echo 'deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian bookworm stable' > /etc/apt/sources.list.d/docker.list && \
apt-get update && \
apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
"
```

- [ ] **Step 4: IP-Adresse ermitteln, Compose-Verzeichnis anlegen**

```bash
pct exec 109 -- hostname -I
pct exec 109 -- mkdir -p /opt/caddy/certs
```
Notiere die IP als `<LXC109_IP>` — wird in Task 6 (DNS) gebraucht.

---

### Task 4: Root-Zertifikat der CA in Caddy bereitstellen

**Files:** keine lokalen Dateien — Kopie zwischen zwei LXCs über den Proxmox-Host.

**Interfaces:**
- Konsumiert: Root-Zertifikat aus Task 2 Step 5 (`/opt/step-ca/data/certs/root_ca.crt` in LXC 108).
- Produziert: dieselbe Datei unter `/opt/caddy/certs/root_ca.crt` in LXC 109 — Voraussetzung für Task 5 (`acme_ca_root`-Direktive in Caddy).

- [ ] **Step 1: Zertifikat vom Host aus zwischen den LXCs kopieren**

```bash
pct exec 108 -- cat /opt/step-ca/data/certs/root_ca.crt > /tmp/root_ca.crt
pct push 109 /tmp/root_ca.crt /opt/caddy/certs/root_ca.crt
rm /tmp/root_ca.crt
```

- [ ] **Step 2: Verifizieren**

```bash
pct exec 109 -- head -c 27 /opt/caddy/certs/root_ca.crt
```
Erwartet: `-----BEGIN CERTIFICATE-----`

---

### Task 5: Caddy konfigurieren und Zertifikatsbezug verifizieren

**Files:**
- Create: `/opt/caddy/Caddyfile` (in LXC 109)
- Create: `/opt/caddy/docker-compose.yml` (in LXC 109)

**Interfaces:**
- Konsumiert: ACME-Directory aus Task 2 (`https://ca.muellar.org/acme/acme/directory` — Task 6 richtet den DNS-Namen ein; bis dahin testweise die IP `<LXC108_IP>` verwenden und nach Task 6 auf den Namen umstellen), Root-Zertifikat aus Task 4.
- Produziert: Caddy unter `https://vault.muellar.org` (nach Task 6) mit automatisch bezogenem, gültigem Zertifikat für ein Platzhalter-Backend — Nachweis, dass die komplette ACME-Kette funktioniert, bevor Vaultwarden (eigenes späteres Vorprojekt) dahinter gestellt wird.

- [ ] **Step 1: `Caddyfile` schreiben** (zunächst mit `<LXC108_IP>` statt `ca.muellar.org`, da DNS erst in Task 6 kommt)

```bash
cat > /tmp/Caddyfile << 'EOF'
{
	acme_ca https://<LXC108_IP>/acme/acme/directory
	acme_ca_root /certs/root_ca.crt
}

vault.muellar.org {
	respond "Platzhalter-Backend: Caddy + step-ca funktionieren." 200
}
EOF
pct push 109 /tmp/Caddyfile /opt/caddy/Caddyfile
rm /tmp/Caddyfile
```

- [ ] **Step 2: `docker-compose.yml` schreiben**

```bash
cat > /tmp/docker-compose.yml << 'EOF'
services:
  caddy:
    image: caddy:latest
    container_name: caddy
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./certs:/certs:ro
      - caddy-data:/data
      - caddy-config:/config
    ports:
      - "80:80"
      - "443:443"
    restart: unless-stopped

volumes:
  caddy-data:
  caddy-config:
EOF
pct push 109 /tmp/docker-compose.yml /opt/caddy/docker-compose.yml
rm /tmp/docker-compose.yml
```

- [ ] **Step 3: Stack starten**

```bash
pct exec 109 -- bash -c "cd /opt/caddy && docker compose up -d"
```

- [ ] **Step 4: Verifizieren — Zertifikat erfolgreich bezogen**

```bash
sleep 15
pct exec 109 -- docker logs caddy 2>&1 | grep -iE "certificate obtained|error|failed" | tail -20
```
Erwartet: eine Zeile mit `certificate obtained successfully` (oder vergleichbar) für `vault.muellar.org`, keine `error`/`failed`-Zeilen zum Zertifikatsbezug. Bei Fehlern: prüfen, ob Port 80 von `step-ca` aus erreichbar ist (`pct exec 108 -- curl -s http://<LXC109_IP>/`, sollte auf die HTTP-01-Challenge-Antwort oder einen Redirect treffen) — die HTTP-01-Challenge von `step-ca` an Caddy braucht funktionierendes Routing zwischen den beiden LXCs im selben `vmbr0`-Netz, was hier standardmäßig der Fall ist.

- [ ] **Step 5: Verifizieren — Antwort über HTTPS mit dem richtigen Zertifikat**

```bash
pct exec 109 -- curl -s --cacert /opt/caddy/certs/root_ca.crt https://vault.muellar.org/ --resolve vault.muellar.org:443:127.0.0.1
```
Erwartet: `Platzhalter-Backend: Caddy + step-ca funktionieren.` — bestätigt, dass das von `step-ca` ausgestellte Zertifikat vom Root-Zertifikat aus Task 4 als gültig anerkannt wird (der komplette Vertrauens-Kreislauf).

---

### Task 6: DNS-Einträge am UniFi-Gateway

**Files:** keine — über `mcp__unifi-network__unifi_execute` mit Tool `unifi_create_dns_record`.

**Interfaces:**
- Konsumiert: `<LXC108_IP>` (Task 1), `<LXC109_IP>` (Task 3).
- Produziert: `ca.muellar.org` und `vault.muellar.org` als A-Records, aufgelöst wie die drei bestehenden `*.muellar.org`-Einträge.

- [ ] **Step 1: Bestehende Einträge zur Orientierung prüfen**

```python
mcp__unifi-network__unifi_execute(tool="unifi_list_dns_records", arguments={})
```
Erwartet: die drei bekannten Einträge (`paperless.muellar.org`, `paperless-mcp.muellar.org`, `unifi-mcp.muellar.org`), keiner davon `ca` oder `vault`.

- [ ] **Step 2: `ca.muellar.org` anlegen**

```python
mcp__unifi-network__unifi_execute(
    tool="unifi_create_dns_record",
    arguments={"key": "ca.muellar.org", "value": "<LXC108_IP>", "record_type": "A"}
)
```

- [ ] **Step 3: `vault.muellar.org` anlegen**

```python
mcp__unifi-network__unifi_execute(
    tool="unifi_create_dns_record",
    arguments={"key": "vault.muellar.org", "value": "<LXC109_IP>", "record_type": "A"}
)
```

- [ ] **Step 4: Verifizieren**

```bash
pct exec 105 -- getent hosts ca.muellar.org
pct exec 105 -- getent hosts vault.muellar.org
```
(LXC 105 hat DNS über denselben UniFi-Resolver wie alle anderen Container.) Erwartet: jeweils `<LXC108_IP>` bzw. `<LXC109_IP>`.

- [ ] **Step 5: Caddyfile von IP auf DNS-Namen umstellen**

```bash
cat > /tmp/Caddyfile << 'EOF'
{
	acme_ca https://ca.muellar.org/acme/acme/directory
	acme_ca_root /certs/root_ca.crt
}

vault.muellar.org {
	respond "Platzhalter-Backend: Caddy + step-ca funktionieren." 200
}
EOF
pct push 109 /tmp/Caddyfile /opt/caddy/Caddyfile
rm /tmp/Caddyfile
pct exec 109 -- bash -c "cd /opt/caddy && docker compose restart caddy"
```

- [ ] **Step 6: Erneut verifizieren (Task 5 Step 4/5 wiederholen)**

```bash
sleep 15
pct exec 109 -- docker logs caddy --since 20s 2>&1 | grep -iE "certificate obtained|error|failed"
pct exec 109 -- curl -s --cacert /opt/caddy/certs/root_ca.crt https://vault.muellar.org/
```
Erwartet: gleiches Ergebnis wie zuvor, jetzt über den echten DNS-Namen statt der IP.

---

### Task 7: Root-Zertifikat für Windows aufbereiten

**Files:**
- Create: lokal `root_ca.crt` (Scratchpad dieser Session)

**Interfaces:**
- Konsumiert: `/opt/step-ca/data/certs/root_ca.crt` aus Task 2.
- Produziert: lokale Datei, die dem Nutzer per `SendUserFile` übergeben wird.

- [ ] **Step 1: Zertifikatsinhalt vom Proxmox-Host lesen**

```
mcp__proxmox__ssh_read_file(node="proxmox", path="/opt/step-ca/data/certs/root_ca.crt")
```
(Erreicht die Datei innerhalb der LXC über den Proxmox-Host — bei Bedarf zuerst `pct exec 108 -- cat ... > /tmp/root_ca.crt` auf dem Host zwischenlegen, dann von dort lesen, analog zu Task 4 Step 1.)

- [ ] **Step 2: Lokal speichern**

Mit dem `Write`-Tool den gelesenen Inhalt nach
`C:\Users\cmuellar\AppData\Local\Temp\claude\C--Users-cmuellar-mcp-servers-proxmox-mcp\4251abbe-511a-4ac4-9cb1-0ac7a3467b92\scratchpad/root_ca.crt` schreiben (exakter Pfad: siehe Abschnitt
"Scratchpad Directory" der Session).

- [ ] **Step 3: An den Nutzer übergeben**

```
SendUserFile(files=["C:\Users\cmuellar\AppData\Local\Temp\claude\C--Users-cmuellar-mcp-servers-proxmox-mcp\4251abbe-511a-4ac4-9cb1-0ac7a3467b92\scratchpad/root_ca.crt"], status="normal",
  caption="Root-Zertifikat der internen CA — Anleitung zum Import in Windows folgt im Chat (certmgr.msc, Speicherort 'Lokaler Computer' → 'Vertrauenswürdige Stammzertifizierungsstellen').")
```

---

### Task 8: `.mobileconfig`-Profil für Apple-Geräte bauen

**Files:**
- Create: lokal `muellar-ca.mobileconfig` (Scratchpad dieser Session)

**Interfaces:**
- Konsumiert: Root-Zertifikat aus Task 7 Step 2 (lokale `root_ca.crt`).
- Produziert: lokale `.mobileconfig`-Datei, per `SendUserFile` übergeben.

- [ ] **Step 1: PEM zu DER konvertieren und Base64-kodieren** (auf dem Proxmox-Host, `openssl` ist dort vorhanden)

```bash
pct exec 108 -- bash -c "openssl x509 -in /opt/step-ca/data/certs/root_ca.crt -outform der | base64 -w0"
```
**Ausgabe notieren** als `<CERT_BASE64>`.

- [ ] **Step 2: `.mobileconfig` lokal per Python erzeugen**

```python
import plistlib
import uuid

cert_base64 = "<CERT_BASE64>"  # aus Step 1
cert_der = __import__("base64").b64decode(cert_base64)

profile = {
    "PayloadContent": [
        {
            "PayloadCertificateFileName": "root_ca.crt",
            "PayloadContent": cert_der,
            "PayloadDescription": "Fuegt das Root-Zertifikat der internen muellar.org-CA hinzu.",
            "PayloadDisplayName": "muellar.org Root CA",
            "PayloadIdentifier": "org.muellar.ca.root",
            "PayloadType": "com.apple.security.root",
            "PayloadUUID": str(uuid.uuid4()),
            "PayloadVersion": 1,
        }
    ],
    "PayloadDescription": "Installiert das Root-Zertifikat der internen muellar.org Zertifizierungsstelle fuer HTTPS ohne Warnung.",
    "PayloadDisplayName": "muellar.org interne CA",
    "PayloadIdentifier": "org.muellar.ca.profile",
    "PayloadRemovalDisallowed": False,
    "PayloadType": "Configuration",
    "PayloadUUID": str(uuid.uuid4()),
    "PayloadVersion": 1,
}

with open("C:\Users\cmuellar\AppData\Local\Temp\claude\C--Users-cmuellar-mcp-servers-proxmox-mcp\4251abbe-511a-4ac4-9cb1-0ac7a3467b92\scratchpad/muellar-ca.mobileconfig", "wb") as f:
    plistlib.dump(profile, f)
```
Ausführen z.B. per `python3 -c "..."` lokal oder als kurzes Skript — `plistlib.dump` erzeugt automatisch korrektes XML inkl. Base64-Kodierung der `PayloadContent`-`data`, keine manuelle XML-Bastelei nötig.

- [ ] **Step 3: Verifizieren**

```bash
python3 -c "import plistlib; d = plistlib.load(open('C:\Users\cmuellar\AppData\Local\Temp\claude\C--Users-cmuellar-mcp-servers-proxmox-mcp\4251abbe-511a-4ac4-9cb1-0ac7a3467b92\scratchpad/muellar-ca.mobileconfig', 'rb')); print(d['PayloadDisplayName'], len(d['PayloadContent']), 'Payload(s)')"
```
Erwartet: `muellar.org interne CA 1 Payload(s)`

- [ ] **Step 4: An den Nutzer übergeben**

```
SendUserFile(files=["C:\Users\cmuellar\AppData\Local\Temp\claude\C--Users-cmuellar-mcp-servers-proxmox-mcp\4251abbe-511a-4ac4-9cb1-0ac7a3467b92\scratchpad/muellar-ca.mobileconfig"], status="normal",
  caption="Konfigurationsprofil fuer iOS/iPadOS/macOS — nach Installation zusaetzlich unter Einstellungen -> Allgemein -> Info -> Zertifikatsvertrauenseinstellungen manuell 'Volles Vertrauen' aktivieren (Apple-Vorgabe, nicht automatisierbar).")
```

---

### Task 9: CA-Schlüssel ins bestehende Backup aufnehmen

**Files:**
- Modify: `/usr/local/sbin/paperless-backup.sh` (auf dem Proxmox-Host)

**Interfaces:**
- Konsumiert: `/opt/step-ca/data` in LXC 108 (Task 2).

- [ ] **Step 1: Aktuellen Skriptinhalt sichern (Referenz für den Diff)**

```bash
cp /usr/local/sbin/paperless-backup.sh /root/paperless-backup.sh.vor-stepca-backup
```

- [ ] **Step 2: `GAESTE`-Liste um LXC 108 erweitern**

In `/usr/local/sbin/paperless-backup.sh` die Zeile
```bash
GAESTE="100 101 102"
```
ersetzen durch
```bash
GAESTE="100 101 102 108"
```
(sichert LXC 108 zusätzlich per `vzdump`, wie die anderen kritischen Gäste — sinnvoll als vollständiges Container-Restore, ergänzt die im nächsten Schritt hinzugefügte gezielte Sicherung des CA-Datenverzeichnisses.)

- [ ] **Step 3: Neuen Sicherungsblock für `/opt/step-ca/data` einfügen**

Direkt nach dem bestehenden Block `# --- Dokumenten-Spiegel: ...` (vor `# --- Gaeste sichern ---`) folgenden Abschnitt einfügen:

```bash
# --- step-ca Root-Schluessel sichern -----------------------------------
# /opt/step-ca liegt nur im LXC-Dateisystem (kein ZFS-Bind-Mount wie bei
# Paperless) - deshalb per pct exec direkt als tar-Stream auf die NAS,
# statt wie beim Dokumenten-Spiegel einen Host-Pfad zu rsyncen.
STEPCA_SPIEGEL="$MNT/step-ca"
mkdir -p "$STEPCA_SPIEGEL"
log "sichere step-ca Root-Schluessel"
pct exec 108 -- tar czf - -C /opt/step-ca/data . > "$STEPCA_SPIEGEL/step-ca-$STAND.tar.gz" 2>>"$LOG" \
  && log "step-ca gesichert: $(du -h "$STEPCA_SPIEGEL/step-ca-$STAND.tar.gz" | cut -f1)" \
  || log "WARNUNG: Sichern von step-ca fehlgeschlagen"

ls -1t "$STEPCA_SPIEGEL"/step-ca-*.tar.gz 2>/dev/null | tail -n +$((BEHALTEN+1)) \
  | while read -r f; do log "entferne alte step-ca-Sicherung: $(basename "$f")"; rm -f "$f"; done
```

Die vollständige Datei über `mcp__proxmox__ssh_write_file` (Pfad `/usr/local/sbin/paperless-backup.sh`, `backup: true` — Standard, legt automatisch eine Sicherung des Vorzustands an) neu schreiben, mit beiden Änderungen (Step 2 + dieser Block) angewendet.

- [ ] **Step 4: Syntax-Check**

```bash
bash -n /usr/local/sbin/paperless-backup.sh && echo "Syntax OK"
```
Erwartet: `Syntax OK`

- [ ] **Step 5: Trockentest nur des neuen Blocks** (ohne die ganze NAS-Weck-Routine anzustoßen)

```bash
pct exec 108 -- tar czf - -C /opt/step-ca/data . | wc -c
```
Erwartet: eine Byte-Zahl > 0 (bestätigt, dass der `tar`-Befehl im echten Skript funktionieren würde, ohne einen vollständigen Backup-Lauf samt NAS-Aufwecken auszulösen).

---

## Hinweis zum CA-Passwort

Das in Task 2 Step 1 generierte `<STEPCA_PASSWORD>` nach Abschluss aller
Tasks an cmuellar mitteilen (Klartext im Chat, wie beim Grafana-Passwort
im Monitoring-Projekt) — es wird nur für administrative CA-Operationen
gebraucht (z.B. `step ca provisioner add` für einen weiteren
Provisioner), nicht für den laufenden Zertifikatsbezug durch Caddy.
Gehört wie die übrigen in dieser Session generierten Secrets perspektivisch
in einen Passwort-Tresor — aktuell keiner verfügbar (siehe Memory
`feedback-vaultwarden-keine-eingabe`), daher zunächst nur im Chat
festgehalten.
