# Prometheus/Grafana-Monitoring für die Proxmox-Infrastruktur

Status: Design genehmigt (2026-08-29).

## Ausgangslage

Am 24./25.08.2026 blieb LXC 102 (`paperless`) unbemerkt hängen: Celery
MainProcess/Beat gerieten in Zustand `Ds`, der RAM des Containers (Limit
6144 MB) lief über knapp 5 Tage schleichend auf 99,4 %, Swap (512 MB)
praktisch voll. Erst als `pct exec 102` komplett unresponsive wurde und der
Webserver (Port 8000) nicht mehr erreichbar war, fiel es auf — durch einen
manuell angestoßenen Systemcheck, nicht durch eine Warnung. `pct reboot 102`
behob es. Details siehe Memory `paperless_monitoring.md`.

Als Sofortmaßnahme entstand am selben Tag ein `systemd timer`
(`paperless-health-check.sh`, alle 10 Minuten), der Cgroup-RAM/Swap von
LXC 102 prüft und über einen neuen generischen HA-Webhook
(`automation.proxmox_host_alert_webhook`) eine Mobile-Push schickt. Dieses
Dokument beschreibt den nächsten Schritt: ein richtiges Monitoring für die
**gesamte** Infrastruktur, nicht nur für einen Container per Ad-hoc-Skript.

## Scope

**Phase 1 (dieses Dokument):** Proxmox-Host + alle VMs/LXCs (100–104) —
CPU/RAM/Swap/Disk/Status, zentral über die Proxmox-API. Deckt genau die
Ursache des Vorfalls ab (schleichende Ressourcen-Erschöpfung), nicht nur das
spätere Symptom.

**Explizit spätere Phase 2 (nicht Teil dieser Umsetzung):**
- HTTP-Erreichbarkeitsprüfung einzelner Dienste (`blackbox_exporter`, z.B.
  Paperless-Webserver Port 8000 — das Symptom, das dem eigentlichen Vorfall
  folgte)
- UniFi-Netzwerk (`unpoller`)
- Synology NAS (SNMP)
- Home-Assistant-Entity-Metriken (HA-eigene Prometheus-Integration)

## Zielarchitektur

Zwei neue unprivilegierte Debian-12-LXCs auf dem Proxmox-Host `proxmox`,
Namenskonvention analog zu `ha-mcp`/`paperless-mcp`/`unifi-mcp`:

| LXC | Rolle |
|---|---|
| **105** `monitoring` | Docker Compose: Prometheus, Alertmanager, `pve-exporter`, Grafana |
| **106** `monitoring-mcp` | Docker: Grafanas offizieller `mcp-grafana` |

```
pve-exporter (fragt Proxmox-API ab)
     │  scrape alle 30s
     ▼
Prometheus (TSDB, Alarmregeln)
     │  Regel feuert
     ▼
Alertmanager ──webhook──▶ automation.proxmox_host_alert_webhook (HA)
                                    │
                                    ▼
                          notify.cm_iphone17p (Mobile-Push)

Grafana ◀──API──── monitoring-mcp (LXC 106) ◀──MCP──── Claude
```

### Komponenten im Detail

- **`pve-exporter`** (Docker, `prompve/prometheus-pve-exporter`): fragt
  zentral die Proxmox-API ab, liefert `pve_memory_usage_bytes` /
  `pve_memory_size_bytes` (RAM), `pve_swap_usage_bytes` /
  `pve_swap_size_bytes`, `pve_cpu_usage_ratio`, `pve_disk_usage_bytes`,
  `pve_up` (läuft/gestoppt) — pro Host **und** pro VM/LXC, ohne Agent in den
  überwachten Containern selbst. Config `pve.yml` mit API-Token, `Verify SSL`
  entsprechend `PROXMOX_VERIFY_SSL=false` (selbstsigniertes Zertifikat, wie
  in diesem Repo üblich).
- **Prometheus**: Standard-Image, scrape-Intervall 30s, Retention 30 Tage
  (unkritisch bei 6 Targets × wenigen Dutzend Metriken).
- **Alertmanager**: ein Webhook-Receiver, Ziel
  `http://10.1.0.10:8123/api/webhook/<webhook_id>` (derselbe Webhook wie
  beim bisherigen Skript).
- **Grafana**: nur im LAN erreichbar (Port 3000), kein externer Zugriff.
  Datasource: Prometheus in LXC 105 selbst (`http://localhost:9090`).
- **`mcp-grafana`** (LXC 106): offizielles Grafana-Labs-Projekt, spricht
  Grafana in LXC 105 über die HTTP-API an (`GRAFANA_URL`,
  `GRAFANA_SERVICE_ACCOUNT_TOKEN`). Funktionsumfang: Dashboards/Panels lesen
  **und schreiben** (Service-Account-Rolle `Editor`), Datasources abfragen
  (PromQL-Passthrough auf die Prometheus-Datasource), Alerting-Regeln
  einsehen. Client-seitig eingebunden wie die anderen drei MCP-Server
  (`mcp-remote`-Bridge in `claude_desktop_config.json`).

### Ressourcen

| LXC | vCPU | RAM | Disk |
|---|---|---|---|
| 105 `monitoring` | 2 | 2 GB | 16 GB (`local-lvm`) |
| 106 `monitoring-mcp` | 1 | 1 GB | 8 GB (`local-lvm`) |

+3 GB RAM-Bedarf auf dem Host — im Rahmen dessen, was die letzte
Migrations-Spec (2026-08-20) schon als "im Auge behalten" markiert hatte.

