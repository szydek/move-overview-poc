# MOVE Overview POC

A minimal web dashboard for [Ableton Move](https://www.ableton.com/en/move/) that runs **directly on the device** — no laptop required after deployment.

Built as a proof of concept for integrating Move set management into [Schwung](https://github.com/charlesvestal/schwung). Runs as a standalone Flask app on port 808 alongside Schwung (port 7700).

## Features

- **32-pad grid** — mirrors your Move's pad layout with set names, BPM, and color
- **Active pad indicator** — pulsing gold highlight on the currently loaded set, updates every 2 seconds in real time
- **Disk usage bar** — storage visualization for `/data/UserData`
- **Sets table** — sortable list with BPM, key, mode, and Camelot code
- **Key filtering** — filter sets by Camelot key
- **Move color palette** — pad colors match what you see on the hardware

## Prerequisites

- Ableton Move running firmware with SSH enabled
- [Schwung](https://github.com/charlesvestal/schwung) installed (or SSH access enabled via `http://move.local/development/ssh`)
- Python 3.9+ on Move (comes with Schwung)
- Your Mac/Linux machine on the same WiFi network as Move

## Deployment

### 1. Clone this repo on your Mac

```bash
git clone https://github.com/szydek/move-overview-poc.git
cd move-overview-poc
```

### 2. Copy files to Move

```bash
# Create directory structure on Move
ssh ableton@move.local "mkdir -p /data/UserData/move-overview-poc/services /data/UserData/move-overview-poc/utils /data/UserData/move-overview-poc/templates"

# Copy all files
scp -r app.py config.py requirements.txt services/ utils/ templates/ \
    ableton@move.local:/data/UserData/move-overview-poc/
```

### 3. Set up Python environment on Move

```bash
ssh ableton@move.local

# On Move:
cd /data/UserData/move-overview-poc
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### 4. Run the app

```bash
# Still on Move, in the venv:
python app.py
```

You should see:
```
 * Running on http://0.0.0.0:808
```

### 5. Open in browser

Navigate to:
```
http://move.local:808
```

## Auto-start with systemd (Optional)

To have the app start automatically when Move boots:

```bash
# On Move
sudo tee /etc/systemd/system/move-overview.service > /dev/null << 'EOF'
[Unit]
Description=MOVE Overview
After=network.target

[Service]
Type=simple
User=ableton
WorkingDirectory=/data/UserData/move-overview-poc
Environment="PATH=/data/UserData/move-overview-poc/venv/bin"
ExecStart=/data/UserData/move-overview-poc/venv/bin/python app.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable move-overview
sudo systemctl start move-overview
```

Check status:
```bash
sudo systemctl status move-overview
sudo journalctl -u move-overview -f
```

## Updating

When you change files locally, push them to Move with `scp`:

```bash
# Template only (no restart needed — Flask hot-reloads in debug mode)
scp templates/index.html ableton@move.local:/data/UserData/move-overview-poc/templates/

# Python files (restart required)
scp app.py ableton@move.local:/data/UserData/move-overview-poc/
scp services/move_connection.py ableton@move.local:/data/UserData/move-overview-poc/services/
```

## Architecture

```
move-overview-poc/
├── app.py                    # Flask app, routes: /, /api/data, /api/active-slot
├── config.py                 # Settings (sets root path, etc.)
├── requirements.txt          # Dependencies: flask only
├── services/
│   ├── move_connection.py    # Local filesystem access (no SSH — runs on device)
│   └── bundle_parser.py      # Parses Song.abl JSON for BPM/key/scale
├── utils/
│   └── camelot.py            # Camelot wheel key mapping
└── templates/
    └── index.html            # Single-page UI (vanilla JS, no build step)
```

**Key design decisions:**
- **No SSH/SFTP** — reads `/data/UserData` directly since the app runs on Move itself
- **No frontend build step** — plain HTML/CSS/JS served by Flask
- **Two polling rates** — full data refresh every 30s, active pad slot every 2s
- **`os.getxattr()`** — reads pad position and color xattrs via Python stdlib (no subprocess)

## Ports

| Service | Port |
|---------|------|
| Schwung | 7700 |
| MOVE Overview POC | 808 |
| extending-move (if installed) | 909 |

## Troubleshooting

**`ImportError: cannot import name 'MoveConnection'`**
You have an old `app.py` on Move. Re-copy it:
```bash
scp app.py ableton@move.local:/data/UserData/move-overview-poc/
```

**Pad grid shows no colors / all pads empty**
Check that xattrs are readable:
```bash
python3 -c "import os; print(os.getxattr('/data/UserData/UserLibrary/Sets/$(ls /data/UserData/UserLibrary/Sets/ | head -n 1)', 'user.song-index'))"
```

**Active pad not highlighting**
Check `currentSongIndex` exists in Settings.json:
```bash
python3 -c "import json; d=json.load(open('/data/UserData/settings/Settings.json')); print(d.get('currentSongIndex'))"
```

**Port 808 already in use**
Change the port in `app.py` (last line) to any unused port, e.g. `8080`.

## Related Projects

- [Schwung](https://github.com/charlesvestal/schwung) — open framework for Ableton Move (synths, FX, tools)
- [extending-move](https://github.com/gonelink/extending-move) — Python Flask runtime for Move (port 909)
- [Music Producer Toolkit](https://github.com/guerrilladigital/music-producer-toolkit) — the full desktop toolkit this POC was derived from
