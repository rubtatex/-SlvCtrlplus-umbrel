# Fetches the prebuilt SlvCtrl+ frontend release tarball (same artifact
# install-rpi.sh downloads) and serves it with nginx, replacing the
# nginx-on-the-host config from the original script.
FROM node:24-bookworm-slim AS fetch

WORKDIR /src

RUN apt-get update && apt-get install -y --no-install-recommends \
        wget \
        ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Pin a specific upstream release tag here (e.g. v1.4.0) instead of "latest"
# once you've picked a known-good version, so rebuilds stay reproducible.
ARG SLVCTRLPLUS_FRONTEND_REF=latest
RUN set -eux; \
    if [ "$SLVCTRLPLUS_FRONTEND_REF" = "latest" ]; then \
        URL="https://github.com/SlvCtrlPlus/slvctrlplus-frontend/releases/latest/download/dist.tar.gz"; \
    else \
        URL="https://github.com/SlvCtrlPlus/slvctrlplus-frontend/releases/download/${SLVCTRLPLUS_FRONTEND_REF}/dist.tar.gz"; \
    fi; \
    wget -cq "$URL" -O - | tar -xz -C /src

FROM nginx:1.27-alpine

COPY --from=fetch /src/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
