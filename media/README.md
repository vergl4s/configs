# Self-Hosted Media Stack

Custom-WireGuard-protected download clients + full *arr automation + Jellyfin, on Docker Compose. Eleven containers, three files, no reverse proxy, no SSO.

## Services

| Service | Port | Purpose |
|---|---|---|
| Gluetun | — | WireGuard VPN (qBit + SAB share its network) |
| qBittorrent | 8200 | Torrent client (via VPN) |
| SABnzbd | 8080 | Usenet client (via VPN) |
| FlareSolverr | 8191 | Cloudflare bypass for torrent indexers |
| Prowlarr | 9696 | Indexer manager |
| Sonarr | 8989 | TV automation |
| Radarr | 7878 | Movie automation |
| Readarr | 8787 | Book automation (hotio image) |
| Bazarr | 6767 | Subtitle automation |
| Jellyfin | 8096 | Media server |
| Jellyseerr | 5055 | Request UI |

## Architecture

- **VPN scope**: only qBittorrent + SABnzbd via `network_mode: "service:gluetun"` → kill switch enforced
- **Hardlinks**: single `/data` root → downloads + library on same FS → instant moves, no copy
- **Storage**: `/data` is a **mergerfs pool** spanning multiple disks at host level; containers see one filesystem
- **No reverse proxy / no SSO**: direct LAN access by IP:port

## Prerequisites

- Linux host with Docker Engine + Compose v2 plugin
- Custom WireGuard server on a public IP
- Optional but recommended: 2+ disks for mergerfs storage pool
- Paid Usenet provider + indexer (Newshosting + NZBgeek is the standard combo)

## Storage setup (mergerfs)

On the host, before bringing up the stack:

```bash
# 1. Format new disk
sudo mkfs.ext4 -L disk2 /dev/vdb

# 2. Mount points
sudo mkdir -p /mnt/disk1 /mnt/disk2 /data

# 3. Mount disk2 by UUID (get with: sudo blkid /dev/vdb)
echo 'UUID=<your-uuid>  /mnt/disk2  ext4  defaults,nofail  0  2' | sudo tee -a /etc/fstab

# 4. Install mergerfs
sudo apt install -y mergerfs

# 5. Pool entry in /etc/fstab
echo '/mnt/disk1:/mnt/disk2  /data  fuse.mergerfs  defaults,allow_other,use_ino,cache.files=partial,dropcacheonclose=true,category.create=mfs,minfreespace=10G,fsname=mergerfs  0  0' | sudo tee -a /etc/fstab

# 6. Mount everything
sudo mount -a
df -h /data    # should show combined capacity
```

To add another disk later: format it, mount at `/mnt/disk3`, edit the mergerfs fstab line to add `:/mnt/disk3`, `sudo mount -o remount /data`. Zero downtime.

## Bring-up

```bash
# 1. Create .env and fill it in (VPN section especially)
cp .env.example .env
nano .env

# 2. Start
docker compose up -d
```

The `init` container runs once, creates the directory tree, pre-seeds `qBittorrent.conf` (wg0 binding + save paths + LAN auth whitelist), then marks itself done.

## Verify the VPN

Must pass before adding any indexers:

```bash
docker exec gluetun wget -qO- https://www.cloudflare.com/cdn-cgi/trace      # should show VPS IP
docker exec qbittorrent wget -qO- https://www.cloudflare.com/cdn-cgi/trace  # same
```

If gluetun is unhealthy: `docker logs gluetun` — usually wrong WireGuard keys or VPS firewall blocking the listen port.

---

## App configuration

Do these in order. Each step needs the previous one's API key.

### qBittorrent (`:8200`) — mostly automated
- Pre-configured: wg0 binding, save paths, LAN whitelist
- Just confirm **Tools → Options → Connection → Port = 6881**

### SABnzbd (`:8080`)
- Wizard: provider hostname / 563 / SSL / username / password
- Folders: `/data/usenet/incomplete` and `/data/usenet/complete`
- Categories: add `tv`, `movies`, `books` with `/data/usenet/complete/<cat>` folders
- **Config → Special → host_whitelist**: add `gluetun` (plus any DNS names you use)
- Grab API key from **Config → General**

### Prowlarr (`:9696`)
- Add Usenet indexer (NZBgeek): paste API key → Test → Save
- Add **Indexer Proxy → FlareSolverr**: host `http://flaresolverr:8191`, tag `flaresolverr`
- Add torrent indexers — YTS, TheRARBG, 1337x, TorrentGalaxy (tag last two with `flaresolverr`)
- **Settings → Apps → Add Application** for Sonarr, Radarr, Readarr — paste each app's API key

### Sonarr / Radarr / Readarr
- **Settings → Media Management → Root Folders**: `/data/media/tv`, `/data/media/movies`, `/data/media/books`
- **Settings → Media Management → Use Hardlinks**: ✓
- **Settings → Download Clients**:
  - SABnzbd: host `gluetun`, port `8080`, API key, category `tv`/`movies`/`books`, **Priority 1**
  - qBittorrent: host `gluetun`, port `8200`, no auth, category `tv`/`movies`/`books`, **Priority 2**
- **Settings → Profiles → Delay Profiles**: Preferred Protocol `Usenet`, Torrent Delay `30` minutes
- **Settings → Connect → Jellyfin**: host `jellyfin`, port `8096`, Jellyfin API key, enable On Import / On Upgrade / On Delete

