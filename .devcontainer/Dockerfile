FROM ghcr.io/esphome/devcontainer:addons

# Install uv directly into the container's executable path
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

# Set up global virtual environment path if needed (matching base image paths)
ENV PATH="/opt/venv/bin:$PATH"

# Ensure the non-root vscode user can manage packages in the virtual environment
RUN chown -R vscode:vscode /opt/venv || true
