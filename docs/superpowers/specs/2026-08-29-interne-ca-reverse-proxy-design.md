# Interne CA (step-ca) + Reverse-Proxy (Caddy) für `*.muellar.org`

Status: Design genehmigt (2026-08-29).

## Ausgangslage

CLAUDE.md in diesem Repo verweist auf einen Vaultwarden unter
`vault.sitecraft-it.com` (Organisation "SiteCraft") als zentralen
Secret-Speicher. Beim Versuch, dort drei neue Secrets aus dem
Monitoring-Aufbau (siehe `docs/superpowers/specs/2026-08-29-prometheus-grafana-monitoring-design.md`)
einzutragen, stellte sich heraus: cmuellar hat keinen Zugriff darauf und
wusste nichts von dessen Existenz — vermutlich veraltete/nie fertig
eingerichtete Referenz. Entscheidung: einen neuen, selbst gehosteten
Vaultwarden auf dem eigenen Proxmox aufsetzen.

Vaultwarden braucht für seine Web-Oberfläche technisch HTTPS (WebCrypto
verlangt einen "secure context"). Statt dafür nur ein einzelnes
selbstsigniertes Zertifikat zu bauen, hat cmuellar entschieden, das
"Security-Thema" grundsätzlich zu lösen: eine eigene interne
Zertifizierungsstelle für `*.muellar.org`, wiederverwendbar für Vaultwarden
und künftige interne Dienste. Dieses Dokument deckt **nur** die CA + den
Reverse-Proxy ab — Vaultwarden selbst ist ein eigenes, späteres Vorprojekt
(nutzt am Ende nur ein Zertifikat von hier).

## Scope

**Teil dieses Vorprojekts:**
- Interne CA (`step-ca`) mit ACME-Provisioner, erreichbar unter
  `ca.muellar.org`
- Reverse-Proxy (Caddy), erreichbar für interne HTTPS-Dienste, bezieht
  Zertifikate automatisch von der CA
- DNS-Einträge am UniFi-Gateway (`ca.muellar.org`, vorbereitend
  `vault.muellar.org` für das Vaultwarden-Vorprojekt)
- Verteilung des Root-CA-Zertifikats an cmuellars Geräte (manueller Schritt
  durch cmuellar selbst)
- Aufnahme des CA-Root-Schlüssels in die bestehende Backup-Routine

**Explizit außerhalb dieses Vorprojekts:**
- Vaultwarden selbst (eigene Spec, sobald diese CA steht)
- Rückwirkendes Umziehen bestehender Dienste (Grafana, `paperless-mcp`,
  `unifi-mcp`, `ha-mcp`) hinter den neuen Reverse-Proxy — bleiben wie bisher
  direkt per IP/Port erreichbar
- Externe Erreichbarkeit/öffentliche Zertifikate — beides bleibt rein intern

## Zielarchitektur

Zwei neue unprivilegierte Debian-12-LXCs auf dem Proxmox-Host `proxmox`,
gleiches Muster wie die bisherigen Dienst-LXCs (101–107):

| LXC | Rolle | Erreichbar unter |
|---|---|---|
| `step-ca` (neue VMID) | Zertifizierungsstelle + ACME-Server | `https://ca.muellar.org` (Port 443, selbstterminiert) |
| `caddy` (neue VMID) | Reverse-Proxy für interne HTTPS-Dienste | `https://vault.muellar.org` (später, Port 443) |

```
Browser/Client (PC, Handy)
        │  https://vault.muellar.org
        ▼
   Caddy (LXC "caddy", Port 443/80)
        │  bezieht/erneuert Zertifikat automatisch per ACME
        ▼
   step-ca (LXC "step-ca", Port 443) ── ist die Root of Trust
        │
        ▼
   reverse_proxy → Backend-Dienst (z.B. Vaultwarden-LXC, Port 80, Klartext-HTTP im LAN)
```

**Warum getrennte LXCs statt einer:** Konsistent mit dem in dieser
Infrastruktur etablierten 1-Dienst-pro-Container-Muster (siehe
`paperless`/`paperless-mcp`, `monitoring`/`grafana`). Zusätzlich inhaltlich
sinnvoll getrennt: `step-ca` hält den hochkritischen privaten
CA-Schlüssel — je kleiner die Angriffsfläche um diesen Container, desto
besser.

### Komponenten im Detail

- **`step-ca`** (Docker-Image `smallstep/step-ca`): einmalige
  Initialisierung per `step ca init` (Root- + Intermediate-CA, DNS-Name
  `ca.muellar.org` + LAN-IP als SAN), danach ein ACME-Provisioner
  (`step ca provisioner add acme --type ACME`) für automatische
  Zertifikatsanfragen von Caddy. Läuft selbstterminiert auf Port 443 —
  das erste Zertifikat der CA ist ihr eigenes (self-signed Root), braucht
  also keinen Caddy davor.
