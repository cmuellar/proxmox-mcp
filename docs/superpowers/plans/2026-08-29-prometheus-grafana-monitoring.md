# Prometheus/Grafana-Monitoring + monitoring-mcp Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Proxmox-Host + alle VMs/LXCs (100-104) über Prometheus/Grafana überwachen, mit Alarmierung über den bestehenden HA-Webhook und einem `monitoring-mcp` für Claude-Zugriff auf Grafana.

**Architecture:** Zwei neue unprivilegierte Debian-12-LXCs auf dem Proxmox-Host `proxmox`. LXC 105 (`monitoring`) trägt einen Docker-Compose-Stack aus `pve-exporter` (fragt die Proxmox-API ab), Prometheus (TSDB + Alarmregeln), Alertmanager (schickt Alarme als Webhook an Home Assistant) und Grafana (Dashboards). LXC 106 (`monitoring-mcp`) trägt Grafanas offiziellen `mcp-grafana` und spricht Grafana in LXC 105 über dessen HTTP-API an.

**Tech Stack:** Docker Compose, Prometheus, Alertmanager, `prometheus-pve-exporter` (Docker-Image `prompve/prometheus-pve-exporter`), Grafana, `mcp-grafana` (Grafana Labs), Home Assistant (Automation-Update über den `homeassistant`-MCP).

## Abweichungen bei der Ausführung (2026-08-29)

**Architekturänderung (auf Wunsch von cmuellar, nach Task 8):** Grafana läuft
NICHT im `monitoring`-Compose-Stack (LXC 105) mit, sondern in einer eigenen
neuen LXC **107** (`grafana`), analog zum bestehenden 1-Dienst-pro-LXC-Muster
(wie `paperless`/`paperless-mcp` getrennt sind). LXC 105 enthält damit nur
noch `pve-exporter` + Prometheus + Alertmanager. Grafanas Prometheus-Datasource
zeigt auf die IP von LXC 105 (`http://10.1.0.133:9090`) statt auf den
internen Compose-Netzwerknamen `prometheus`, da beide jetzt in getrennten
Containern laufen. **Task 10 (`monitoring-mcp`) muss entsprechend
`GRAFANA_URL=http://10.1.0.123:3000` verwenden** (LXC 107s IP), nicht die
IP von LXC 105.

Bei Task 6 festgestellt: `notify.cm_iphone17p` als **Service-Name** existiert
nicht (mehr) — korrekt ist der Service `notify.send_message` mit
`target: {entity_id: notify.cm_iphone17p}`. Zusätzlich akzeptiert
`notify.send_message` **kein** verschachteltes `data.data.push.*`-Feld (die
`interruption-level`-Option aus den älteren `notify.<target>`-Diensten) —
führt zu `extra keys not allowed @ data['data']`. Fix: nur `title`/`message`
im `data`-Block, keine Push-Zusatzoptionen. Über drei Testalarme
(`MonitoringTest`/`2`/`3`/`4`) end-to-end verifiziert, mit `ha_get_automation_traces`
den tatsächlichen Ausführungsfehler gefunden statt nur `last_triggered` zu
prüfen (das aktualisiert sich auch bei einem Automation-Lauf, der intern
mit Fehler abbricht).

Bei Task 3 festgestellt: `prometheus-pve-exporter` liefert **keine
Swap-Metriken** (Proxmox' Cluster-Resources-API, auf der der Exporter
aufbaut, gibt Swap nicht her — nur `pve_cpu_usage_ratio`,
`pve_memory_usage_bytes`/`_size_bytes`, `pve_disk_*`, `pve_up`,
`pve_uptime_seconds` u.a.). Mit cmuellar abgestimmt: Swap-Regel
(`HohesSwap`) und Swap-Panels entfallen für Phase 1, RAM-Überwachung bleibt
wie geplant. Zweite Abweichung: Der Proxmox-Hostname `proxmox` ist aus den
Containern heraus nicht auflösbar — der Exporter braucht als `target` die
tatsächliche IP `10.1.0.20` (Bridge `vmbr0` auf dem Host), nicht den
Hostnamen.

## Global Constraints

- Proxmox-Node: `proxmox`. Storage-Pool für Root-FS: `local-lvm`. Netzwerk-Bridge: `vmbr0`. LXC-Template: `local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst`.
- Neue LXCs unprivilegiert (`--unprivileged 1`), `--features nesting=1,keyctl=1` (Docker-Voraussetzung, siehe LXC 103), `--net0 name=eth0,bridge=vmbr0,ip=dhcp`, `--onboot 1`.
- Docker-Installation über das `get.docker.com`-Convenience-Script, APT-Repo `https://download.docker.com/linux/debian bookworm stable` (gleiches Muster wie LXC 101/103/104).
- Secrets (Proxmox-API-Token, Grafana-Service-Account-Token) liegen als Klartext-Env-Var in der jeweiligen `docker-compose.yml` — kein Vaultwarden-Pull, kein separates `.env` (bewusste, projektweite Entscheidung, siehe Design-Spec Abschnitt "Secrets").
- HA-Webhook-URL: `http://10.1.0.10:8123/api/webhook/8e9441363ecf659a62ea776e1afd2ef6` (bestehende Automation `automation.proxmox_host_alert_webhook`).
- Alle neuen Container-/Service-Namen und Compose-Dateien folgen dem Stil aus `/opt/paperless-mcp/docker-compose.yml`: `services:` → `<name>: image / container_name / restart: unless-stopped`.
- Design-Spec: `docs/superpowers/specs/2026-08-29-prometheus-grafana-monitoring-design.md`.

---

### Task 1: LXC 105 (`monitoring`) anlegen + Docker installieren

