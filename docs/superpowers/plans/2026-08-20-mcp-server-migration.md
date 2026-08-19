# paperless-mcp & unifi-mcp Proxmox Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move the `paperless` and `unifi-network` MCP servers off the local Windows PC onto two new dedicated Proxmox LXCs (`paperless-mcp`, `unifi-mcp`), reachable over HTTP via `mcp-remote`, mirroring the existing `ha-mcp` setup.

**Architecture:** Two new unprivileged Debian 12 LXCs on Proxmox host `proxmox`, each running Docker with a single MCP-server container exposing a Streamable HTTP endpoint at `POST /mcp`. `claude_desktop_config.json` on the Windows PC switches each server's `command`/`args` from a local process spawn to `npx mcp-remote http://<lxc-ip>:3000/mcp --allow-http`.

**Tech Stack:** Proxmox `pct`, Docker + Docker Compose, `paperless-ngx-mcp` (npm), `unifi-network-mcp` (PyPI, via `uv`), `mcp-remote` (npm bridge), UniFi Network controller (DHCP reservation + DNS record).

## Global Constraints

- Naming convention: every new resource is named `<funktion>-mcp` (LXC hostname, DNS record, Docker container name) — e.g. `paperless-mcp`, `unifi-mcp`.
- Secrets go directly as plaintext env vars in each `docker-compose.yml` (same pattern as the existing `ha-mcp` LXC 101) — no Vaultwarden CLI pull, no external secret file. Never paste the actual secret VALUES into this repo or into any committed file — only reference where to read them from.
- Each LXC: 1 vCPU, 1024 MB RAM, 512 MB swap, 8 GB disk on `local-lvm`, unprivileged, `features nesting=1,keyctl=1` (required for Docker-in-LXC — confirmed present on the working `ha-mcp` LXC 101).
- Template: `local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst` (confirmed present on `proxmox` node).
- Network: `bridge=vmbr0`, `ip=dhcp` (matches `ha-mcp`'s LXC 101 config) + a UniFi DHCP fixed-IP reservation so the address stays stable for the DNS record.
- proxmox-mcp itself is explicitly OUT OF SCOPE for this plan (see spec, `docs/superpowers/specs/2026-08-20-mcp-server-migration-design.md`, section "Out of scope").
- Next free Proxmox VMIDs (confirmed via `pvesh get /cluster/nextid` on 2026-08-20): use **103** for `paperless-mcp`, **104** for `unifi-mcp`.

---

### Task 1: Create the `paperless-mcp` LXC

**Files:** none (Proxmox host state only)

**Interfaces:**
- Produces: a running, empty LXC named `paperless-mcp` at VMID 103, reachable via `mcp__proxmox__pct_exec(vmid=103, node="proxmox", command=...)` for all later steps in this task set.

- [ ] **Step 1: Create the LXC**

Run via `mcp__proxmox__ssh_execute(node="proxmox", command=...)`:

```bash
pct create 103 local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
  --hostname paperless-mcp \
  --cores 1 \
  --memory 1024 \
  --swap 512 \
  --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp,type=veth \
  --unprivileged 1 \
  --features nesting=1,keyctl=1 \
  --ostype debian \
  --start 1
```

Expected: exit code 0, output ends with `TASK OK`.

- [ ] **Step 2: Verify the LXC is running and get its IP**

Run via `mcp__proxmox__pct_exec(vmid=103, node="proxmox", command="hostname -I")`.

Expected: a single IPv4 address in the `10.1.0.0/24` range (e.g. `10.1.0.xxx`). Write this address down — it's needed in Task 3 and Task 4. If the command fails with "container not running" or times out, wait 10s (DHCP lease takes a moment after first boot) and retry once.

- [ ] **Step 3: Confirm base connectivity**

Run via `mcp__proxmox__pct_exec(vmid=103, node="proxmox", command="apt-get update -qq && echo APT_OK")`.

Expected: `APT_OK` printed, no error output. This confirms the container has working DNS/internet access before installing Docker in Task 2.

---

### Task 2: Install Docker in `paperless-mcp`

**Files:** none

**Interfaces:**
- Consumes: LXC 103 running (Task 1).
- Produces: working `docker` and `docker compose` CLI inside LXC 103.

- [ ] **Step 1: Install Docker via the official convenience script**

Run via `mcp__proxmox__pct_exec(vmid=103, node="proxmox", command="curl -fsSL https://get.docker.com | sh", timeout=120)`.

Expected: script completes, last lines mention `Docker Engine ... is now installed` (or similar), exit code 0.

- [ ] **Step 2: Verify Docker works**

Run via `mcp__proxmox__pct_exec(vmid=103, node="proxmox", command="docker version --format '{{.Server.Version}}' && docker compose version")`.

Expected: a version string (e.g. `29.x.x`) followed by a `Docker Compose version vX.X.X` line, no errors. If this fails with a cgroup/permission error, re-check that `--features nesting=1,keyctl=1` was actually applied (`pct config 103 | grep features` should show it) — Docker-in-LXC does not work without it.

---

### Task 3: Deploy the `paperless-ngx-mcp` container

**Files:**
- Create (inside LXC 103, not in this git repo): `/opt/paperless-mcp/docker-compose.yml`

**Interfaces:**
- Consumes: Docker running in LXC 103 (Task 2); the current Paperless API token, read from the **local** `claude_desktop_config.json` → `mcpServers.paperless.env.PAPERLESS_API_KEY` (do not copy this value into any file that gets committed to git).
- Produces: an HTTP MCP endpoint at `http://<lxc-103-ip>:3000/mcp`, used by Task 5.

- [ ] **Step 1: Read the current Paperless API token locally**

Run: `grep -A 3 '"paperless"' "/c/Users/cmuellar/AppData/Roaming/Claude/claude_desktop_config.json"` and note the `PAPERLESS_API_KEY` value. Keep it only in your working memory for the next step — do not write it to any file under `C:\Users\cmuellar\mcp-servers\proxmox-mcp`.

- [ ] **Step 2: Write the compose file**

Run via `mcp__proxmox__pct_exec(vmid=103, node="proxmox", command="mkdir -p /opt/paperless-mcp")`.

Then via `mcp__proxmox__ssh_write_file` (or `pct_exec` with a heredoc) write `/opt/paperless-mcp/docker-compose.yml` inside LXC 103 with this content, substituting the real token from Step 1 for `<PAPERLESS_API_KEY>`:

```yaml
services:
  paperless-mcp:
    image: node:22-alpine
    container_name: paperless-mcp
    working_dir: /app
    command: ["npx", "-y", "paperless-ngx-mcp", "--http"]
    ports:
      - "3000:3000"
    environment:
      - PAPERLESS_URL=http://10.1.0.30:8000
      - PAPERLESS_API_KEY=<PAPERLESS_API_KEY>
    restart: unless-stopped
```

(This mirrors the CLI/env contract of `paperless-ngx-mcp` verified directly in its `build/index.js`: `--http` switches it to Streamable HTTP mode on port 3000 by default, serving `POST /mcp`; `PAPERLESS_URL`/`PAPERLESS_API_KEY` are its documented env var names.)

- [ ] **Step 3: Start the container**

Run via `mcp__proxmox__pct_exec(vmid=103, node="proxmox", command="cd /opt/paperless-mcp && docker compose up -d", timeout=60)`.

Expected: `Container paperless-mcp Started`, exit code 0.

- [ ] **Step 4: Verify clean startup**

Run via `mcp__proxmox__pct_exec(vmid=103, node="proxmox", command="sleep 3 && docker ps --format '{{.Names}}\t{{.Status}}' && docker logs paperless-mcp --tail 20")`.

Expected: `paperless-mcp   Up X seconds` in the `docker ps` output, and the log contains `MCP Streamable HTTP Server listening on port 3000` with no stack traces above it.

- [ ] **Step 5: Verify the endpoint is reachable from the Proxmox host**

Run via `mcp__proxmox__ssh_execute(node="proxmox", command="curl -s -o /dev/null -w 'HTTP %{http_code}\\n' --max-time 5 http://<lxc-103-ip>:3000/mcp")` (substitute the real IP from Task 1 Step 2).

Expected: some HTTP status code is printed (not a connection error/timeout) — a 4xx here is fine and expected for an unauthenticated bare GET against a session-based POST endpoint; a connection refused/timeout means the container isn't actually listening and Steps 3-4 need re-checking.

---

### Task 4: DHCP reservation + DNS record for `paperless-mcp`

**Files:** none (UniFi controller state only)

**Interfaces:**
- Consumes: LXC 103's current DHCP-assigned IP and MAC address.
- Produces: a stable IP for `paperless-mcp` and a resolvable hostname `paperless-mcp.muellar.org`.

- [ ] **Step 1: Get the LXC's MAC address**

Run via `mcp__proxmox__ssh_execute(node="proxmox", command="pct config 103 | grep net0")`.

Expected: a line like `net0: name=eth0,bridge=vmbr0,hwaddr=XX:XX:XX:XX:XX:XX,ip=dhcp,type=veth`. Note the `hwaddr` value.

- [ ] **Step 2: Load the UniFi MCP tools**

Call `ToolSearch(query="select:mcp__unifi-network__unifi_tool_index,mcp__unifi-network__unifi_execute", max_results=5)` if these are not already loaded in the current session.

- [ ] **Step 3: Find the right tools**

Call `mcp__unifi-network__unifi_tool_index` with a query for "fixed ip" / "DHCP reservation" and separately for "DNS record" to get the exact current tool names and parameter shapes (per `unifi_mcp_integration.md` memory, parameter names on this server often diverge from their descriptions — always read the actual tool schema returned here, don't guess).

- [ ] **Step 4: Create the fixed-IP (DHCP reservation) for the MAC from Step 1**

Use the tool found in Step 3 to reserve a fixed IP for `hwaddr` — reuse whatever IP the LXC already got via DHCP in Task 1 Step 2 (simplest: turn the existing lease into a reservation instead of picking a new address).

Expected: the tool call reports success against the "default" site.

- [ ] **Step 5: Create the DNS A record**

Use the DNS-record tool found in Step 3 to create `paperless-mcp.muellar.org` → the fixed IP from Step 4. Per the known trap documented in `unifi_mcp_integration.md`: newly created DNS records default to `enabled: false` — immediately follow up with the corresponding update call to set `enabled: true`.

- [ ] **Step 6: Verify resolution**

Run via `mcp__proxmox__ssh_execute(node="proxmox", command="getent hosts paperless-mcp.muellar.org")`.

Expected: resolves to the fixed IP from Step 4. If it doesn't resolve yet, DNS propagation on the UniFi gateway can take up to a minute — retry once after a short wait.

---

### Task 5: Switch the local `paperless` MCP config to remote

**Files:**
- Modify: `C:\Users\cmuellar\AppData\Roaming\Claude\claude_desktop_config.json`

**Interfaces:**
- Consumes: the working HTTP endpoint from Task 3, the resolvable hostname from Task 4.
- Produces: the `paperless` MCP server now runs remotely; no later task depends on this one.

- [ ] **Step 1: Back up the config file**

Run: `cp "/c/Users/cmuellar/AppData/Roaming/Claude/claude_desktop_config.json" "/c/Users/cmuellar/AppData/Roaming/Claude/claude_desktop_config.json.bak-2026-08-20"`.

- [ ] **Step 2: Edit the `paperless` entry**

Using the Edit tool, change the `paperless` block in `claude_desktop_config.json` from:

```json
"paperless": {
  "command": "npx",
  "args": ["-y", "paperless-ngx-mcp"],
  "env": {
    "PAPERLESS_URL": "http://10.1.0.30:8000",
    "PAPERLESS_API_KEY": "..."
  }
}
```

to:

```json
"paperless": {
  "command": "npx",
  "args": ["-y", "mcp-remote", "http://paperless-mcp.muellar.org:3000/mcp", "--allow-http"]
}
```

(Drop the `env` block entirely — the token now lives only in the remote container's compose file, not on the local machine.)

- [ ] **Step 3: Restart Claude Desktop and verify**

Ask cmuellar to fully quit and reopen Claude Desktop (MCP server config is only read at startup). Once reconnected, call any `mcp__paperless__*` tool (e.g. list the 5 most recent documents) and confirm it returns real data matching what the old local server would have returned.

Expected: tool call succeeds, data looks correct (real document titles/dates from the live Paperless instance, not an error).

- [ ] **Step 4: Update memory**

Update `mcp_server_locations.md` (in the auto-memory store) to reflect that `paperless` now runs via `mcp-remote` against LXC 103 `paperless-mcp`, no longer locally via `npx`.

---

### Task 6: Create the `unifi-mcp` LXC

**Files:** none

**Interfaces:**
- Produces: a running, empty LXC named `unifi-mcp` at VMID 104.

- [ ] **Step 1: Create the LXC**

Run via `mcp__proxmox__ssh_execute(node="proxmox", command=...)`:

```bash
pct create 104 local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
  --hostname unifi-mcp \
  --cores 1 \
  --memory 1024 \
  --swap 512 \
  --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp,type=veth \
  --unprivileged 1 \
  --features nesting=1,keyctl=1 \
  --ostype debian \
  --start 1
```

Expected: exit code 0, output ends with `TASK OK`.

- [ ] **Step 2: Verify the LXC is running and get its IP**

Run via `mcp__proxmox__pct_exec(vmid=104, node="proxmox", command="hostname -I")`.

Expected: a single IPv4 address in `10.1.0.0/24`. Note it for Task 8.

- [ ] **Step 3: Confirm base connectivity**

Run via `mcp__proxmox__pct_exec(vmid=104, node="proxmox", command="apt-get update -qq && echo APT_OK")`.

Expected: `APT_OK`, no errors.

---

### Task 7: Install Docker and deploy `unifi-network-mcp` in `unifi-mcp`

**Files:**
- Create (inside LXC 104, not in this git repo): `/opt/unifi-mcp/Dockerfile`, `/opt/unifi-mcp/docker-compose.yml`

**Interfaces:**
- Consumes: LXC 104 running (Task 6); the current UniFi `mcp-bot` password, read from the **local** `claude_desktop_config.json` → `mcpServers.unifi-network.env.UNIFI_NETWORK_PASSWORD` (do not copy this value into any file that gets committed to git).
- Produces: an HTTP MCP endpoint at `http://<lxc-104-ip>:3000/mcp`, used by Task 9.

- [ ] **Step 1: Install Docker via the official convenience script**

Run via `mcp__proxmox__pct_exec(vmid=104, node="proxmox", command="curl -fsSL https://get.docker.com | sh", timeout=120)`.

Expected: exit code 0, script reports Docker installed.

- [ ] **Step 2: Verify Docker works**

Run via `mcp__proxmox__pct_exec(vmid=104, node="proxmox", command="docker version --format '{{.Server.Version}}' && docker compose version")`.

Expected: version strings printed, no errors.

- [ ] **Step 3: Read the current UniFi password locally**

Run: `grep -A 4 '"unifi-network"' "/c/Users/cmuellar/AppData/Roaming/Claude/claude_desktop_config.json"` and note the `UNIFI_NETWORK_PASSWORD` value. Keep it only in working memory.

- [ ] **Step 4: Write the Dockerfile**

Run via `mcp__proxmox__pct_exec(vmid=104, node="proxmox", command="mkdir -p /opt/unifi-mcp")`.

Then write `/opt/unifi-mcp/Dockerfile` inside LXC 104 with:

```dockerfile
FROM python:3.12-slim
RUN pip install --no-cache-dir uv
ENTRYPOINT ["uvx"]
CMD ["unifi-network-mcp@latest"]
```

(`unifi-network-mcp` ships only as a PyPI package meant to be run via `uvx` — there is no official prebuilt image, so this Dockerfile wraps it. Confirmed locally: its bundled config resolves HTTP mode via `UNIFI_MCP_HTTP_ENABLED` (default `true`) on port 3000 (env `UNIFI_MCP_PORT`, default `3000`), and its credential env vars are `UNIFI_NETWORK_HOST`/`UNIFI_NETWORK_USERNAME`/`UNIFI_NETWORK_PASSWORD` — confirmed via `bootstrap.py`'s `env_prefix="NETWORK"`, which is what makes the already-working local config's `UNIFI_NETWORK_*` names correct.)

- [ ] **Step 5: Write the compose file**

Write `/opt/unifi-mcp/docker-compose.yml` inside LXC 104, substituting the real password from Step 3 for `<UNIFI_NETWORK_PASSWORD>`:

```yaml
services:
  unifi-mcp:
    build: .
    container_name: unifi-mcp
    ports:
      - "3000:3000"
    environment:
      - UNIFI_NETWORK_HOST=unifi.muellar.org
      - UNIFI_NETWORK_USERNAME=mcp-bot
      - UNIFI_NETWORK_PASSWORD=<UNIFI_NETWORK_PASSWORD>
      - UNIFI_MCP_HTTP_ENABLED=true
    restart: unless-stopped
```

- [ ] **Step 6: Build and start the container**

Run via `mcp__proxmox__pct_exec(vmid=104, node="proxmox", command="cd /opt/unifi-mcp && docker compose up -d --build", timeout=180)`.

Expected: image builds successfully (pip install of `uv` completes), `Container unifi-mcp Started`, exit code 0.

- [ ] **Step 7: Verify clean startup**

Run via `mcp__proxmox__pct_exec(vmid=104, node="proxmox", command="sleep 5 && docker ps --format '{{.Names}}\t{{.Status}}' && docker logs unifi-mcp --tail 30")`.

Expected: `unifi-mcp   Up X seconds`, and the log shows the server starting up and reaching a "listening"/"ready" state with no `No controller host configured` error (that error means the env vars from Step 5 weren't picked up — re-check the compose file) and no authentication failure against the UniFi controller.

- [ ] **Step 8: Verify the endpoint is reachable from the Proxmox host**

Run via `mcp__proxmox__ssh_execute(node="proxmox", command="curl -s -o /dev/null -w 'HTTP %{http_code}\\n' --max-time 5 http://<lxc-104-ip>:3000/mcp")` (substitute the real IP from Task 6 Step 2).

Expected: an HTTP status code is printed (connection succeeds); a connection refused/timeout means Steps 6-7 need re-checking.

---

### Task 8: DHCP reservation + DNS record for `unifi-mcp`

**Files:** none (UniFi controller state only)

**Interfaces:**
- Consumes: LXC 104's current DHCP-assigned IP and MAC address; the **still-local** `unifi-network` MCP server (not yet decommissioned — Task 9 does that), used to make this very change.
- Produces: a stable IP for `unifi-mcp` and a resolvable hostname `unifi-mcp.muellar.org`.

- [ ] **Step 1: Get the LXC's MAC address**

Run via `mcp__proxmox__ssh_execute(node="proxmox", command="pct config 104 | grep net0")`.

Expected: a line with `hwaddr=XX:XX:XX:XX:XX:XX`. Note it.

- [ ] **Step 2: Create the fixed-IP (DHCP reservation)**

Using `mcp__unifi-network__unifi_tool_index` (already loaded from Task 4) to confirm the current tool/parameter names, reserve a fixed IP for the MAC from Step 1 — reuse the IP already assigned via DHCP in Task 6 Step 2.

Expected: success.

- [ ] **Step 3: Create the DNS A record**

Create `unifi-mcp.muellar.org` → the fixed IP from Step 2, then immediately set `enabled: true` (same known trap as Task 4 Step 5).

- [ ] **Step 4: Verify resolution**

Run via `mcp__proxmox__ssh_execute(node="proxmox", command="getent hosts unifi-mcp.muellar.org")`.

Expected: resolves to the fixed IP. Retry once after a short wait if it doesn't resolve immediately.

---

### Task 9: Switch the local `unifi-network` MCP config to remote

**Files:**
- Modify: `C:\Users\cmuellar\AppData\Roaming\Claude\claude_desktop_config.json`

**Interfaces:**
- Consumes: the working HTTP endpoint from Task 7, the resolvable hostname from Task 8.
- Produces: all three targeted MCP servers now run remotely on Proxmox; nothing further depends on this.

- [ ] **Step 1: Back up the config file**

Run: `cp "/c/Users/cmuellar/AppData/Roaming/Claude/claude_desktop_config.json" "/c/Users/cmuellar/AppData/Roaming/Claude/claude_desktop_config.json.bak-2026-08-20b"` (skip if Task 5 Step 1's backup is still fresh and you're editing the same session).

- [ ] **Step 2: Edit the `unifi-network` entry**

Using the Edit tool, change the `unifi-network` block from:

```json
"unifi-network": {
  "command": "uvx",
  "args": ["unifi-network-mcp@latest"],
  "env": {
    "UNIFI_NETWORK_HOST": "unifi.muellar.org",
    "UNIFI_NETWORK_USERNAME": "mcp-bot",
    "UNIFI_NETWORK_PASSWORD": "..."
  }
}
```

to:

```json
"unifi-network": {
  "command": "npx",
  "args": ["-y", "mcp-remote", "http://unifi-mcp.muellar.org:3000/mcp", "--allow-http"]
}
```

- [ ] **Step 3: Restart Claude Desktop and verify**

Ask cmuellar to fully quit and reopen Claude Desktop. Once reconnected, call `mcp__unifi-network__unifi_tool_index` and then one real read tool (e.g. list clients) and confirm it returns real data.

Expected: tool calls succeed with real UniFi data, same as before the migration.

- [ ] **Step 4: Update memory**

Update `mcp_server_locations.md` to reflect that `unifi-network` now runs via `mcp-remote` against LXC 104 `unifi-mcp`. Note in `ha_mcp_server_infra.md` (or a new sibling memory file) that `paperless-mcp` (LXC 103) and `unifi-mcp` (LXC 104) exist and follow the same `docker compose pull/up` update pattern as `ha-mcp`.

---

## Self-Review Notes

- **Spec coverage:** LXC creation/specs (Tasks 1, 6), Docker (Tasks 2, 7), per-service deployment (Tasks 3, 7), secrets handling matching `ha-mcp` precedent (Tasks 3, 7), DNS naming convention (Tasks 4, 8), client config cutover (Tasks 5, 9), rollout order paperless-first-then-unifi (Task numbering) — all spec sections are covered. proxmox-mcp is explicitly excluded per spec.
- **Deviation from spec noted inline:** the spec mentioned port 8087 for `paperless-mcp`, carried over from an earlier draft where it was going to share LXC 101 with `ha-mcp` (port 8086 was taken). Since it now gets its own LXC, there's no collision reason left, so this plan uses the package's actual default port 3000 instead — simpler, no `--port` flag needed, verified against the real CLI parsing code.
- **No placeholders:** all commands, file contents, env var names, ports, and paths are concrete and were verified against the actual installed packages (`paperless-ngx-mcp`'s `build/index.js`, `unifi-network-mcp`'s `config.yaml` + `bootstrap.py`) rather than assumed. The only intentionally-elided values are the two real secrets (Paperless token, UniFi password), which must never be committed to this repo — each task states exactly where to read the real value from at execution time.