- **`caddy`** (Docker-Image `caddy`, eigenes `Caddyfile`): globale
  ACME-CA-Konfiguration zeigt auf `https://ca.muellar.org/acme/acme/directory`,
  Caddy vertraut dafür dem Root-Zertifikat der eigenen CA (als Datei
  gemountet). Fordert für jeden konfigurierten internen Hostnamen
  (zunächst nur `vault.muellar.org`, vorbereitet für spätere) automatisch
  ein Zertifikat an und erneuert es von selbst. Braucht Port 80 erreichbar
  für die HTTP-01-Challenge, mit der `step-ca` die Kontrolle über den
  Hostnamen prüft (rein intern, kein Bezug zum öffentlichen Internet).
- **DNS:** `ca.muellar.org` → IP der `step-ca`-LXC als statischer A-Record
  am UniFi-Gateway, gleiches Muster wie die drei bestehenden
  `*.muellar.org`-Einträge (`paperless`, `paperless-mcp`, `unifi-mcp`).
  `vault.muellar.org` wird vorbereitend als Platzhalter-Eintrag angelegt
  (zeigt schon auf die `caddy`-LXC), auch wenn Vaultwarden selbst noch
  nicht existiert — Caddy kann so ohne weitere DNS-Änderung sofort ein
  Zertifikat dafür beziehen, sobald der Vaultwarden-Dienst folgt.

### Root-Zertifikat verteilen

`step ca root` exportiert das Root-Zertifikat als `.crt`-Datei. Diese Datei
reiche ich cmuellar weiter — der eigentliche Import in den
Windows-Zertifikatsspeicher (`certmgr.msc` → Vertrauenswürdige
Stammzertifizierungsstellen) bleibt bewusst cmuellars eigener Schritt,
analog zur Vaultwarden-Regel: keine Eingaben in Sicherheits-relevante
Systemdialoge durch mich selbst (siehe Memory
`feedback-vaultwarden-keine-eingabe`). Nach dem Import zeigen Browser für
jeden künftigen `*.muellar.org`-Dienst hinter Caddy ein normales
Schloss-Symbol ohne Warnung.

### Secrets

- **CA-Root-Passwort** (beim `step ca init` generiert): liegt als
  Klartext-Datei in der `step-ca`-LXC (Standard-Ablage von `step-ca`
  selbst, `secrets/password.txt`), zusätzlich cmuellar zum Notieren
  mitgeteilt — nicht Teil einer `docker-compose.yml`-Env-Var, da
  `step-ca` es aus einer gemounteten Datei liest.
- Kein Proxmox-API-Token o.Ä. nötig für dieses Vorprojekt (anders als beim
  Monitoring-Stack) — beide Container sprechen nur miteinander und mit dem
  UniFi-Gateway (DNS, manuell eingetragen).

### Backup

Der private CA-Schlüssel ist ab jetzt ein hochkritisches Secret — sein
Verlust bedeutet, dass alle darüber ausgestellten Zertifikate ungültig
würden und intern neu vertraut werden müsste. Aufnahme in die bestehende
Sonntags-Backup-Routine (`paperless-backup.sh`-Muster: `vzdump` für die
LXC selbst sichert zwar die Container-Konfiguration, aber nicht
zwingend performant genug für ein schnelles Restore-Bedürfnis) — konkret:
zusätzlicher `rsync`-Schritt für `/opt/step-ca/` (bzw. das dortige
Docker-Volume) zur Synology, analog zum bestehenden Dokumente-Spiegel bei
Paperless. Details zur genauen Integration in
`/usr/local/sbin/paperless-backup.sh` gehören in die Umsetzungsplanung.

## Testing / Verifikation

- `step-ca`: `curl https://ca.muellar.org/health` liefert `ok` (Zertifikat
  zunächst noch nicht vertraut, `-k` nötig, bis Root importiert ist)
- ACME-Directory erreichbar: `curl -k https://ca.muellar.org/acme/acme/directory`
  liefert die Standard-ACME-Endpunkte als JSON
- Caddy bezieht ein Zertifikat: Log von Caddy prüfen
  (`docker logs caddy`) auf erfolgreichen ACME-Abschluss für
  `vault.muellar.org`, sobald ein Backend dahinter existiert (Platzhalter-
  Backend für den reinen CA-Test genügt, z.B. ein simpler `whoami`-Container)
- Root-Zertifikat-Vertrauen: nach Import in Windows —
  `https://vault.muellar.org` (oder testweise ein anderer konfigurierter
  Name) zeigt ein normales Schloss-Symbol im Browser, keine Warnung

## Out of scope

Siehe "Scope" oben — Vaultwarden selbst und das Migrieren bestehender
Dienste hinter Caddy sind bewusst separate, spätere Schritte.
