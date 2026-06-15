ARG BUILD_FROM
FROM ${BUILD_FROM}

RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Pre-cache PlatformIO platform, toolchain, and framework dependencies.
RUN mkdir -p /tmp/dummy-pio && \
    printf "[env:smhub]\nplatform = https://github.com/smlight-smhub/platform-sg2000.git\nboard = smhub\nframework = freertos\n" > /tmp/dummy-pio/platformio.ini && \
    platformio pkg install --project-dir /tmp/dummy-pio && \
    rm -rf /tmp/dummy-pio

COPY esphome-src/ /opt/esphome-src/

COPY common-core.yaml smhub-esphome.yaml nano-esphome.yaml /opt/

# Install the fork, replacing the official ESPHome package
# We use --no-deps to preserve all existing dependencies pre-installed in the base image
RUN pip install --no-deps --no-build-isolation --force-reinstall /opt/esphome-src

# Pre-cache PlatformIO dependencies (platform, framework, toolchain) by compiling a dummy project
# This ensures that when the user runs the add-on, compilation is instant and doesn't require a ~250MB download
RUN printf "esphome:\n  name: dummy\nsg2000:\n  board: smhub\n" > /tmp/dummy.yaml && \
    esphome compile /tmp/dummy.yaml || true && \
    rm -rf /tmp/dummy.yaml /tmp/.esphome