**Files:**
- Keine lokalen Dateien — Proxmox-Host- und LXC-105-seitige Shell-Kommandos über `mcp__proxmox__ssh_execute` / `mcp__proxmox__pct_exec`.

**Interfaces:**
- Produziert: laufender, per `pct exec 105` erreichbarer Debian-12-LXC mit Docker CE, IP-Adresse (per DHCP, wird in Step 4 ermittelt und für alle Folgetasks als `<LXC105_IP>` referenziert).

- [ ] **Step 1: LXC 105 erstellen**

```bash
pct create 105 local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
  --hostname monitoring \
  --unprivileged 1 \
  --features nesting=1,keyctl=1 \
  --cores 2 \
  --memory 2048 \
  --swap 512 \
  --rootfs local-lvm:16 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --onboot 1
pct start 105
```

Ausführen über `mcp__proxmox__ssh_execute` (node=`proxmox`).

- [ ] **Step 2: Verifizieren, dass der Container läuft**

```bash
pct status 105
```
Erwartet: `status: running`

- [ ] **Step 3: Docker CE installieren (get.docker.com-Muster)**

```bash
pct exec 105 -- bash -c "
apt-get update && apt-get install -y ca-certificates curl && \
install -m 0755 -d /etc/apt/keyrings && \
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc && \
chmod a+r /etc/apt/keyrings/docker.asc && \
echo 'deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian bookworm stable' > /etc/apt/sources.list.d/docker.list && \
apt-get update && \
apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
"
```

- [ ] **Step 4: Docker-Version und IP-Adresse ermitteln**

```bash
pct exec 105 -- docker --version
pct exec 105 -- hostname -I
```
Erwartet: eine Docker-Version wird ausgegeben, eine IP im Bereich `10.1.0.0/24`. Notiere die IP — sie wird in den folgenden Tasks als `<LXC105_IP>` verwendet.

- [ ] **Step 5: Compose-Verzeichnis anlegen**

```bash
pct exec 105 -- mkdir -p /opt/monitoring/prometheus /opt/monitoring/alertmanager /opt/monitoring/pve-exporter /opt/monitoring/grafana/provisioning/datasources /opt/monitoring/grafana/provisioning/dashboards
```

---

### Task 2: Proxmox-API-Token für `pve-exporter` (Rolle `PVEAuditor`)

**Files:**
- Keine — Proxmox-Host-seitige `pveum`-Kommandos.

**Interfaces:**
- Produziert: API-User `pve-exporter@pve`, Token-ID `pve-exporter@pve!monitoring`, Token-Secret (wird in Task 3 in `pve.yml` eingetragen).

- [ ] **Step 1: API-User anlegen**

```bash
pveum user add pve-exporter@pve --comment "Read-only fuer prometheus-pve-exporter"
```

- [ ] **Step 2: Rolle `PVEAuditor` zuweisen (nur lesend, kein Passwort-Login)**

```bash
pveum acl modify / --users pve-exporter@pve --roles PVEAuditor
```

- [ ] **Step 3: API-Token erzeugen**

```bash
pveum user token add pve-exporter@pve monitoring --privsep 0
```
Erwartet: JSON-Ausgabe mit `"value"` = Token-Secret (UUID-Format). **Diesen Wert notieren** — er wird nur einmal angezeigt und in Task 3 gebraucht.

- [ ] **Step 4: Verifizieren, dass der Token nur lesend darf**

```bash
pveum user permissions pve-exporter@pve
```
Erwartet: Zeile mit `PVEAuditor` und ausschließlich Audit-Berechtigungen (kein `VM.Config.*`, kein `VM.PowerMgmt`), Pfad `/`.

---

### Task 3: `pve-exporter` als Compose-Service

**Files:**
- Create: `/opt/monitoring/pve-exporter/pve.yml` (in LXC 105)
- Create: `/opt/monitoring/docker-compose.yml` (in LXC 105)

**Interfaces:**
- Konsumiert: Token aus Task 2 (`pve-exporter@pve!monitoring`, Secret aus Step 3).
- Produziert: `pve-exporter` erreichbar unter `http://<LXC105_IP>:9221/pve?target=proxmox`, liefert Metriken `pve_up`, `pve_memory_usage_bytes`, `pve_memory_size_bytes`, `pve_swap_usage_bytes`, `pve_swap_size_bytes`, `pve_cpu_usage_ratio`, `pve_disk_usage_bytes`, `pve_disk_size_bytes` mit Label `id` (z.B. `node/proxmox`, `lxc/102`, `qemu/100`) — Basis für Tasks 4/5/8.

- [ ] **Step 1: `pve.yml` schreiben** (Token-Secret aus Task 2 Step 3 einsetzen)

Ziel ist ein Pfad **im LXC 105**, nicht auf dem Proxmox-Host — deshalb Datei zuerst im Host-`/tmp` erzeugen und per `pct push` in den Container kopieren. Ausführen über `mcp__proxmox__ssh_execute` (node=`proxmox`):

```bash
cat > /tmp/pve.yml << 'EOF'
default:
    user: pve-exporter@pve
    token_name: monitoring
    token_value: <TOKEN_SECRET_AUS_TASK_2>
    verify_ssl: false
EOF
pct push 105 /tmp/pve.yml /opt/monitoring/pve-exporter/pve.yml
rm /tmp/pve.yml
```
Ausführen über `mcp__proxmox__ssh_execute` (node=`proxmox`).

- [ ] **Step 2: `docker-compose.yml` schreiben (erster Service)**

