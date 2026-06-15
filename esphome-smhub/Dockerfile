ARG BUILD_FROM
FROM ${BUILD_FROM}

# Install any extra build dependencies if needed
RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Copy our custom ESPHome python source fork
COPY esphome-src/ /opt/esphome-src/

# Copy the default smhub-esphome.yaml template file
COPY smhub-esphome.yaml /opt/smhub-esphome.yaml

# Install the fork, replacing the official ESPHome package
# We use --no-deps to preserve all existing dependencies pre-installed in the base image
RUN pip install --no-deps --force-reinstall /opt/esphome-src

# Pre-cache PlatformIO dependencies (platform, framework, toolchain) by compiling a dummy project
# This ensures that when the user runs the add-on, compilation is instant and doesn't require a ~250MB download
RUN printf "esphome:\n  name: dummy\nsg2000:\n  board: smhub\n" > /tmp/dummy.yaml && \
    esphome compile /tmp/dummy.yaml || true && \
    rm -rf /tmp/dummy.yaml /tmp/.esphome
