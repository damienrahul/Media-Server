# Raspberry Pi Media Server — Complete Setup Guide

Fully automated media server on a Raspberry Pi. Family members browse and request movies/TV shows, downloads happen automatically, and content appears in Plex/Jellyfin ready to watch.

## What You Get

| Service | Port | Purpose |
|---------|------|---------|
| Jellyfin | `8096` | Media player (free, open source) |
| Plex | `32400` | Media player (optional, alongside Jellyfin) |
| Jellyseerr | `5055` | Family-facing request system (browse + request) |
| Radarr | `7878` | Movie automation |
| Sonarr | `8989` | TV show automation |
| Prowlarr | `9696` | Indexer manager (feeds Radarr + Sonarr) |
| qBittorrent | `8080` | Torrent download client |
| Navidrome | `4533` | Music streaming (Subsonic-compatible) |
| Pi-hole | `80` | Network-wide ad blocker |

**Access all services at:** `http://<your-pi-hostname>.local:<PORT>`

---

## Prerequisites

- Raspberry Pi 5 (8GB recommended)
- At least one external drive mounted (e.g. `/mnt/media`)
- Docker and Docker Compose installed
- Optional: second drive for mergerfs pooling

---

## Part 1: Drive Setup & mergerfs (Optional — Multi-Drive)

If you have two drives and want them to appear as one combined pool:

### Install mergerfs

```bash
sudo apt update && sudo apt install -y mergerfs
```

### Mount the pool

```bash
sudo mkdir -p /mnt/pool

sudo mergerfs \
    /mnt/media:/mnt/media-b \
    /mnt/pool \
    -o defaults,allow_other,use_ino,category.create=mfs,moveonenospc=true,dropcacheonclose=true,cache.files=partial
```

- `category.create=mfs` — new files go to the drive with the most free space
- `moveonenospc=true` — if one drive fills up, overflow to the other

### Make it survive reboot — add to `/etc/fstab`

```
/mnt/media:/mnt/media-b /mnt/pool mergerfs defaults,allow_other,use_ino,category.create=mfs,moveonenospc=true,dropcacheonclose=true,cache.files=partial 0 0
```

### Create media folders

```bash
mkdir -p /mnt/media/downloads/movies
mkdir -p /mnt/media/downloads/tv
mkdir -p /mnt/media/downloads/music
mkdir -p /mnt/media/Movies
mkdir -p "/mnt/media/TV Shows"
mkdir -p /mnt/media/Music
```

> **Single drive?** Skip mergerfs entirely. Use `/mnt/media` directly instead of `/mnt/pool` everywhere below.

---

## Part 2: Docker Compose

### Create config directories

```bash
cd /srv/docker-server  # or wherever your compose lives
sudo mkdir -p prowlarr/config radarr/config sonarr/config jellyseerr/config navidrome/data
sudo chown -R 1000:1000 prowlarr radarr sonarr jellyseerr navidrome
```

### docker-compose.yml

Replace `YOUR_TIMEZONE` with your timezone (e.g. `Asia/Kolkata`, `America/New_York`).
Replace `YOUR_PIHOLE_PASSWORD` with your desired Pi-hole admin password.

