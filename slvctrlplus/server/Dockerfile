# Builds the SlvCtrl+ server from its prebuilt GitHub release tarball — the
# exact artifact install-rpi.sh downloads with:
#   wget ... slvctrlplus-server/releases/latest/download/dist.tar.gz
#
# Debian ("bookworm"), not Alpine, on purpose: the release tarball bundles
# precompiled native Node addons (serialport, usb, @stoprocent/noble,
# speaker) built against glibc, the same libc Raspberry Pi OS uses. Alpine's
# musl libc would not load those binaries.
FROM node:24-bookworm-slim

# libasound2/libusb-1.0-0/libudev1/libbluetooth3/libdbus-1-3: runtime shared
# libraries the native Node addons (speaker/usb/noble) dlopen() at startup —
# this is an apt-get fix, not an npm one. Without libasound2 in particular,
# the process crashes immediately with "ERR_DLOPEN_FAILED: libasound.so.2:
# cannot open shared object file", before the HTTP/WebSocket server ever
# binds to its port.
RUN apt-get update && apt-get install -y --no-install-recommends \
        wget \
        ca-certificates \
        libasound2 \
        libusb-1.0-0 \
        libudev1 \
        libbluetooth3 \
        libdbus-1-3 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Pin a specific upstream release tag here (e.g. v1.4.0) instead of "latest"
# once you've picked a known-good version, so rebuilds stay reproducible.
ARG SLVCTRLPLUS_SERVER_REF=latest
RUN set -eux; \
    if [ "$SLVCTRLPLUS_SERVER_REF" = "latest" ]; then \
        URL="https://github.com/SlvCtrlPlus/slvctrlplus-server/releases/latest/download/dist.tar.gz"; \
    else \
        URL="https://github.com/SlvCtrlPlus/slvctrlplus-server/releases/download/${SLVCTRLPLUS_SERVER_REF}/dist.tar.gz"; \
    fi; \
    wget -cq "$URL" -O - | tar -xz -C /app

# PORT, LOG_LEVEL and DATA_PATH are set via docker-compose.yml. HTTPS is not
# configured here (no SSL_KEY_FILE/SSL_CERT_FILE), matching the original
# HTTP-only install script.
EXPOSE 1337

CMD ["node", "/app/dist/index.js"]
