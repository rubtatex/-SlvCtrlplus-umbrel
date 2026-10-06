# SlvCtrl+ for Umbrel

An unofficial [umbrelOS](https://umbrel.com) (Umbrel 2.0) community app store that packages
[SlvCtrl+](https://github.com/SlvCtrlPlus) — a self-hosted server and web UI for controlling
serial, USB and Bluetooth LE hardware devices.

It is a port of the upstream `install-rpi.sh` script, which installs SlvCtrl+ directly on a
Raspberry Pi, into a containerized Umbrel app.

> **Note:** This project is not affiliated with the SlvCtrl+ or Umbrel teams.

## Contents

- [How it works](#how-it-works)
- [Repository layout](#repository-layout)
- [Setup](#setup)
  - [1. Publish the Docker images](#1-publish-the-docker-images)
  - [2. Add the app store to Umbrel](#2-add-the-app-store-to-umbrel)
  - [3. Configure the backend URL](#3-configure-the-backend-url)
- [Hardware access](#hardware-access)
- [Updating](#updating)
- [Troubleshooting](#troubleshooting)
- [Building locally](#building-locally)
- [Submitting to the official Umbrel App Store](#submitting-to-the-official-umbrel-app-store)

## How it works

SlvCtrl+ does not publish official Docker images, and umbrelOS never builds Dockerfiles
itself — it only pulls prebuilt images. This repository bridges the gap:

1. A GitHub Actions workflow builds the server and frontend images from the official
   SlvCtrl+ release tarballs (the same artifacts `install-rpi.sh` downloads).
2. The images are built for `linux/amd64` and `linux/arm64` and pushed to the GitHub
   Container Registry (`ghcr.io`).
3. umbrelOS pulls those images when you install the app from this community store.

The app runs three services:

| Service     | Role                                                                      |
| ----------- | ------------------------------------------------------------------------- |
| `app_proxy` | Umbrel's reverse proxy, routes the app's main page to the frontend        |
| `frontend`  | Vue web UI served by nginx (unprivileged)                                 |
| `server`    | Node.js backend on port `1337`, with access to serial/USB/Bluetooth       |

## Repository layout

```
.
├── .github/workflows/build-images.yml   # Builds and pushes images to GHCR
├── umbrel-app-store.yml                 # Community app store manifest
└── slvctrlplus/
    ├── umbrel-app.yml                   # App manifest
    ├── icon.svg                         # App icon
    ├── docker-compose.yml               # app_proxy, frontend, server
    ├── server/Dockerfile                # Node.js backend (official dist.tar.gz)
    └── frontend/
        ├── Dockerfile                   # Vue frontend served by nginx
        └── nginx.conf
```

## Setup

### 1. Publish the Docker images

This only needs to be done once.

1. Fork or push this repository to a **public** GitHub repository. umbrelOS pulls images
   anonymously, so the images must be publicly accessible.
2. Replace the `OWNER` / `REPO` placeholders with your GitHub username (lowercase) and
   repository name:
   - `slvctrlplus/docker-compose.yml`

     ```yaml
     image: ghcr.io/<owner>/slvctrlplus-server:latest
     image: ghcr.io/<owner>/slvctrlplus-frontend:latest
     ```

   - `slvctrlplus/umbrel-app.yml`

     ```yaml
     icon: https://raw.githubusercontent.com/<owner>/<repo>/main/slvctrlplus/icon.svg
     ```

3. Push to `main`. The **Build & push SlvCtrl+ images** workflow runs automatically when
   files under `slvctrlplus/server/`, `slvctrlplus/frontend/` or the workflow itself change.
   You can also start it manually from **Actions → Build & push SlvCtrl+ images → Run
   workflow**. No secrets are required; it uses the built-in `GITHUB_TOKEN`.
4. Once the workflow succeeds, go to your GitHub profile → **Packages**, open both
   `slvctrlplus-server` and `slvctrlplus-frontend`, then **Package settings → Change
   visibility → Public**.

   > This step is easy to forget. Without it, umbrelOS cannot download the images and the
   > installation fails.

### 2. Add the app store to Umbrel

1. In umbrelOS, open **Settings → App Store → Add community app store**.
2. Paste the URL of your GitHub repository.
3. **SlvCtrl+** appears in the new store — install it like any other app.

The `umbrel-app-store.yml` file at the repository root is required by umbrelOS and is
already included.

### 3. Configure the backend URL

The SlvCtrl+ web UI does **not** reach the server through the app's main page. It stores
the backend URL in the browser's `localStorage` and talks to the server directly over
REST and WebSocket on port `1337`. This is how the upstream frontend works, not a
limitation of this package.

After installing, open the app, go to **Settings**, and set the backend URL to:

```
http://<umbrel-ip-or-hostname>:1337
```

Nothing will work until this is set.

## Hardware access

The `server` service runs with `privileged: true` and `network_mode: host`, reproducing
the access the upstream script gets when installed directly on the host:

- **`privileged: true`** exposes host device nodes (USB-serial adapters, GPIO UART,
  generic USB).
- **`network_mode: host`** lets Bluetooth LE talk to BlueZ over the host's D-Bus/HCI
  socket, which Docker cannot bridge into an isolated network namespace. The official
  `ee-gateway` Umbrel app uses the same approach for Bluetooth.

This is broader access than a typical Umbrel app needs, and is justified only because
SlvCtrl+ exists to talk to local hardware. If you only use a USB-serial adapter and never
Bluetooth, `docker-compose.yml` documents a more restricted alternative without host
networking.

**GPIO serial (hardware UART):** if your device is wired to the Raspberry Pi's GPIO serial
pins rather than plugged in over USB, enable the UART once on the host, outside of Umbrel:

```bash
sudo raspi-config   # Interface Options → Serial Port
```

## Updating

umbrelOS does not re-pull a `:latest` image that already exists locally, so a simple
**Restart** will not pick up new images. After the workflow has published a new build,
force a pull over SSH:

```bash
cd ~/umbrel/app-data/slvctrlplus
sudo docker compose pull
sudo docker compose up -d --force-recreate
```

## Troubleshooting

### Installation fails with `app-script install slvctrlplus` exit code 1

This message is only the outer error. Common causes:

- The images are not published yet, or are still **private** on GHCR (see
  [step 1](#1-publish-the-docker-images)).
- The `OWNER` placeholder in `docker-compose.yml` was not replaced.

The real cause is in the umbreld logs:

```bash
tail -n 200 ~/umbrel/logs/umbreld.log
```

### The UI keeps showing "Cannot connect to server"

1. Make sure the backend URL is set in the app's **Settings** (see
   [step 3](#3-configure-the-backend-url)).
2. Check that the server is running and listening:

   ```bash
   sudo docker logs --tail 50 slvctrlplus_server_1
   sudo ss -tlnp | grep 1337
   ```

If the logs show `libasound.so.2: cannot open shared object file`, you are running an old
server image. The current `server/Dockerfile` installs the system libraries required by the
native Node addons (`libasound2`, `libusb-1.0-0`, `libudev1`, `libbluetooth3`,
`libdbus-1-3`). Rebuild the images and [update](#updating).

## Building locally

To check that the Dockerfiles build before going through GitHub Actions:

```bash
cd slvctrlplus
docker build -t slvctrlplus-server server
docker build -t slvctrlplus-frontend frontend
```

The server build pulls the latest upstream release by default. To pin a specific version
for reproducible builds:

```bash
docker build --build-arg SLVCTRLPLUS_SERVER_REF=v1.4.0 -t slvctrlplus-server server
```

## Submitting to the official Umbrel App Store

This package is intended for personal use through a community store. A submission to
[`getumbrel/umbrel-apps`](https://github.com/getumbrel/umbrel-apps) would additionally
require:

- Images pinned by digest (`image: ...@sha256:...`) rather than `:latest`.
- A strong justification for `privileged: true` and `network_mode: host` during review.
