# MCP-Server-Migration nach Proxmox: paperless-mcp & unifi-mcp

Status: Design genehmigt (2026-08-20). Umfasst NICHT proxmox-mcp (siehe Abschnitt "Out of scope").

## Ausgangslage

Aktuell laufen vier MCP-Server für Claude, konfiguriert in
`C:\Users\cmuellar\AppData\Roaming\Claude\claude_desktop_config.json`:

| Server | Betrieb heute | Ziel |
|---|---|---|
| `homeassistant` | Docker in LXC 101 (`ha-mcp`), erreichbar via `npx mcp-remote http://10.1.0.155:8086/mcp` | bereits migriert, dient als Vorbild |
| `paperless` | lokal, `npx -y paperless-ngx-mcp`, spricht `http://10.1.0.30:8000` an | **migrieren** |
| `unifi-network` | lokal, `uvx unifi-network-mcp@latest` | **migrieren** |
| `proxmox` | lokal, `python -m proxmox_mcp.server` (dieses Repo) | separater Task, siehe unten |

cmuellar möchte künftig keine MCP-Server lokal auf dem Windows-PC laufen lassen, sondern
konsequent auf dem Proxmox-Host, analog zu `ha-mcp`.

## Out of scope: proxmox-mcp

`proxmox-mcp` (dieses Repo) unterstützt aktuell ausschließlich stdio-Transport
(`mcp.server.stdio.stdio_server` in `src/proxmox_mcp/server.py`). Um es remote per
`mcp-remote` erreichbar zu machen, braucht es zuerst einen HTTP/Streamable-Transport-Modus
im offiziellen `mcp`-SDK (kein Wechsel zu FastMCP, siehe CLAUDE.md). Das ist eine echte
Code-Änderung mit eigenen Tests, kein reiner Infra-Task, und wird bewusst als **eigene,
spätere Spec** behandelt. Zusätzlich hat proxmox-mcp einen besonderen Blast-Radius (volles
VM/LXC-Root + SSH-Zugriff auf den Host, auf dem es liefe) — auch das verdient eine eigene,
fokussierte Betrachtung statt im selben Rutsch mitgemacht zu werden.

Dieses Dokument deckt ausschließlich `paperless-mcp` und `unifi-mcp` ab.

## Zielarchitektur

Zwei neue **unprivilegierte Debian-12-LXCs** auf dem Proxmox-Host `proxmox`, Namenskonvention
durchgängig `<funktion>-mcp` (analog `ha-mcp`):

- **`paperless-mcp`**: Docker-Container mit dem npm-Paket `paperless-ngx-mcp` im `--http`-Modus,
  spricht `PAPERLESS_URL=http://10.1.0.30:8000` (Paperless-ngx in LXC 102) an. Port 8087
  (ursprünglich für genau diesen Zweck reserviert, siehe `paperless_projekt.md`-Memory).
- **`unifi-mcp`**: Docker-Container mit dem PyPI-Paket `unifi-network-mcp`, HTTP nativ per
  Default aktiviert (`UNIFI_MCP_HTTP_ENABLED=true`, Default-Port 3000). Spricht den
  UniFi-Controller (`unifi.muellar.org`) mit dem bestehenden lokalen Admin-Konto `mcp-bot` an.

Jede LXC bekommt Docker CE per `get.docker.com`-Convenience-Script (gleiches Muster wie LXC 101,
dort aktuell Docker 29.6.2), danach ein eigenes `/opt/<funktion>-mcp/docker-compose.yml`.

Client-seitig wird in `claude_desktop_config.json` der jeweilige Eintrag von lokalem
`command`/`args` (npx/uvx) auf das `mcp-remote`-Bridge-Muster umgestellt, das beim
`homeassistant`-Eintrag schon funktioniert:

```json
"paperless": {
  "command": "npx",
  "args": ["-y", "mcp-remote", "http://<paperless-mcp-ip>:8087/mcp", "--allow-http"]
},
"unifi-network": {
  "command": "npx",
  "args": ["-y", "mcp-remote", "http://<unifi-mcp-ip>:3000/mcp", "--allow-http"]
}
```

## Ressourcen

Pro LXC: 1 vCPU, 1 GB RAM, 8 GB Disk auf `local-lvm`, unprivilegiert. Macht +2 GB RAM-Bedarf
auf dem Host (aktuell ~6 GB von 16 GB belegt durch HAOS + `ha-mcp`) — bleibt unkritisch, aber
im Auge behalten, falls später `proxmox-mcp` als dritte LXC dazukommt.

## Secrets

Bewusste Entscheidung (2026-08-20): wie bei `ha-mcp` werden Paperless-API-Token und
UniFi-`mcp-bot`-Zugangsdaten direkt als Klartext-Env-Var in der jeweiligen `docker-compose.yml`
hinterlegt — kein Vaultwarden-CLI-Pull, keine externe Secret-Datei. Das wiederholt bewusst das
bestehende `ha-mcp`-Muster statt es zu beheben; falls das später revidiert wird, gilt es dann
konsequenterweise für alle drei LXCs gleichermaßen.

## DNS

Neue statische UniFi-DNS-Records `paperless-mcp.muellar.org` und `unifi-mcp.muellar.org` (feste
IPs der neuen LXCs). Vermeidet die bereits einmal erlebte Falle mit veralteten/toten
Client-Namen als DNS-Basis (siehe `unifi_mcp_integration.md`-Memory, Vorfall
`paperless.muellar.org`).

## Rollout-Reihenfolge

1. **`paperless-mcp` zuerst, komplett end-to-end:**
   - LXC anlegen (Debian-12-Template, unprivilegiert, 1 vCPU/1 GB/8 GB)
   - Docker installieren
   - `/opt/paperless-mcp/docker-compose.yml` schreiben (Image/Paket, Env-Vars, Port 8087)
   - `docker compose up -d`, Container-Log auf sauberen Start prüfen
   - Erreichbarkeit vom Proxmox-Host testen (curl auf den HTTP-Endpunkt)
   - UniFi-DNS-Record `paperless-mcp.muellar.org` anlegen
   - `claude_desktop_config.json` für `paperless` auf `mcp-remote` umstellen
   - Claude neu verbinden, Smoke-Test mit einem echten Paperless-Tool-Call (z.B. Dokument
     auflisten), Ergebnis mit dem alten lokalen Verhalten vergleichen
   - Erst wenn das sauber durchläuft: alter lokaler Eintrag gilt als abgelöst (kein Cleanup
     nötig, `npx`/`uvx` sind ohnehin nur Ad-hoc-Prozesse)
2. **Danach `unifi-mcp` nach demselben Muster**, inkl. Prüfung der tatsächlichen Env-Var-Namen
   (`UNIFI_HOST`/`UNIFI_USERNAME`/`UNIFI_PASSWORD` laut Package-Default-Config vs. lokal
   bislang genutzte `UNIFI_NETWORK_*`-Namen — vor dem Deploy verifizieren, welche das laufende
   Package tatsächlich liest, nicht raten).

## Testing/Verifikation

Nach jeder Migration: mindestens ein echter Tool-Call über die neue Remote-Verbindung
(nicht nur "Container läuft"), Container-Log auf Fehler/Exceptions prüfen,
`restart: unless-stopped` in der Compose-Datei sicherstellen (Persistenz nach LXC-Reboot).

## Später (nicht Teil dieser Spec)

- `proxmox-mcp`: HTTP-Transport-Feature + Migration in eigene LXC, eigene Spec.
- Vaultwarden-Integration für alle drei MCP-LXCs nachziehen, falls das Secrets-Handling
  irgendwann konsolidiert werden soll.