```bash
cat > /tmp/docker-compose.yml << 'EOF'
services:
  pve-exporter:
    image: prompve/prometheus-pve-exporter:latest
    container_name: pve-exporter
    volumes:
      - ./pve-exporter/pve.yml:/etc/pve.yml:ro
    environment:
      - PVE_EXPORTER_CONFIG=/etc/pve.yml
    ports:
      - "9221:9221"
    restart: unless-stopped
EOF
pct push 105 /tmp/docker-compose.yml /opt/monitoring/docker-compose.yml
rm /tmp/docker-compose.yml
```

- [ ] **Step 3: Stack starten**

```bash
pct exec 105 -- bash -c "cd /opt/monitoring && docker compose up -d"
```

- [ ] **Step 4: Verifizieren — Metriken abrufen**

```bash
pct exec 105 -- curl -s "http://localhost:9221/pve?target=proxmox" | grep -E "^pve_(up|memory_usage_bytes|memory_size_bytes|swap_usage_bytes|swap_size_bytes)" | head -20
```
Erwartet: Zeilen wie `pve_memory_usage_bytes{id="lxc/102",...} 1.6e+09` für jedes der Ziele `node/proxmox`, `qemu/100`, `lxc/101`..`lxc/104`. Falls das Image `prompve/prometheus-pve-exporter` nicht zieht oder die Endpoint-Struktur abweicht: `docker logs pve-exporter` prüfen, ggf. Image-Tag/Config gegen die aktuelle Projekt-Doku (github.com/prometheus-pve/prometheus-pve-exporter) korrigieren — dieser Schritt ist die Absicherung dafür.

- [ ] **Step 5: Commit-Notiz** (kein Git-Repo für LXC-Configs — stattdessen Eintrag ins Setup-Wissen)

Kein Commit nötig (Proxmox-Host-seitige Config, kein Repo). Weiter zu Task 4.

---

### Task 4: Prometheus — Scrape-Config gegen `pve-exporter`

**Files:**
- Create: `/opt/monitoring/prometheus/prometheus.yml` (in LXC 105)
- Modify: `/opt/monitoring/docker-compose.yml` (in LXC 105) — Prometheus-Service ergänzen

**Interfaces:**
- Konsumiert: `pve-exporter`-Container aus Task 3 (Compose-Service-Name `pve-exporter`, intern Port 9221).
- Produziert: Prometheus erreichbar unter `http://<LXC105_IP>:9090`, Target `pve` mit Status `up` — Basis für Task 5 (Alarmregeln) und Task 7 (Grafana-Datasource).

- [ ] **Step 1: `prometheus.yml` schreiben**

Proxmox-VE-API des Hosts selbst läuft auf `10.1.0.20:8006` (ermittelt in der Vorbereitung, siehe Bridge-IP). Das Relabeling-Muster folgt der offiziellen `prometheus-pve-exporter`-Doku (Multi-Target via `__param_target`):

```bash
cat > /tmp/prometheus.yml << 'EOF'
global:
  scrape_interval: 30s

rule_files:
  - /etc/prometheus/alert_rules.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]

scrape_configs:
  - job_name: 'pve'
    static_configs:
      - targets:
          - proxmox
    metrics_path: /pve
    params:
      target: [proxmox]
    relabel_configs:
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: pve-exporter:9221
EOF
pct push 105 /tmp/prometheus.yml /opt/monitoring/prometheus/prometheus.yml
rm /tmp/prometheus.yml
```

- [ ] **Step 2: Prometheus-Service in `docker-compose.yml` ergänzen**

Datei komplett neu schreiben (Task 3 + Task 4 Services), über `mcp__proxmox__ssh_execute` mit `pct push`:

```bash
cat > /tmp/docker-compose.yml << 'EOF'
services:
  pve-exporter:
    image: prompve/prometheus-pve-exporter:latest
    container_name: pve-exporter
    volumes:
      - ./pve-exporter/pve.yml:/etc/pve.yml:ro
    environment:
      - PVE_EXPORTER_CONFIG=/etc/pve.yml
    ports:
      - "9221:9221"
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/alert_rules.yml:/etc/prometheus/alert_rules.yml:ro
      - prometheus-data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.retention.time=30d"
    ports:
      - "9090:9090"
    restart: unless-stopped
    depends_on:
      - pve-exporter

volumes:
  prometheus-data:
EOF
pct push 105 /tmp/docker-compose.yml /opt/monitoring/docker-compose.yml
rm /tmp/docker-compose.yml
```

Hinweis: `alert_rules.yml` existiert noch nicht (kommt in Task 5) — eine leere Platzhalterdatei anlegen, damit `docker compose up` nicht am fehlenden Mount scheitert:

```bash
pct exec 105 -- bash -c "echo 'groups: []' > /opt/monitoring/prometheus/alert_rules.yml"
```

- [ ] **Step 3: Stack neu starten**

```bash
pct exec 105 -- bash -c "cd /opt/monitoring && docker compose up -d"
```

- [ ] **Step 4: Verifizieren — Target-Status**

```bash
pct exec 105 -- curl -s http://localhost:9090/api/v1/targets | python3 -c "import json,sys; d=json.load(sys.stdin); print([(t['labels']['job'], t['health']) for t in d['data']['activeTargets']])"
```
Erwartet: `[('pve', 'up')]`

---

### Task 5: Alarmregeln + Alertmanager (Webhook an Home Assistant)

**Files:**
- Create: `/opt/monitoring/prometheus/alert_rules.yml` (in LXC 105, überschreibt den Platzhalter aus Task 4)
- Create: `/opt/monitoring/alertmanager/alertmanager.yml` (in LXC 105)
- Modify: `/opt/monitoring/docker-compose.yml` (in LXC 105) — Alertmanager-Service ergänzen