```yaml
services:

  # ═══════════════════════════════════════════════
  #  MEDIA PLAYERS
  # ═══════════════════════════════════════════════

  plex:
    image: lscr.io/linuxserver/plex:latest
    container_name: plex
    network_mode: host
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=YOUR_TIMEZONE
      - VERSION=docker
    devices:
      - /dev/dri:/dev/dri
    volumes:
      - ./plex/config:/config
      - /tmp/plex/transcode:/transcode
      - /mnt/pool/Movies:/media/movies
      - /mnt/pool/TV Shows:/media/tv
      - /mnt/pool/Music:/media/music
    restart: unless-stopped

  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    user: 1000:1000
    network_mode: host
    volumes:
      - ./jellyfin/config:/config
      - ./jellyfin/cache:/cache
      - /mnt/pool/Movies:/media/movies
      - /mnt/pool/TV Shows:/media/tv
      - /mnt/pool/Music:/media/music
    restart: unless-stopped
    environment:
      - TZ=YOUR_TIMEZONE

  # ═══════════════════════════════════════════════
  #  DOWNLOAD CLIENT
  # ═══════════════════════════════════════════════

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=YOUR_TIMEZONE
      - WEBUI_PORT=8080
    volumes:
      - ./qbittorrent/config:/config
      - /mnt/media/downloads:/downloads
      - /mnt/media:/media_root
    ports:
      - 6889:6881
      - 6889:6881/udp
      - 8080:8080
    restart: unless-stopped

  # ═══════════════════════════════════════════════
  #  ARR STACK (automation)
  # ═══════════════════════════════════════════════

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=YOUR_TIMEZONE
    volumes:
      - ./prowlarr/config:/config
    ports:
      - 9696:9696
    restart: unless-stopped

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=YOUR_TIMEZONE
    volumes:
      - ./radarr/config:/config
      - /mnt/media:/mnt/media
      - /mnt/media-b:/mnt/media-b:ro    # optional second drive (read-only)
    ports:
      - 7878:7878
    restart: unless-stopped

  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=YOUR_TIMEZONE
    volumes:
      - ./sonarr/config:/config
      - /mnt/media:/mnt/media
      - /mnt/media-b:/mnt/media-b:ro    # optional second drive (read-only)
    ports:
      - 8989:8989
    restart: unless-stopped

  # ═══════════════════════════════════════════════
  #  REQUEST SYSTEM (family-facing)
  # ═══════════════════════════════════════════════

  jellyseerr:
    image: fallenbagel/jellyseerr:latest
    container_name: jellyseerr
    environment:
      - TZ=YOUR_TIMEZONE
      - LOG_LEVEL=info
    volumes:
      - ./jellyseerr/config:/app/config
    ports:
      - 5055:5055
    restart: unless-stopped

  # ═══════════════════════════════════════════════
  #  MUSIC STREAMING
  # ═══════════════════════════════════════════════

  navidrome:
    image: deluan/navidrome:0.52.5    # pinned — latest may crash on ARM64
    container_name: navidrome
    environment:
      - ND_SCANSCHEDULE=1h
      - ND_LOGLEVEL=info
      - ND_SESSIONTIMEOUT=24h
      - ND_BASEURL=""
    volumes:
      - ./navidrome/data:/data
      - /mnt/pool/Music:/music:ro
    ports:
      - 4533:4533
    restart: unless-stopped

  # ═══════════════════════════════════════════════
  #  AD BLOCKER
  # ═══════════════════════════════════════════════

  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    network_mode: host
    environment:
      TZ: "YOUR_TIMEZONE"
      FTLCONF_webserver_api_password: "YOUR_PIHOLE_PASSWORD"
    volumes:
      - /srv/pihole/etc-pihole:/etc/pihole
      - /srv/pihole/etc-dnsmasq.d:/etc/dnsmasq.d
    restart: unless-stopped
```

### Start everything

```bash
cd /srv/docker-server
sudo docker compose up -d
```

---

## Part 3: Service Configuration

**Follow this order** — each step depends on the previous one.

---

### Step 1: qBittorrent — Add Download Categories

Open **http://\<hostname\>.local:8080**

Right-click in the left sidebar → **Add Category**:
- `movies` → Save path: `/downloads/movies`
- `tv` → Save path: `/downloads/tv`
- `music` → Save path: `/downloads/music`

---

### Step 2: Prowlarr — Indexer Manager

Open **http://\<hostname\>.local:9696**

#### Add Indexers (Indexers → Add Indexer):
Search and add public indexers:
- `1337x` — general, good for movies/TV
- `EZTV` — specializes in TV shows
- `YTS` — small movie files, good for Pi storage
- `The Pirate Bay`
- `Nyaa` — anime (if needed)

