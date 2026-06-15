# We define BUILD_FROM to allow Home Assistant Add-on build system to specify the base image
ARG BUILD_FROM=ghcr.io/esphome/docker-base:debian-ha-addon-2026.06.1
ARG ESPHOME_REF=dev
ARG DEVICE_BUILDER_REF=main

# Stage 1: Build the device-builder-frontend
FROM node:24-alpine AS frontend-builder
RUN apk add --no-cache git
RUN git clone --depth 1 https://github.com/smlight-smhub/device-builder-frontend.git /frontend-src && \
    cd /frontend-src && \
    npm install && \
    npm run build

# Stage 2: Python Builder
FROM ghcr.io/esphome/docker-base:debian-2026.06.1 AS python-builder
ARG ESPHOME_REF
ARG DEVICE_BUILDER_REF
ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    git-lfs \
    openssh-client \
    curl \
    build-essential \
    python3 \
    python3-pip \
    python3-venv \
    protobuf-compiler \
    && rm -rf /var/lib/apt/lists/*

ENV VIRTUAL_ENV=/opt/venv
RUN python3 -m venv $VIRTUAL_ENV
ENV PATH="$VIRTUAL_ENV/bin:$PATH"

# Force PlatformIO to use a shared global directory
ENV PLATFORMIO_CORE_DIR=/opt/platformio

# Install PlatformIO
RUN pip install platformio

# Fetch esphome-src and device-builder from GitHub
RUN git clone --depth 1 -b ${ESPHOME_REF} https://github.com/smlight-smhub/esphome.git /opt/esphome-src && \
    pip install -r /opt/esphome-src/requirements.txt && \
    pip install -e /opt/esphome-src && \
    pip install --no-cache-dir git+https://github.com/smlight-smhub/device-builder.git@${DEVICE_BUILDER_REF}

COPY --from=frontend-builder /frontend-src /opt/device-builder-frontend
RUN pip install --no-cache-dir /opt/device-builder-frontend

# Pre-cache PlatformIO dependencies by compiling a dummy project
# Since platform-sg2000 points to a public tar.xz, no token is needed for PlatformIO here!
RUN mkdir -p /tmp/dummy-pio/src && \
    printf "[env:smhub]\nplatform = https://github.com/smlight-smhub/platform-sg2000.git\nboard = smhub\nframework = freertos\n" > /tmp/dummy-pio/platformio.ini && \
    printf "int main() { return 0; }\n" > /tmp/dummy-pio/src/main.cpp && \
    platformio run --project-dir /tmp/dummy-pio || true && \
    rm -rf /tmp/dummy-pio /tmp/.esphome && \
    chmod -R 777 /opt/platformio


# Stage 3: Standalone Docker Image
FROM ghcr.io/esphome/docker-base:debian-2026.06.1 AS standalone
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    git-lfs \
    openssh-client \
    curl \
    build-essential \
    python3 \
    python3-pip \
    python3-venv \
    protobuf-compiler \
    nginx \
    && rm -rf /var/lib/apt/lists/*

# Copy the completely prepared environment and PlatformIO cache from builder
ENV VIRTUAL_ENV=/opt/venv
ENV PATH="$VIRTUAL_ENV/bin:$PATH"
ENV PLATFORMIO_CORE_DIR=/opt/platformio
COPY --from=python-builder /opt /opt

# Copy config templates
COPY config/smhub-esphome.yaml config/nano-esphome.yaml config/.common-core.yaml /config/

EXPOSE 6052
# No s6-overlay or HA-specific init scripts
ENTRYPOINT ["esphome-device-builder", "/config", "--ha-addon-allow-public"]


# Stage 4: Home Assistant Add-on Image
FROM ${BUILD_FROM} AS ha-addon
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    git-lfs \
    openssh-client \
    curl \
    build-essential \
    python3 \
    python3-pip \
    python3-venv \
    protobuf-compiler \
    nginx \
    && rm -rf /var/lib/apt/lists/*

# Copy the completely prepared environment and PlatformIO cache from builder
ENV VIRTUAL_ENV=/opt/venv
ENV PATH="$VIRTUAL_ENV/bin:$PATH"
ENV PLATFORMIO_CORE_DIR=/opt/platformio
COPY --from=python-builder /opt /opt

# Inherits the entrypoint from the base HA Addon image
# We must copy rootfs and template configs
COPY --from=python-builder /opt/esphome-src/docker/ha-addon-rootfs/ /
COPY config/smhub-esphome.yaml config/nano-esphome.yaml config/.common-core.yaml /opt/

RUN python3 -c 'f = open("/etc/s6-overlay/s6-rc.d/esphome/run", "r"); content = f.read(); f.close(); insertion = "\nif [ -d /opt/platformio ]; then\n    bashio::log.info \"Pre-populating PlatformIO cache...\"\n    mkdir -p \"${pio_cache_base}\"\n    cp -rn /opt/platformio/* \"${pio_cache_base}/\" || true\nfi\n"; content = content.replace("readonly pio_cache_base=/data/cache/platformio", "readonly pio_cache_base=/data/cache/platformio" + insertion).replace("/config/esphome", "/config/esphome-smhub").replace("mkdir -p \"/config/esphome-smhub\"", "mkdir -p \"/config/esphome-smhub\"\ncp -rn /opt/*esphome*.yaml /config/esphome-smhub/ 2>/dev/null || true\ncp -rn /opt/.common-core.yaml /config/esphome-smhub/ 2>/dev/null || true"); f = open("/etc/s6-overlay/s6-rc.d/esphome/run", "w"); f.write(content); f.close()' && chmod +x /etc/s6-overlay/s6-rc.d/esphome/run