**Interfaces:**
- Konsumiert: Prometheus aus Task 4, `pve_memory_usage_bytes`/`pve_memory_size_bytes`/`pve_swap_usage_bytes`/`pve_swap_size_bytes`/`pve_up`-Metriken aus Task 3.
- Produziert: bei RAM/Swap ≥ 90 % (5 Min.) oder gestopptem Gast eine Alarm-Push über den bestehenden HA-Webhook — Basis für Task 6 (HA-Automation muss das Alertmanager-JSON verstehen).

- [ ] **Step 1: Alarmregeln schreiben**

```bash
cat > /tmp/alert_rules.yml << 'EOF'
groups:
  - name: proxmox-ressourcen
    rules:
      - alert: HohesRAM
        expr: (pve_memory_usage_bytes / pve_memory_size_bytes) * 100 >= 90
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "RAM auf {{ $labels.id }} bei {{ $value | printf \"%.0f\" }}%"
          description: "{{ $labels.id }} liegt seit mindestens 5 Minuten bei {{ $value | printf \"%.0f\" }}% RAM-Auslastung."

      - alert: HohesSwap
        expr: (pve_swap_usage_bytes / pve_swap_size_bytes) * 100 >= 90
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Swap auf {{ $labels.id }} bei {{ $value | printf \"%.0f\" }}%"
          description: "{{ $labels.id }} liegt seit mindestens 5 Minuten bei {{ $value | printf \"%.0f\" }}% Swap-Auslastung."

      - alert: GastGestoppt
        expr: pve_up == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.id }} ist gestoppt"
          description: "{{ $labels.id }} ist seit mindestens 2 Minuten nicht mehr aktiv (erwartet: laeuft)."
EOF
pct push 105 /tmp/alert_rules.yml /opt/monitoring/prometheus/alert_rules.yml
rm /tmp/alert_rules.yml
```

- [ ] **Step 2: `alertmanager.yml` schreiben** (Webhook-URL aus den Global Constraints)

```bash
cat > /tmp/alertmanager.yml << 'EOF'
route:
  receiver: 'ha-webhook'
  group_by: ['alertname', 'id']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

receivers:
  - name: 'ha-webhook'
    webhook_configs:
      - url: 'http://10.1.0.10:8123/api/webhook/8e9441363ecf659a62ea776e1afd2ef6'
        send_resolved: true
EOF
pct push 105 /tmp/alertmanager.yml /opt/monitoring/alertmanager/alertmanager.yml
rm /tmp/alertmanager.yml
```

`repeat_interval: 4h` entspricht bewusst der Wiederholungsfrist aus dem bisherigen `paperless-health-check.sh`. `send_resolved: true` sorgt für die "wieder ok"-Meldung, sobald eine Regel nicht mehr zutrifft.

- [ ] **Step 3: Alertmanager-Service in `docker-compose.yml` ergänzen**

An bestehende Datei anhängen (unter `services:`, vor `volumes:`):

```yaml
  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
    ports:
      - "9093:9093"
    restart: unless-stopped
```

Und unter `volumes:` (Top-Level) `prometheus-data:` bleibt bestehen, kein neuer Volume-Eintrag nötig (Alertmanager bekommt keinen persistenten State in Phase 1).

Komplette Datei über `pct push` neu schreiben wie in Task 4 Step 2, jetzt mit drei Services (`pve-exporter`, `prometheus`, `alertmanager`).

- [ ] **Step 4: Stack neu starten**

```bash
pct exec 105 -- bash -c "cd /opt/monitoring && docker compose up -d"
```

- [ ] **Step 5: Verifizieren — Regeln geladen**

```bash
pct exec 105 -- curl -s http://localhost:9090/api/v1/rules | python3 -c "import json,sys; d=json.load(sys.stdin); print([r['name'] for g in d['data']['groups'] for r in g['rules']])"
```
Erwartet: `['HohesRAM', 'HohesSwap', 'GastGestoppt']`

- [ ] **Step 6: Verifizieren — Alarm feuert wirklich bis zur Push-Nachricht**

Temporäre Testregel mit `for: 0s` und garantiert wahrem Ausdruck, um die komplette Kette (Prometheus → Alertmanager → HA-Webhook → Mobile-Push) ohne 5 Minuten Wartezeit zu prüfen:

```bash
pct exec 105 -- bash -c "cat > /tmp/test_rule.yml << 'EOF'
groups:
  - name: test
    rules:
      - alert: MonitoringTest
        expr: vector(1) == 1
        for: 0s
        labels:
          severity: warning
        annotations:
          summary: \"Testalarm: Monitoring-Kette funktioniert\"
EOF
cp /opt/monitoring/prometheus/alert_rules.yml /tmp/alert_rules_backup.yml
cat /tmp/test_rule.yml >> /opt/monitoring/prometheus/alert_rules.yml
curl -s -X POST http://localhost:9090/-/reload"
```
Push auf dem iPhone abwarten (bis zu `group_wait: 30s` + Scrape-Intervall), dann Testregel wieder entfernen:
```bash
pct exec 105 -- bash -c "cp /tmp/alert_rules_backup.yml /opt/monitoring/prometheus/alert_rules.yml && curl -s -X POST http://localhost:9090/-/reload"
```
Erwartet: Push-Nachricht kommt an (Inhalt zu diesem Zeitpunkt evtl. noch das rohe Alertmanager-JSON bzw. der Fallback-Text, da Task 6 die Automation erst noch anpasst — hier geht es nur um den Transport-Weg).

---

### Task 6: HA-Automation um Alertmanager-Format erweitern

**Files:**
- Modify: HA-Automation `automation.proxmox_host_alert_webhook` (kein lokales File — über `mcp__homeassistant__ha_config_set_automation`)