> **Cloudflare-protected indexers** (like 1337x) may need FlareSolverr. Run it as a standalone container:
> ```bash
> sudo docker run -d --name flaresolverr -p 8191:8191 \
>   --network docker-server_default \
>   ghcr.io/flaresolverr/flaresolverr:latest
> ```
> Then in Prowlarr: Settings → Indexers → Add (FlareSolverr) → Host: `http://flaresolverr:8191` → Tag it → apply same tag to blocked indexers.

#### Add Download Client (Settings → Download Clients → +):
- Type: **qBittorrent**
- Host: `qbittorrent` (Docker service name)
- Port: `8080`
- Username/Password: your qBit credentials
- Test → Save

#### Add Apps (Settings → Apps → +):

Add **Radarr**:
- Prowlarr Server: `http://prowlarr:9696`
- Radarr Server: `http://radarr:7878`
- API Key: get from Radarr → Settings → General → API Key
- Test → Save

Add **Sonarr**:
- Prowlarr Server: `http://prowlarr:9696`
- Sonarr Server: `http://sonarr:8989`
- API Key: get from Sonarr → Settings → General → API Key
- Test → Save

Prowlarr auto-syncs all indexers to both Radarr and Sonarr.

---

### Step 3: Radarr — Movie Automation

Open **http://\<hostname\>.local:7878**

#### Download Client (Settings → Download Clients → +):
- Type: **qBittorrent**
- Host: `qbittorrent`
- Port: `8080`
- Category: `movies` ← must match the qBit category
- Test → Save

#### Root Folder (Settings → Media Management → Root Folders → Add):
- Path: `/mnt/media/Movies`

#### Quality Profile (Settings → Profiles):
- Recommended for Pi: stick to 1080p Bluray/WEB-DL
- Avoid 4K remuxes unless you're direct-playing everything

#### Remote Path Mapping (Settings → Download Clients → Remote Path Mappings):
- Host: `qbittorrent`
- Remote Path: `/downloads/`
- Local Path: `/mnt/media/downloads/`

#### Import Existing Movies (optional):
- Movies → Library Import → add `/mnt/media/Movies`

---

### Step 4: Sonarr — TV Show Automation

Open **http://\<hostname\>.local:8989**

#### Download Client (Settings → Download Clients → +):
- Type: **qBittorrent**
- Host: `qbittorrent`
- Port: `8080`
- Category: `tv` ← must match the qBit category
- Test → Save

#### Root Folder (Settings → Media Management → Root Folders → Add):
- Path: `/mnt/media/TV Shows`

#### Remote Path Mapping (Settings → Download Clients → Remote Path Mappings):
- Host: `qbittorrent`
- Remote Path: `/downloads/`
- Local Path: `/mnt/media/downloads/`

#### Import Existing Shows (optional):
- Series → Library Import → add `/mnt/media/TV Shows`

---

### Step 5: Jellyseerr — Request System

Open **http://\<hostname\>.local:5055**

This is the app your family uses to browse trending content and request movies/TV shows.

#### Initial Setup Wizard:
1. Choose **Jellyfin** as primary (or Plex)
2. Jellyfin URL: `http://jellyfin:8096`
   OR Plex: sign in with your Plex account

#### Connect Radarr (Settings → Radarr):
- Default Server: ON
- Hostname: `radarr`
- Port: `7878`
- API Key: from Radarr → Settings → General
- Quality Profile: your preferred one
- Root Folder: `/mnt/media/Movies`
- Test → Save

#### Connect Sonarr (Settings → Sonarr):
- Default Server: ON
- Hostname: `sonarr`
- Port: `8989`
- API Key: from Sonarr → Settings → General
- Quality Profile: your preferred one
- Root Folder: `/mnt/media/TV Shows`
- Test → Save

#### Email Notifications (Settings → Notifications → Email):
- SMTP Host: `smtp.gmail.com`
- SMTP Port: `587`
- Encryption: Use TLS
- Username: your Gmail address
- Password: Gmail App Password (see appendix)
- Enable: "Media Requested" notification

