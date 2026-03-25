# Project N.O.M.A.D. — Setup & Migration Notes

## 1. Pre-launch

Edit `.env` and set real passwords before first start:

```bash
cd ~/apps/project-nomad
nano .env          # set DB_PASSWORD and DB_ROOT_PASSWORD
docker compose up -d
```

Verify the Command Center is up at `http://192.168.50.128:8283`.

---

## 2. Cloudflare Tunnel — Public Access

Your existing daedalus tunnel (`737cefd4-dfac-47df-bdbf-d58506dbcd51`) handles this with the same
pattern as every other subdomain.

### 2a. Public Hostname
Zero Trust → Networks → Tunnels → `daedalus` → Public Hostnames → Add a public hostname:

```
Subdomain: nomad
Domain:    chrislawrence.ca
Path:      (leave blank)
Type:      HTTP
URL:       caddy:80
```

### 2b. Access Application
Zero Trust → Access → Applications → Add an application → Self-hosted:

```
Name:    Project N.O.M.A.D.
Domain:  nomad.chrislawrence.ca
Policy:  Admin   ← your personal-only policy
         + Blocked (China/Russia) if desired
```

---

## 3. NOMAD First-Run Wizard — AI Integration

When the wizard asks for AI backend settings, **do not install new Ollama/Qdrant**.
Both are already running on the shared homelab-web network as part of Zeus:

| Service | Container        | Internal URL                  |
|---------|-----------------|-------------------------------|
| Ollama  | `zeus-ollama`   | `http://zeus-ollama:11434`    |
| Qdrant  | `zeus-qdrant`   | `http://zeus-qdrant:6333`     |

This reuses existing models and avoids a second GPU/VRAM allocation.

---

## 4. Kiwix / Offline-Wiki Migration

**Goal**: Migrate your existing ZIM library into NOMAD's Kiwix instance, then retire
the standalone `offline-wiki` container so only one Kiwix is running.

### 4a. Existing ZIM data locations

From `~/services/offline-wiki/compose.yaml`:

| Mount in container | Host path                         |
|--------------------|-----------------------------------|
| `/data`            | `/mnt/storage/kiwix`              |
| `/data/zims`       | `/mnt/hermes/appdata/kiwix/zims`  |

The library index is at `/mnt/storage/kiwix/library.xml`.

### 4b. After NOMAD installs its Kiwix

NOMAD manages Kiwix as a container it launches itself. Once the Kiwix service is
installed via the NOMAD Command Center UI:

1. **Find where NOMAD put the Kiwix data volume**:
   ```bash
   docker inspect nomad-kiwix 2>/dev/null || docker ps --format '{{.Names}}' | grep -i kiwix
   # Then inspect that container:
   docker inspect <nomad-kiwix-container> | grep -A5 Mounts
   ```

2. **Copy / symlink ZIM files into NOMAD's Kiwix data path**:
   ```bash
   # Example — adjust NOMAD_KIWIX_PATH to what you found above
   NOMAD_KIWIX_PATH=/path/to/nomad/kiwix/data

   # Option A: Hard-copy (safest, works even if original is removed)
   rsync -av --progress /mnt/hermes/appdata/kiwix/zims/ "$NOMAD_KIWIX_PATH/zims/"

   # Option B: Symlink (saves disk space, both must stay mounted)
   ln -s /mnt/hermes/appdata/kiwix/zims "$NOMAD_KIWIX_PATH/zims"
   ```

3. **Restart NOMAD's Kiwix container** so it picks up the new ZIMs:
   ```bash
   docker restart <nomad-kiwix-container>
   ```

4. **Verify** the ZIM library loads in the NOMAD UI at `https://nomad.chrislawrence.ca`.

### 4c. Decommission standalone offline-wiki

Once NOMAD's Kiwix has your full ZIM library and you've verified it works:

**Stop the standalone container:**
```bash
cd ~/services/offline-wiki
docker compose down
```

**Comment out the Caddyfile block** (`services/daedalus-infra/proxy/Caddyfile`):
```caddy
# wiki.chrislawrence.ca:80 { ... }   ← comment out or remove
```

**Disable in services.yml** (`services/daedalus-infra/config/services.yml`):
```yaml
offline-wiki:
  status: "retired"
  enabled: false
```

**Remove from dashy/conf.yml** — delete the "Offline Wiki" entries from the three
sections (local, tailscale, public).

**Remove Cloudflare Public Hostname** for `wiki.chrislawrence.ca` (or redirect it to
`nomad.chrislawrence.ca` — Zero Trust → Tunnels → daedalus → edit `wiki` hostname,
change URL to `https://nomad.chrislawrence.ca`).

**Reload Caddy:**
```bash
docker exec caddy caddy reload --config /etc/caddy/Caddyfile
```

---

## 5. Port Reference

| Port (host) | Container port | Service                     |
|-------------|---------------|-----------------------------|
| `8283`      | `8080`        | project-nomad (Command Center) |
| `3306`      | `3306`        | project-nomad-db (MySQL, internal only) |

---

## 6. Notes

- NOMAD needs the Docker socket (`/var/run/docker.sock`) to install and manage
  sub-service containers (Kiwix, Kolibri, ProtoMaps, CyberChef, FlatNotes, etc.).
- Sub-services launched by NOMAD's wizard may not be on the `homelab-web` network;
  they will only be reachable through NOMAD's own UI proxy at `nomad.chrislawrence.ca`.
  This is by design — NOMAD is a self-contained knowledge server.
- If you want NOMAD's Kiwix exposed separately at `wiki.chrislawrence.ca` after
  migration, add a Caddy block pointing to whichever port NOMAD assigns to it.