**Interfaces:**
- Konsumiert: Alertmanager-Webhook-Payload aus Task 5 (`{status, alerts: [{status, labels, annotations, ...}]}`).
- Produziert: Mobile-Push pro Alert, mit Titel je nach `firing`/`resolved`, weiterhin abwärtskompatibel zum einfachen `{title, message}`-Aufruf.

- [ ] **Step 1: Best-Practice-Key holen**

```
mcp__homeassistant__ha_get_skill_guide(skill="home-assistant-best-practices", file="references/automation-patterns.md")
```
Den `Acknowledgment key` aus der Antwort für Step 2 notieren (rotiert stündlich, nicht aus einer alten Session wiederverwenden).

- [ ] **Step 2: Automation aktualisieren**

```python
mcp__homeassistant__ha_config_set_automation(
    identifier="automation.proxmox_host_alert_webhook",
    config={
        "alias": "Proxmox Host-Alert (Webhook)",
        "description": "Genereller Webhook fuer Host-/Infrastruktur-Alerts. Versteht sowohl Alertmanager-Payloads ({status, alerts:[...]}) als auch das einfache {title, message}-Format.",
        "triggers": [
            {
                "trigger": "webhook",
                "webhook_id": "8e9441363ecf659a62ea776e1afd2ef6",
                "allowed_methods": ["POST"],
                "local_only": True
            }
        ],
        "actions": [
            {
                "choose": [
                    {
                        "conditions": ["{{ 'alerts' in trigger.json }}"],
                        "sequence": [
                            {
                                "repeat": {
                                    "for_each": "{{ trigger.json.alerts }}",
                                    "sequence": [
                                        {
                                            "action": "notify.cm_iphone17p",
                                            "data": {
                                                "title": "{{ '✅ ' + repeat.item.labels.alertname + ' wieder ok' if repeat.item.status == 'resolved' else '⚠️ ' + repeat.item.labels.alertname }}",
                                                "message": "{{ repeat.item.annotations.summary | default(repeat.item.labels.id | default('(kein Ziel)')) }}",
                                                "data": {
                                                    "push": {
                                                        "interruption-level": "{{ 'time-sensitive' if repeat.item.status == 'firing' else 'active' }}"
                                                    }
                                                }
                                            }
                                        }
                                    ]
                                }
                            }
                        ]
                    }
                ],
                "default": [
                    {
                        "action": "notify.cm_iphone17p",
                        "data": {
                            "title": "{{ trigger.json.title | default('Host-Alert') }}",
                            "message": "{{ trigger.json.message | default('(keine Nachricht)') }}",
                            "data": {
                                "push": {
                                    "interruption-level": "time-sensitive"
                                }
                            }
                        }
                    }
                ]
            }
        ],
        "mode": "queued",
        "max": 10
    },
    BestPracticeKey="<KEY_AUS_STEP_1>",
    MandatoryBPS=False
)
```

- [ ] **Step 3: Verifizieren — echten Alertmanager-Test erneut auslösen**

Task-5-Step-6 wiederholen (Testregel kurz aktivieren). Erwartet: Push-Titel `⚠️ MonitoringTest`, Nachricht `Testalarm: Monitoring-Kette funktioniert`. Nach Entfernen der Testregel und Ablauf von `repeat_interval`/Resolve: zweite Push mit `✅ MonitoringTest wieder ok`.

- [ ] **Step 4: Verifizieren — alter Fallback-Pfad funktioniert weiterhin**

```bash
curl -s -X POST -H 'Content-Type: application/json' \
  -d '{"title":"Fallback-Test","message":"Einfaches Format funktioniert noch"}' \
  http://10.1.0.10:8123/api/webhook/8e9441363ecf659a62ea776e1afd2ef6
```
Erwartet: Push mit Titel "Fallback-Test" kommt an (bestätigt, dass `default`-Zweig weiterhin funktioniert).

---

### Task 7: Grafana + Prometheus-Datasource

**Files:**
- Create: `/opt/monitoring/grafana/provisioning/datasources/datasource.yml` (in LXC 105)
- Modify: `/opt/monitoring/docker-compose.yml` (in LXC 105) — Grafana-Service ergänzen

**Interfaces:**
- Konsumiert: Prometheus-Service aus Task 4 (Compose-Service-Name `prometheus`, intern Port 9090).
- Produziert: Grafana unter `http://<LXC105_IP>:3000` erreichbar, Datasource `prometheus` (UID `prometheus`, wird in Task 8 referenziert), Admin-Login mit gesetztem Passwort (nicht `admin`/`admin`).

- [ ] **Step 1: Admin-Passwort festlegen**

```bash
GRAFANA_ADMIN_PASSWORD=$(python3 -c "import secrets; print(secrets.token_urlsafe(18))")
echo "Grafana-Admin-Passwort: $GRAFANA_ADMIN_PASSWORD"
```
**Ausgabe notieren** — wird in Step 3 (Compose-Env) und in Task 10 (Service-Account-Erstellung per API) gebraucht. (Dieses Passwort gehört danach ins Vaultwarden — siehe Hinweis am Ende dieses Plans.)

- [ ] **Step 2: Datasource-Provisioning schreiben**

```bash
cat > /tmp/datasource.yml << 'EOF'
apiVersion: 1
datasources:
  - name: Prometheus
    uid: prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true
EOF
pct push 105 /tmp/datasource.yml /opt/monitoring/grafana/provisioning/datasources/datasource.yml
rm /tmp/datasource.yml
```

- [ ] **Step 3: Grafana-Service in `docker-compose.yml` ergänzen**

```yaml
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=<GRAFANA_ADMIN_PASSWORD_AUS_STEP_1>
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
      - grafana-data:/var/lib/grafana
    ports:
      - "3000:3000"
    restart: unless-stopped
    depends_on:
      - prometheus
```