#### User Permissions (Settings → Users):
- Import users from Jellyfin
- Set request limits per user (e.g. 5 movies/week)
- Toggle auto-approve ON if you trust them, OFF to review first

---

### Step 6: Navidrome — Music Streaming

Open **http://\<hostname\>.local:4533**

1. Create admin account on first launch
2. It auto-scans your Music folder
3. Family can stream using Subsonic-compatible apps:
   - **Android**: Substreamer (Play Store, free)
   - **iOS**: play:Sub or Substreamer
   - **Desktop**: Feishin or Sonixd
   - Server URL: `http://<hostname>.local:4533`

---

### Step 7: Update Jellyfin Libraries

1. Open Jellyfin → Dashboard → Libraries
2. Edit **Movies** library → Content folder: `/media/movies`
3. Edit **TV Shows** library → Content folder: `/media/tv`
4. Add **Music** library → Content folder: `/media/music`
5. Trigger a full library scan

---

### Step 8: Update Plex Libraries

1. Open Plex → Settings → Libraries
2. Edit **Movies**: add `/media/movies`
3. Edit **TV Shows**: add `/media/tv`
4. Add **Music**: `/media/music`

---

## The Complete Automated Flow

```
Family opens Jellyseerr → browses trending content
       ↓
Clicks "Request" on a movie or show
       ↓
You get an email notification
       ↓
Open Jellyseerr → Approve (or set auto-approve)
       ↓
Jellyseerr sends to Radarr (movie) or Sonarr (TV)
       ↓
They search Prowlarr indexers for the best torrent
       ↓
Torrent sent to qBittorrent → downloads to /mnt/media/downloads/
       ↓
On completion, Radarr/Sonarr hardlink + rename to Movies/ or TV Shows/
       ↓
mergerfs makes it visible in /mnt/pool/ (if using multi-drive)
       ↓
Jellyfin + Plex detect the new file → ready to watch
       ↓
Zero manual work after you hit Approve
```

---

## Remote Access — Tailscale

Access everything from your phone when away from home. No port forwarding needed.

```bash
# Install on Pi
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up

# Install Tailscale app on your phone (Play Store / App Store)
# Both devices join the same private network
# Access via http://100.x.x.x:PORT (Tailscale assigns the IP)
```

Share the Tailscale network with family so they can use Jellyseerr remotely too.

---

## Gmail App Password Setup

Gmail blocks direct SMTP login. You need an App Password:

1. Go to [https://myaccount.google.com/security](https://myaccount.google.com/security)
2. Enable 2-Factor Authentication (required)
3. Go to [https://myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords)
4. Create app password → name it "MediaServer"
5. Copy the 16-character password
6. Use this in Jellyseerr instead of your real Gmail password

---

## Useful Commands

```bash
# Start/stop everything
cd /srv/docker-server
sudo docker compose up -d          # start all
sudo docker compose down            # stop all
sudo docker compose restart radarr  # restart one service

# View logs
sudo docker compose logs -f radarr
sudo docker compose logs -f sonarr
sudo docker compose logs -f jellyseerr

# Update all containers
sudo docker compose pull && sudo docker compose up -d

# Check mergerfs pool health
df -h /mnt/pool

# Check drive health
sudo smartctl -H /dev/sda
```

---

## Known Gotchas

- **Navidrome on ARM64**: `latest` tag may crash. Pin to `deluan/navidrome:0.52.5`.
- **Cloudflare-blocked indexers**: Add FlareSolverr (see Step 2).
- **4K playback in Plex web browser**: HEVC requires transcoding — Pi can't handle 4K transcode. Use the TV app or desktop Plex app for 4K.
- **Radarr/Sonarr can't find downloads**: You need Remote Path Mappings (see Steps 3 & 4). qBit sees `/downloads/`, but Radarr/Sonarr see `/mnt/media/downloads/`.
- **Jellyseerr can't reach Radarr/Sonarr**: Use Docker service names (`radarr`, `sonarr`) as hostnames, not `localhost` or your Pi's IP.
