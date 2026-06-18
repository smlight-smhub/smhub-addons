ARG BUILD_FROM=python:3.14-slim
FROM ${BUILD_FROM}

ARG GH_TOKEN=""

ENV DEBIAN_FRONTEND=noninteractive

RUN if [ -f /etc/alpine-release ]; then \
        apk add --no-cache git curl build-base python3 py3-pip python3-dev protobuf nginx gcompat libstdc++; \
    else \
        apt-get update && apt-get install -y --no-install-recommends \
            git \
            curl \
            build-essential \
            python3 \
            python3-pip \
            python3-venv \
            protobuf-compiler \
            nginx \
        && rm -rf /var/lib/apt/lists/*; \
    fi

ENV VIRTUAL_ENV=/opt/venv
RUN python3 -m venv $VIRTUAL_ENV
ENV PATH="$VIRTUAL_ENV/bin:$PATH"

# Force PlatformIO to use a shared global directory so caches persist regardless of $HOME changes in runners/userspace.
ENV PLATFORMIO_CORE_DIR=/opt/platformio

COPY build.yaml /tmp/build.yaml

# Pre-cache PlatformIO platform, toolchain, and framework dependencies.
RUN --mount=type=secret,id=GH_TOKEN \
    pip install platformio && \
    TOKEN=$( [ -f /run/secrets/GH_TOKEN ] && cat /run/secrets/GH_TOKEN || echo "${GH_TOKEN}" ) && \
    if [ -z "$TOKEN" ] && [ -f /tmp/build.yaml ]; then \
        TOKEN=$(grep -oE "GH_TOKEN:[[:space:]]*['\"]?[a-zA-Z0-9_-]+['\"]?" /tmp/build.yaml | sed -E "s/GH_TOKEN:[[:space:]]*['\"]?//g" | sed "s/['\"]//g"); \
    fi && \
    if [ "$TOKEN" = "YOUR_GITHUB_PAT_HERE" ]; then TOKEN=""; fi && \
    echo "Debug: Token length is ${#TOKEN}" && \
    if [ -n "$TOKEN" ]; then \
        git config --system url."https://x-access-token:${TOKEN}@github.com/".insteadOf "https://github.com/" && \
        mkdir -p /tmp/dummy-pio && \
        printf "[env:smhub]\nplatform = https://github.com/smlight-smhub/platform-sg2000.git\nboard = smhub\nframework = freertos\n" > /tmp/dummy-pio/platformio.ini && \
        platformio pkg install --project-dir /tmp/dummy-pio && \
        rm -rf /tmp/dummy-pio && \
        git config --system --remove-section url."https://x-access-token:${TOKEN}@github.com/"; \
    else \
        echo "Warning: No GitHub token provided. Skipping pre-caching of private PlatformIO dependencies."; \
    fi

COPY esphome-src/ /opt/esphome-src/

COPY esphome-src/docker/ha-addon-rootfs/ /

RUN python3 -c 'f = open("/etc/s6-overlay/s6-rc.d/esphome/run", "r"); content = f.read(); f.close(); insertion = "\nif [ -d /opt/platformio ] && [ ! -d \"${pio_cache_base}/packages\" ]; then\n    bashio::log.info \"Pre-populating PlatformIO cache...\"\n    mkdir -p \"${pio_cache_base}\"\n    cp -rp /opt/platformio/* \"${pio_cache_base}/\" || true\nfi\n"; content = content.replace("readonly pio_cache_base=/data/cache/platformio", "readonly pio_cache_base=/data/cache/platformio" + insertion); f = open("/etc/s6-overlay/s6-rc.d/esphome/run", "w"); f.write(content); f.close()' && chmod +x /etc/s6-overlay/s6-rc.d/esphome/run

COPY common-core.yaml smhub-esphome.yaml nano-esphome.yaml /opt/

# Install the fork, replacing the official ESPHome package
RUN pip install -r /opt/esphome-src/requirements.txt && \
    pip install -e /opt/esphome-src

# Pre-cache PlatformIO dependencies (platform, framework, toolchain) by compiling a dummy project
# This ensures that when the user runs the add-on, compilation is instant and doesn't require a ~250MB download
RUN --mount=type=secret,id=GH_TOKEN \
    TOKEN=$( [ -f /run/secrets/GH_TOKEN ] && cat /run/secrets/GH_TOKEN || echo "${GH_TOKEN}" ) && \
    if [ -z "$TOKEN" ] && [ -f /tmp/build.yaml ]; then \
        TOKEN=$(grep -oE "GH_TOKEN:[[:space:]]*['\"]?[a-zA-Z0-9_-]+['\"]?" /tmp/build.yaml | sed -E "s/GH_TOKEN:[[:space:]]*['\"]?//g" | sed "s/['\"]//g"); \
    fi && \
    if [ "$TOKEN" = "YOUR_GITHUB_PAT_HERE" ]; then TOKEN=""; fi && \
    echo "Debug: Token length is ${#TOKEN}" && \
    if [ -n "$TOKEN" ]; then \
        git config --system url."https://x-access-token:${TOKEN}@github.com/".insteadOf "https://github.com/" && \
        printf "esphome:\n  name: dummy\nsg2000:\n  board: smhub\n" > /tmp/dummy.yaml && \
        esphome compile /tmp/dummy.yaml || true && \
        rm -rf /tmp/dummy.yaml /tmp/.esphome && \
        git config --system --remove-section url."https://x-access-token:${TOKEN}@github.com/"; \
    fi && \
    rm -f /tmp/build.yaml && \
    mkdir -p /opt/platformio && \
    chmod -R 777 /opt/platformio