Unter `volumes:` (Top-Level) ergänzen: `grafana-data:` neben `prometheus-data:`. Komplette Datei per `pct push` neu schreiben (vier Services jetzt: `pve-exporter`, `prometheus`, `alertmanager`, `grafana`).

- [ ] **Step 4: Stack neu starten**

```bash
pct exec 105 -- bash -c "cd /opt/monitoring && docker compose up -d"
```

- [ ] **Step 5: Verifizieren — Login + Datasource**

```bash
pct exec 105 -- curl -s -o /dev/null -w '%{http_code}\n' http://localhost:3000/login
pct exec 105 -- curl -s -u "admin:<GRAFANA_ADMIN_PASSWORD>" http://localhost:3000/api/datasources | python3 -c "import json,sys; print([d['name'] for d in json.load(sys.stdin)])"
```
Erwartet: `200`, dann `['Prometheus']`

---

### Task 8: Grafana-Dashboard (Status-Reihe + Verlaufs-Raster)

**Files:**
- Create: `/opt/monitoring/grafana/provisioning/dashboards/provider.yml` (in LXC 105)
- Create: `/opt/monitoring/grafana/provisioning/dashboards/proxmox-overview.json` (in LXC 105)

**Interfaces:**
- Konsumiert: Datasource `prometheus` (UID `prometheus`) aus Task 7.
- Produziert: Dashboard "Proxmox-Infrastruktur — Übersicht" unter `http://<LXC105_IP>:3000`, Layout wie im Brainstorming freigegeben (Status-Kacheln oben, Verlaufs-Panels unten, ein Satz Panels pro Ziel über eine Template-Variable wiederholt statt sechsmal von Hand).

- [ ] **Step 1: Dashboard-Provider schreiben**

```bash
cat > /tmp/provider.yml << 'EOF'
apiVersion: 1
providers:
  - name: 'proxmox'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUiUpdates: true
    options:
      path: /etc/grafana/provisioning/dashboards
EOF
pct push 105 /tmp/provider.yml /opt/monitoring/grafana/provisioning/dashboards/provider.yml
rm /tmp/provider.yml
```

- [ ] **Step 2: Dashboard-JSON schreiben**

```bash
cat > /tmp/proxmox-overview.json << 'EOF'
{
  "title": "Proxmox-Infrastruktur - Uebersicht",
  "uid": "proxmox-overview",
  "schemaVersion": 39,
  "version": 1,
  "editable": true,
  "timezone": "browser",
  "time": { "from": "now-24h", "to": "now" },
  "refresh": "1m",
  "templating": {
    "list": [
      {
        "name": "target",
        "type": "query",
        "datasource": { "type": "prometheus", "uid": "prometheus" },
        "query": "label_values(pve_up, id)",
        "refresh": 2,
        "sort": 1,
        "multi": false,
        "includeAll": false
      }
    ]
  },
  "panels": [
    {
      "id": 1,
      "type": "stat",
      "title": "$target",
      "gridPos": { "h": 4, "w": 4, "x": 0, "y": 0 },
      "repeat": "target",
      "repeatDirection": "h",
      "datasource": { "type": "prometheus", "uid": "prometheus" },
      "targets": [
        {
          "expr": "(pve_memory_usage_bytes{id=\"$target\"} / pve_memory_size_bytes{id=\"$target\"}) * 100",
          "legendFormat": "RAM %",
          "refId": "A"
        },
        {
          "expr": "(pve_swap_usage_bytes{id=\"$target\"} / pve_swap_size_bytes{id=\"$target\"}) * 100",
          "legendFormat": "Swap %",
          "refId": "B"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "percent",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "red", "value": 90 }
            ]
          }
        },
        "overrides": []
      },
      "options": {
        "textMode": "value_and_name",
        "colorMode": "background",
        "graphMode": "none"
      }
    },
    {
      "id": 2,
      "type": "timeseries",
      "title": "RAM/Swap % - $target",
      "gridPos": { "h": 8, "w": 6, "x": 0, "y": 4 },
      "repeat": "target",
      "repeatDirection": "h",
      "datasource": { "type": "prometheus", "uid": "prometheus" },
      "targets": [
        {
          "expr": "(pve_memory_usage_bytes{id=\"$target\"} / pve_memory_size_bytes{id=\"$target\"}) * 100",
          "legendFormat": "RAM %",
          "refId": "A"
        },
        {
          "expr": "(pve_swap_usage_bytes{id=\"$target\"} / pve_swap_size_bytes{id=\"$target\"}) * 100",
          "legendFormat": "Swap %",
          "refId": "B"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "percent",
          "max": 100,
          "min": 0,
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "red", "value": 90 }
            ]
          }
        },
        "overrides": []
      }
    }
  ]
}
EOF
pct push 105 /tmp/proxmox-overview.json /opt/monitoring/grafana/provisioning/dashboards/proxmox-overview.json
rm /tmp/proxmox-overview.json
```

- [ ] **Step 3: Grafana neu starten, damit Provisioning greift**

```bash
pct exec 105 -- bash -c "cd /opt/monitoring && docker compose restart grafana"
```

- [ ] **Step 4: Verifizieren**

```bash
pct exec 105 -- curl -s -u "admin:<GRAFANA_ADMIN_PASSWORD>" http://localhost:3000/api/dashboards/uid/proxmox-overview | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['dashboard']['title'], len(d['dashboard']['panels']), 'Panel-Definitionen (vor Repeat-Expansion)')"
```
Erwartet: `Proxmox-Infrastruktur - Uebersicht 2 Panel-Definitionen (vor Repeat-Expansion)`. Danach im Browser `http://<LXC105_IP>:3000/d/proxmox-overview` gegen die im Brainstorming freigegebenen Mockups sichtprüfen (6 Status-Kacheln oben, 6 Verlaufs-Panels darunter, `102 · paperless` o.ä. mit rotem Rand falls über Schwelle).

