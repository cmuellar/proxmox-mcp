# Vaultwarden

Status: Design genehmigt (2026-08-29).

## Ausgangslage

CLAUDE.md in diesem Repo verwies auf einen Vaultwarden unter
`vault.sitecraft-it.com`, der sich als für cmuellar nicht
zugänglich/unbekannt herausstellte (siehe Memory
`feedback-vaultwarden-keine-eingabe`). Als Vorbereitung wurde bereits eine
interne Zertifizierungsstelle (`step-ca`, LXC 108) und ein Reverse-Proxy
(`caddy`, LXC 109) aufgesetzt — siehe
`docs/superpowers/specs/2026-08-29-interne-ca-reverse-proxy-design.md`.
`vault.muellar.org` löst bereits auf die Caddy-LXC auf und zeigt aktuell
auf ein Platzhalter-Backend. Dieses Dokument deckt Vaultwarden selbst ab,
das dieses Platzhalter-Backend ersetzt.

**Scope-Entscheidung aus dem CA-Vorprojekt (gilt weiter):** Nur
Infrastruktur-Secrets (API-Tokens, Service-Passwörter), kein
vollwertiger Passwort-Manager für private/Familien-Logins. Zugriff
hauptsächlich vom PC aus, nicht von unterwegs.

## Zielarchitektur

Eine neue unprivilegierte Debian-12-LXC `vaultwarden` (nächste freie
VMID **110**), Docker-Compose mit dem offiziellen `vaultwarden/server`-
Image. Läuft intern nur über Klartext-HTTP — TLS-Terminierung übernimmt
Caddy (LXC 109) bereits, keine zweite TLS-Schicht nötig.

```
Browser/Bitwarden-App
        │  https://vault.muellar.org
        ▼
   Caddy (LXC 109) ── Zertifikat von step-ca (LXC 108), bereits fertig
        │  reverse_proxy, Klartext-HTTP im LAN
        ▼
   Vaultwarden (LXC 110, neu)
        │
        ▼
   SQLite-Datenbank + Anhänge in /opt/vaultwarden/data (bind-gemountet)
```

**Änderung an bestehender Komponente:** Caddys `Caddyfile`
(`/opt/caddy/Caddyfile` in LXC 109) wird angepasst — der `vault.muellar.org`-
Block wechselt von `respond "Platzhalter..."` auf
`reverse_proxy <vaultwarden-ip>:80`. Sonst keine Änderung an LXC 108/109.

### Konfiguration

- `WEBSOCKET_ENABLED=true` — ohne das kein Live-Sync für Browser-
  Erweiterung/mobile App (reines Polling wäre spürbar langsamer)
- `ADMIN_TOKEN` (generiert, für das `/admin`-Panel — Nutzerverwaltung,
  Diagnose, ohne dass ein normaler Account nötig ist)
- `SIGNUPS_ALLOWED=true` zunächst — cmuellar legt seinen Account über die
  Web-Oberfläche an, danach wird auf `SIGNUPS_ALLOWED=false` umgestellt
  (verhindert dauerhaft, dass sich jemand sonst registrieren kann, selbst
  bei einem Leck der `vault.muellar.org`-URL)
- Secrets als Klartext-Env-Var in der `docker-compose.yml`, konsistent mit
  allen bisherigen Diensten in diesem Projekt

### Backup

`/opt/vaultwarden/data` (SQLite-DB + Anhänge) ist ab Inbetriebnahme die mit
Abstand kritischste Sicherung aller in diesem Projekt neu geschaffenen
Dienste — hier landen perspektivisch alle Secrets. Gleiches Muster wie
beim CA-Schlüssel (siehe Vorprojekt-Spec): Aufnahme von LXC 110 in die
`vzdump`-Liste des bestehenden Sonntags-Backup-Skripts, zusätzlich
gezielter `tar`-Sicherungsschritt für `/opt/vaultwarden/data` auf die
Synology.

## Testing / Verifikation

- Web-Oberfläche unter `https://vault.muellar.org` erreichbar, Zertifikat
  ohne Browser-Warnung (setzt den bereits durchgeführten Root-Zertifikat-
  Import aus dem CA-Vorprojekt voraus)
- Account-Anlage über die Web-Oberfläche, ein Test-Secret anlegen und
  wieder abrufen
- Nach `SIGNUPS_ALLOWED=false`: ein weiterer Registrierungsversuch muss
  von Vaultwarden abgelehnt werden
- Admin-Panel unter `https://vault.muellar.org/admin` mit dem
  `ADMIN_TOKEN` erreichbar
- Backup-Skript: Syntax-Check (`bash -n`) plus isolierter Trockentest des
  neuen `tar`-Schritts, wie beim CA-Schlüssel

## Out of scope

- Mobile Zugriff von unterwegs (externe Erreichbarkeit) — bewusst nicht
  vorgesehen, siehe Scope-Entscheidung oben
- Mehrbenutzer-/Organisations-Setup für Familien-Logins — passt nicht zum
  festgelegten Scope "nur Infrastruktur-Secrets"
- Migration bestehender, an anderer Stelle notierter Secrets nach
  Vaultwarden (z.B. die in dieser Session generierten Grafana-/Proxmox-
  Tokens) — eigener, späterer Schritt, sobald Vaultwarden läuft