### Bazarr (`:6767`)
- **Settings → Sonarr / Radarr**: hostname `sonarr`/`radarr`, default port, paste API key
- **Settings → Providers**: add OpenSubtitles.com (free account), Subscene, Podnapisi
- **Settings → Languages → Languages Filter**: tick English (+save)
- **Settings → Languages → Profiles → +**: name `English`, add English language, save
- **Settings → Sonarr / Radarr → Default Settings**: assign `English` profile to Series / Movies
- **Series / Movies tab → Mass Edit**: apply profile to existing library

### Jellyfin (`:8096`)
- Set up admin user
- Add libraries: Movies → `/data/media/movies`, Shows → `/data/media/tv`
- **Dashboard → API Keys → +** — create one for Radarr/Sonarr/Bazarr
- **Dashboard → Scheduled Tasks → Scan Library**: trigger hourly as fallback

### Jellyseerr (`:5055`)
- Sign in with Jellyfin account
- **Settings → Jellyfin**: `http://jellyfin:8096`
- **Settings → Services → Radarr / Sonarr**: hostname, port, API key, default root folder + quality profile
- **Settings → Users**: set Auto-Approve permission per user

## Hostname whitelist gotchas

Several apps reject incoming HTTP requests with unknown `Host:` headers (anti-DNS-rebinding). When you give them DNS names or container hostnames, you'll need to whitelist them:

- **SABnzbd**: edit `/config/sabnzbd/sabnzbd.ini`, add `gluetun` and any DNS name to `host_whitelist`, restart container
- **qBittorrent**: pre-configured via init container for `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`
- **Sonarr/Radarr/Prowlarr**: **Settings → General → URL Base** if you proxy them
- **Jellyfin/Jellyseerr**: permissive by default

## End-to-end: watch a movie

1. Request in **Jellyseerr** (phone/TV)
2. **Radarr** receives → queries **Prowlarr** → picks best release per quality profile
3. **qBit / SAB** downloads through **Gluetun** tunnel
4. **Radarr** hardlinks completed file from `/data/torrents/complete/...` (or `/data/usenet/complete/...`) into `/data/media/movies/...`
5. **Bazarr** detects new file → downloads English subtitles
6. **Radarr** pings **Jellyfin** to scan
7. **Jellyfin app on TV** (Apple TV / Android TV / Roku / Fire TV / smart TV) → play

Typical: tap-to-play in 10–30 minutes for popular content.

## Common ops

```bash
# Update
docker compose pull && docker compose up -d

# Logs
docker logs <container> --tail 100
docker compose logs -f

# Backup priorities
# - /config         : critical (DBs, settings, API keys)
# - /data/media     : optional (reproducible from indexers)
# - /data/torrents  : scratch, don't bother
# - /data/usenet    : scratch, don't bother

# Re-run init container (after deleting marker)
sudo rm /config/.mediastack-init-done && docker compose up -d init

# Check disk usage across mergerfs pool
df -h /mnt/disk1 /mnt/disk2 /data
```

## Troubleshooting

- **Gluetun unhealthy** → wrong WireGuard keys, wrong endpoint port, VPS firewall blocking
- **`ERR_CONNECTION_REFUSED` on SAB** → SAB listens on 8080, check `SABNZBD_PORT=8080` in `.env`
- **`Hostname verification failed`** → add hostname to that app's whitelist
- **Sonarr can't reach SAB/qBit** → hostname is `gluetun`, not `sabnzbd`/`qbittorrent`
- **Indexer disabled in Prowlarr** → daily API limit hit, or auth error; check `System → Events`, hit Test to re-enable
- **Cloudflare-blocked indexer** → tag it with `flaresolverr` in Prowlarr indexer settings
- **`Articles unavailable on all servers`** → DMCA takedown; blocklist in Sonarr, let it try alternates
- **`Download wasn't grabbed by Sonarr, skipping`** → don't submit NZBs manually; always trigger from Sonarr's Interactive Search
- **Sonarr keeps failing same release** → blocklist it (Activity → History → Blocklist Release)
- **Movie not appearing in Jellyfin** → set up Radarr → Jellyfin webhook (Settings → Connect)
- **Hardlinks not working** → confirm `df /data/torrents /data/media` shows same mountpoint
- **Bluray-1080p file is 15GB** → set Max Size per quality tier in Radarr or use TRaSH-Guides custom formats

## Future improvements

- **Buildarr** — declarative YAML config for all *arr settings (indexers, profiles, connections); useful if rebuilding or running multiple instances
- **TRaSH-Guides Custom Formats** — community-tuned quality scoring that prefers good encodes (x265, specific release groups) and avoids bad ones (LAMA, scene rips); see trash-guides.info
- **Sportarr** — *arr-style automation for sports (UFC, NBA, F1, etc.); slots in like another *arr, uses same Prowlarr/SAB/qBit
- **Recyclarr** — alternative to Buildarr, focused specifically on syncing TRaSH-Guides profiles
- **Tdarr** — automatic re-encoding (e.g., x264 → x265) to shrink library footprint
- **Hardware transcoding** — uncomment `/dev/dri` in Jellyfin service if host has Intel/AMD iGPU