---

### Task 9: LXC 106 (`monitoring-mcp`) anlegen + Docker installieren

**Files:**
- Keine lokalen Dateien.

**Interfaces:**
- Produziert: laufender Debian-12-LXC mit Docker CE, IP-Adresse (`<LXC106_IP>`).

- [ ] **Step 1: LXC 106 erstellen**

```bash
pct create 106 local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
  --hostname monitoring-mcp \
  --unprivileged 1 \
  --features nesting=1,keyctl=1 \
  --cores 1 \
  --memory 1024 \
  --swap 512 \
  --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --onboot 1
pct start 106
```

- [ ] **Step 2: Verifizieren**

```bash
pct status 106
```
Erwartet: `status: running`

- [ ] **Step 3: Docker CE installieren** (identisches Kommando wie Task 1 Step 3, mit `pct exec 106`)

```bash
pct exec 106 -- bash -c "
apt-get update && apt-get install -y ca-certificates curl && \
install -m 0755 -d /etc/apt/keyrings && \
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc && \
chmod a+r /etc/apt/keyrings/docker.asc && \
echo 'deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian bookworm stable' > /etc/apt/sources.list.d/docker.list && \
apt-get update && \
apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
"
```

- [ ] **Step 4: IP-Adresse ermitteln**

```bash
pct exec 106 -- hostname -I
```
Notieren als `<LXC106_IP>`.

- [ ] **Step 5: Compose-Verzeichnis anlegen**

```bash
pct exec 106 -- mkdir -p /opt/monitoring-mcp
```

---

### Task 10: Grafana-Service-Account (Rolle `Editor`) + `mcp-grafana` deployen

**Files:**
- Create: `/opt/monitoring-mcp/docker-compose.yml` (in LXC 106)

**Interfaces:**
- Konsumiert: Grafana aus Task 7 (`http://<LXC105_IP>:3000`), Admin-Login aus Task 7 Step 1.
- Produziert: `mcp-grafana` erreichbar unter `http://<LXC106_IP>:8000/mcp` (Standardport des offiziellen Images — bei Abweichung siehe Verifikationsschritt), Service-Account-Token mit Rolle `Editor`.

- [ ] **Step 1: Service-Account + Token per Grafana-API anlegen**

```bash
pct exec 105 -- bash -c '
curl -s -u "admin:<GRAFANA_ADMIN_PASSWORD>" -X POST http://localhost:3000/api/serviceaccounts \
  -H "Content-Type: application/json" \
  -d "{\"name\":\"monitoring-mcp\",\"role\":\"Editor\"}"
'
```
Erwartet: JSON mit `"id"` — diese ID für Step 2 notieren als `<SA_ID>`.

```bash
pct exec 105 -- bash -c '
curl -s -u "admin:<GRAFANA_ADMIN_PASSWORD>" -X POST http://localhost:3000/api/serviceaccounts/<SA_ID>/tokens \
  -H "Content-Type: application/json" \
  -d "{\"name\":\"monitoring-mcp-token\"}"
'
```
Erwartet: JSON mit `"key"` = das eigentliche Token (nur einmal sichtbar). **Notieren** als `<GRAFANA_SA_TOKEN>`.

- [ ] **Step 2: `docker-compose.yml` für `mcp-grafana` schreiben**

```bash
cat > /tmp/docker-compose.yml << 'EOF'
services:
  monitoring-mcp:
    image: mcp/grafana:latest
    container_name: monitoring-mcp
    command: ["-t", "streamable-http"]
    environment:
      - GRAFANA_URL=http://<LXC105_IP>:3000
      - GRAFANA_SERVICE_ACCOUNT_TOKEN=<GRAFANA_SA_TOKEN>
    ports:
      - "8000:8000"
    restart: unless-stopped
EOF
pct push 106 /tmp/docker-compose.yml /opt/monitoring-mcp/docker-compose.yml
rm /tmp/docker-compose.yml
```

- [ ] **Step 3: Stack starten**

```bash
pct exec 106 -- bash -c "cd /opt/monitoring-mcp && docker compose up -d"
```

- [ ] **Step 4: Verifizieren — Container läuft, Port antwortet**

```bash
pct exec 106 -- docker compose -f /opt/monitoring-mcp/docker-compose.yml ps
pct exec 106 -- curl -s -o /dev/null -w '%{http_code}\n' --max-time 5 http://localhost:8000/mcp
```
Erwartet: Service-Status `running`/`Up`. Bei abweichendem HTTP-Code oder Verbindungsfehler: `docker logs monitoring-mcp` prüfen — Image-Name (`mcp/grafana`) und Transport-Flag (`-t streamable-http`) stammen aus der aktuellen `mcp-grafana`-Doku von Grafana Labs zum Zeitpunkt der Planerstellung; bei Abweichung dort die aktuelle Syntax nachschlagen und diesen Schritt entsprechend anpassen — dieser Verifikationsschritt ist bewusst die Absicherung dafür.

---

### Task 11: `monitoring-mcp` in `claude_desktop_config.json` registrieren

**Files:**
- Modify: `claude_desktop_config.json` (MSIX-Pfad, siehe Hinweis unten)

**Interfaces:**
- Konsumiert: `mcp-grafana`-Endpoint aus Task 10 (`http://<LXC106_IP>:8000/mcp`).
- Produziert: Claude kann `monitoring-mcp`-Tools nutzen (nach Neustart der Desktop-App).

- [ ] **Step 1: Richtigen Pfad der Config-Datei ermitteln**