### Secrets

Bewusste Entscheidung (2026-08-29, konsistent mit der Migrations-Spec vom
2026-08-20): Zugangsdaten liegen als Klartext-Env-Var direkt in der
jeweiligen `docker-compose.yml` — kein Vaultwarden-Pull, kein separates
`.env`. Gleiches Muster wie bei `ha-mcp`/`paperless-mcp`/`unifi-mcp`, bewusst
für alle MCP-nahen LXCs einheitlich gehalten.

- **Proxmox-API-Token** für `pve-exporter`: neuer API-User (z.B.
  `pve-exporter@pve`) mit Rolle `PVEAuditor` (nur lesend) — kein Root, kein
  Passwort-Login.
- **Grafana-Service-Account-Token** für `mcp-grafana`: Rolle `Editor`
  (wegen der gewünschten Dashboard-Verwaltung per MCP).

### Alerting

Alertmanager schickt sein natives Webhook-JSON
(`{status, alerts: [{labels, annotations, startsAt, ...}]}`) an
`automation.proxmox_host_alert_webhook`. Die bestehende Automation wird
erweitert: Sie prüft, ob `trigger.json` ein `alerts`-Feld enthält
(Alertmanager-Format) und baut Titel/Nachricht daraus zusammen (Status +
betroffenes Ziel aus `labels`, Kurztext aus `annotations.summary`); fehlt
das Feld, greift wie bisher der einfache `{title, message}`-Pfad. Damit
bleibt der Webhook generisch für künftige direkte Aufrufer nutzbar (siehe
`home-assistant-senior`-Skill, Abschnitt "Generischer
Infrastruktur-Alert-Webhook").

**Alarmregeln Phase 1** (Schwellen identisch zum bisherigen Skript, für
Vergleichbarkeit):
- RAM ≥ 90 % für 5 Min. — pro Host und pro VM/LXC
- Swap ≥ 90 % für 5 Min. — pro Host und pro VM/LXC
- `pve_up == 0` für eine an sich erwartete laufende VM/LXC (Status-Wechsel
  auf "gestoppt")

Jede Regel trägt Name des betroffenen Ziels (z.B. "102 · paperless") in der
`summary`-Annotation, damit die Push-Nachricht ohne Nachschauen verständlich
ist.

### Dashboard-Design

Ein Haupt-Dashboard "Proxmox-Infrastruktur — Übersicht", per
`mcp-grafana`/JSON-Provisioning gebaut (kein Import eines
Community-Dashboards, da individuelles Layout gewünscht). Zwei Bereiche
übereinander, im Brainstorming visuell festgelegt:

1. **Status-Reihe (oben):** eine Kachel pro Host/VM/LXC (6 Kacheln:
   Proxmox-Host, 100 HA, 101 ha-mcp, 102 paperless, 103 paperless-mcp,
   104 unifi-mcp). Grüner/roter linker Rand je nach Alarmzustand (aus
   derselben Schwelle wie die Push-Alarmregeln), Kurzwerte RAM/Swap bzw.
   CPU/RAM beim Host. Auf einen Blick erfassbar, welches Ziel Aufmerksamkeit
   braucht.
2. **Verlaufs-Raster (unten):** ein Zeitreihen-Panel pro Ziel (RAM %,
   zusätzlich Swap % für auffällige Ziele), Standard-Zeitraum 24h, umschaltbar
   auf 7d/30d. Zeigt ohne Klick, ob sich ein Problem plötzlich oder
   schleichend aufgebaut hat — genau die Information, die beim
   Paperless-Vorfall gefehlt hätte, weil es niemand über 5 Tage beobachtet
   hat.

Reihenfolge der Verlaufs-Panels nach Schweregrad (auffällige Ziele zuerst),
sofern in Grafana ohne größeren Aufwand umsetzbar; sonst feste Reihenfolge
nach VMID.

### Migration des alten Skripts

Sobald die Alertmanager-Regeln produktiv sind und mindestens einen echten
Zyklus (10 Min. Vergleichszeitraum) unauffällig durchlaufen haben:
`systemctl disable --now paperless-health-check.timer` auf dem Proxmox-Host.
Datei und Service-Unit bleiben liegen (nicht gelöscht) als Referenz/Fallback,
falls der Monitoring-LXC selbst mal ausfällt und cmuellar das Skript manuell
reaktivieren will.

## Testing / Verifikation

- `pve-exporter`: `curl http://<lxc105-ip>:9221/pve` liefert Metriken für
  Host + alle 5 Gäste
- Prometheus: Targets-Seite zeigt `pve-exporter` als `up`
- Alertmanager: manuell eine Testregel mit `for: 0s` feuern lassen, prüfen
  dass die Push-Nachricht ankommt (analog zum Test des alten Skripts am
  2026-08-29), danach reguläre Regel mit `for: 5m` scharf schalten
- Grafana-Dashboard: Sichtprüfung gegen die im Brainstorming freigegebenen
  Mockups (Status-Reihe + Verlaufs-Raster)
- `mcp-grafana`: über Claude ein Dashboard lesen und eine Kleinigkeit
  ändern (z.B. Panel-Titel), verifizieren dass die Änderung in der
  Grafana-Web-UI ankommt

## Out of scope

Siehe "Scope" oben — `blackbox_exporter`, UniFi, Synology, HA-Entity-Metriken
sind bewusst zurückgestellt auf eine spätere Phase 2, um Phase 1 fokussiert
und in überschaubarer Zeit umsetzbar zu halten.