Claude Desktop ist als MSIX-Paket installiert — die tatsächlich gelesene Datei liegt **nicht** unter `AppData\Roaming\Claude\`, sondern unter `AppData\Local\Packages\Claude_<paketkennung>\LocalCache\Roaming\Claude\claude_desktop_config.json` (siehe `paperless-senior`-Skill, Abschnitt "Windows-Konfigurationsdatei"). Paketkennung ermitteln:

```powershell
Get-ChildItem "$env:LOCALAPPDATA\Packages" -Filter "Claude_*" | Select-Object -ExpandProperty Name
```

- [ ] **Step 2: Bestehenden Eintrag als Vorlage lesen**

```powershell
$path = "$env:LOCALAPPDATA\Packages\<PAKETKENNUNG>\LocalCache\Roaming\Claude\claude_desktop_config.json"
Get-Content $path -Raw
```

- [ ] **Step 3: Eintrag für `monitoring-mcp` ergänzen** (gleiches `mcp-remote`-Bridge-Muster wie `paperless`/`unifi-network`)

Neuen Eintrag unter `mcpServers` einfügen, analog zu:
```json
"monitoring": {
  "command": "npx",
  "args": ["-y", "mcp-remote", "http://<LXC106_IP>:8000/mcp", "--allow-http"]
}
```
Datei mit `[System.IO.File]::WriteAllBytes` schreiben (kein `-Encoding utf8` mit BOM, siehe Troubleshooting-Skill "PowerShell 5.1 mit -Encoding utf8 schreibt ein BOM").

- [ ] **Step 4: Alte `npx`-Prozesse beenden, App neu starten**

```powershell
Get-Process npx -ErrorAction SilentlyContinue | Stop-Process -Force
```
Claude Desktop komplett schließen und neu starten (nicht nur das Fenster — die App muss beendet sein, damit sie die geänderte Config neu liest).

- [ ] **Step 5: Verifizieren**

In einer neuen Claude-Desktop-Konversation ein `monitoring-mcp`-Tool aufrufen (z.B. Dashboards auflisten) und prüfen, dass das Dashboard aus Task 8 zurückkommt.

---

### Task 12: Altes Health-Check-Skript ablösen + Dokumentation nachziehen

**Files:**
- Modify: `paperless-health-check.timer` (Proxmox-Host, kein lokales File)
- Modify: `C:\Users\cmuellar\.claude\skills\paperless-senior\references\setup-knowledge.md` — Abschnitt "Monitoring" aktualisieren
- Modify: `C:\Users\cmuellar\.claude\projects\C--Users-cmuellar-mcp-servers-proxmox-mcp\memory\paperless_monitoring.md` — auf neuen Stack verweisen

**Interfaces:**
- Konsumiert: verifizierte Alarmkette aus Task 5/6 (mindestens ein sauberer Testlauf ohne Fehlalarm).

- [ ] **Step 1: Sicherstellen, dass die neue Kette mindestens einen reellen 10-Minuten-Zyklus unauffällig lief**

```bash
pct exec 105 -- curl -s http://localhost:9090/api/v1/rules | python3 -c "import json,sys; d=json.load(sys.stdin); print([(r['name'], r['health']) for g in d['data']['groups'] for r in g['rules']])"
```
Erwartet: alle drei Regeln `health: 'ok'`.

- [ ] **Step 2: Altes Skript deaktivieren (nicht löschen)**

```bash
systemctl disable --now paperless-health-check.timer
systemctl status paperless-health-check.timer --no-pager
```
Erwartet: `Loaded: loaded ... ; disabled`, `Active: inactive (dead)`.

- [ ] **Step 3: `setup-knowledge.md` aktualisieren**

Abschnitt "Monitoring: Health-Check gegen Speicher-/Swap-Erschöpfung (seit 2026-08-29)" um einen Absatz ergänzen: Skript ist seit `<HEUTIGES DATUM>` deaktiviert, abgelöst durch den Prometheus/Grafana/Alertmanager-Stack in LXC 105/106 (Verweis auf `docs/superpowers/specs/2026-08-29-prometheus-grafana-monitoring-design.md` in diesem Repo). Alarmschwellen (RAM/Swap ≥ 90 %) sind identisch geblieben, jetzt aber als Prometheus-Regeln statt Bash-Skript.

- [ ] **Step 4: Memory `paperless_monitoring.md` aktualisieren**

`How to apply`-Abschnitt ergänzen: Bei künftigen Fragen zum Monitoring zuerst LXC 105 (Prometheus/Grafana/Alertmanager) und LXC 106 (`monitoring-mcp`) prüfen, nicht mehr das alte Skript — das ist nur noch Fallback-Referenz, deaktiviert.

- [ ] **Step 5: Commit der Skill-/Doku-Änderungen**

Skill- und Memory-Dateien liegen außerhalb des `proxmox-mcp`-Repos (`~/.claude/skills/`, `~/.claude/projects/.../memory/`) — kein Git-Commit in diesem Repo nötig für Step 3/4. Für den Plan selbst: keine weiteren Repo-Änderungen in diesem Task.

---

## Hinweis zu Secrets nach Abschluss

Nach Task 2 (Proxmox-API-Token), Task 7 (Grafana-Admin-Passwort) und Task 10
(Grafana-Service-Account-Token) bewusst als Klartext in Compose-Dateien/
Kommandos verwendet (siehe Global Constraints). Diese drei Werte sollten
trotzdem — wie schon bei den vorherigen MCP-Migrationen offen gelassen —
irgendwann in Vaultwarden nachgetragen werden, spätestens wenn cmuellar die
generelle Secret-Strategie für alle MCP-LXCs überarbeitet. Kein Blocker für
diesen Plan.
