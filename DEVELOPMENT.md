# ESPHome SMHUB Add-on Standalone Build Methods

This document provides a technical reference for running and compiling ESPHome RTOS configurations independently of Home Assistant. This is useful for development, CI/CD validation, and local debugging.

---

## Overview of Methods

We support two distinct methods to build and run the ESPHome compiler fork:

| Feature | Method 1: Standalone Docker Container | Method 2: Local Python Virtual Environment |
|---|---|---|
| **Primary Use Case** | Clean build environments, CI/CD validation, sandboxed execution | Rapid development, testing private repositories, interactive debugging |
| **Prerequisites** | Docker installed | Python 3.12, PlatformIO, Python Virtual Environment (`.venv`) |
| **Authentication** | Isolated (requires credentials/SSH keys injected for private git repos) | Shared (automatically inherits host SSH keys, git configuration, and credentials) |
| **Performance** | Slightly slower due to container boundaries (unless cached) | Faster, native speed with local toolchain caching |

---

## Method 1: Standalone Docker Container

This method runs the builder in a container environment using the root `Dockerfile` (located at the root of the main workspace). It packages PlatformIO, the custom `esphome-src` fork, and pre-caches the RISC-V compilation toolchain.

### 1. Build the Docker Image
To build the standalone compiler image, run the following command from the root of the `esphome-smhub` workspace:
```bash
docker build -t smhub-rtos-builder -f Dockerfile .
```

### 2. Compile a YAML Configuration
To compile a YAML configuration, mount the local configuration directory to `/config` inside the container:
```bash
docker run --rm -v $(pwd):/config smhub-rtos-builder smhub-esphome.yaml
```
* **Output:** The compiled FreeRTOS ELF firmware binary (stripped for production) will be outputted to `.pio/build/sg2000-git/smhub-rtos.elf` on your host, along with `.pio/build/sg2000-git/smhub-rtos_debug.elf` (retaining full debug symbols).

### 3. Run the Interactive Dashboard
To start the ESPHome Web Dashboard UI inside Docker, override the entrypoint and expose port `6052`:
```bash
docker run --rm -it -p 6052:6052 -v $(pwd):/config --entrypoint esphome smhub-rtos-builder dashboard /config
```
Access the dashboard via `http://localhost:6052`.

### 4. Local Developer Testing of Private Repositories & mDNS
When running the dashboard container locally against configurations that reference private git repositories (such as `platform-sg2000`) and require multicast DNS (mDNS) device discovery:

* **Use Host Networking (`--network host`)**: Ensures mDNS discovery packets broadcast and route cleanly between the host and container.
* **Forward SSH Agent**: Mount the host's SSH agent socket so git inside the container can authenticate securely without storing plaintext credentials.
* **Disable Host Key Verification**: Configure SSH inside the container to disable strict host key checking to prevent VCS cloning commands from stalling on interactive prompts.

Launch the test container using SSH Agent forwarding:
```bash
docker run -d --name smhub-dashboard-test \
  --network host \
  -v "$SSH_AUTH_SOCK:/run/ssh-agent" \
  -e SSH_AUTH_SOCK=/run/ssh-agent \
  -v $(pwd)/esphome-smhub/config:/config \
  smhub-rtos-builder \
  sh -c 'git config --global url."git@github.com:".insteadOf "https://github.com/" && git config --global core.sshCommand "ssh -o StrictHostKeyChecking=no" && exec esphome-device-builder /config'
```

Alternatively, if you prefer to authenticate using a GitHub Personal Access Token (PAT):
```bash
docker run -d --name smhub-dashboard-test \
  --network host \
  -e GH_TOKEN="YOUR_GITHUB_PAT" \
  -v $(pwd)/esphome-smhub/config:/config \
  smhub-rtos-builder \
  sh -c 'git config --global url."https://x-access-token:${GH_TOKEN}@github.com/".insteadOf "https://github.com/" && exec esphome-device-builder /config'
```

---

## Method 2: Local Python Virtual Environment

This method runs ESPHome natively on your host machine. Because it runs directly in your active shell session, it automatically inherits your terminal's authentication context, making it the preferred method when working with private or development repositories.

### Setup Instructions

We track the environment configuration using a standard `requirements.txt` file at the repository root. This allows developers to provision a virtual environment easily using either `uv` or standard Python `pip`.

#### Method 2A: Setup with `uv` (Recommended)
If you have `uv` installed, run these commands inside the `smhub-addons` directory to create the virtual environment and install the dependencies (which automatically links the local editable ESPHome fork at `../esphome-src`):
```bash
uv venv
source .venv/bin/activate
uv pip install -r requirements.txt
```

#### Method 2B: Setup with Standard `pip`
If you do not use `uv`, you can manually initialize the virtual environment using standard Python tooling:

1. **Create and activate a Python virtual environment**:
   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   ```

2. **Install the dependencies**:
   ```bash
   pip install -r requirements.txt
   ```

> [!TIP]
> **Installing directly from GitHub instead of local path:**
> If you want to run the compiler using the remote GitHub repository directly instead of a local edit/checkout (`../esphome-src`), you can run:
> ```bash
> pip install platformio
> pip install git+https://github.com/smlight-smhub/esphome.git@main
> ```

### 2. Compile the YAML Configuration
Compile the target device configuration file using the local ESPHome CLI:
```bash
esphome compile esphome-smhub/smhub-esphome.yaml
```

* **Output Paths:** The compiled SG2000 RTOS firmware files are outputted inside the ESPHome build directory:
  - **Debug ELF Binary** (retaining GDB symbols): `esphome-smhub/.esphome/build/rtos-smhub/.pioenvs/rtos-smhub/program_debug.elf`
  - **Stripped ELF Binary** (required for Linux `remoteproc` loader): `esphome-smhub/.esphome/build/rtos-smhub/.pioenvs/rtos-smhub/program.elf`
  - **Raw Binary** (for flashing): `esphome-smhub/.esphome/build/rtos-smhub/.pioenvs/rtos-smhub/program.bin`

### 3. Start the Local Dashboard Server
Launch the interactive web dashboard on your host machine:
```bash
esphome dashboard esphome-smhub/
```
Access the dashboard via `http://localhost:6052`.

---

## Method 3: VS Code DevContainer

For a fully containerized environment, you can open the `smhub-addons` workspace in VS Code using the **Dev Containers** extension. We provide two separate devcontainer configurations depending on your development flow:

### 1. ESPHome SMHUB Add-on (Production Git)
* **Use Case:** Compiling configs against the remote production GitHub repositories.
* **Config Location:** `.devcontainer/git/devcontainer.json`
* **Startup Action:** Automatically updates the virtual environment using `requirements.txt` (installs dependencies from remote Git repositories).

### 2. ESPHome SMHUB Add-on (Local Development)
* **Use Case:** Developing the custom compiler fork, platforms, or frameworks locally.
* **Config Location:** `.devcontainer/dev/devcontainer.json`
* **Startup Action:** 
  - Automatically installs development dependencies from `requirements-dev.txt` (links the local `dev/esphome-src` in editable mode).
  - Pre-stages a local `platformio_override.ini` inside `.esphome/build/rtos-smhub/` pointing to your local `dev/platform-sg2000` and `dev/framework-sg2000-rtos` submodules.

### How to Select and Open

1. Open the `smhub-addons` folder in VS Code.
2. Select **Dev Containers: Reopen in Container** from the Command Palette.
3. VS Code will present a dropdown prompt to select the configuration:
   - Choose **ESPHome SMHUB Add-on (Production Git)**
   - OR **ESPHome SMHUB Add-on (Local Development)**
4. The container will build and run the respective bootstrap script automatically on startup.

### Running the CLI/Dashboard in the DevContainer

> [!WARNING]
> **Shebang / Virtual Environment path conflict:**
> Do NOT run `.venv/bin/esphome` inside either DevContainer. The host's `.venv/` contains shebang paths unique to your host's python executable location, which will fail with a `required file not found` error inside Linux.
> 
> Instead, simply run the command directly. The container has `/opt/venv/bin/` pre-configured in the `PATH`:
> ```bash
> # Compile target config
> esphome compile esphome-smhub/smhub-esphome.yaml
> 
> # Start dashboard
> esphome dashboard esphome-smhub/
> ```

---

## Under the Hood: The Build Lifecycle

Understanding the compilation pipeline helps debug configuration or compiler issues:

1. **YAML Merging**: ESPHome parses your device YAML (e.g. `smhub-esphome.yaml`), resolves any `!include` imports (such as `common-core.yaml`), and merges them into a single configuration.
2. **C++ Code Generation**: The custom `esphome-src` package parses the merged configuration and generates C++ source code under `.esphome/build/rtos/src/`.
3. **PlatformIO Manifest Generation**: ESPHome automatically writes a `platformio.ini` file declaring dependencies on:
   - Platform definition (`platform-sg2000`)
   - Operating system (`framework-sg2000-rtos` based on FreeRTOS)
   - RISC-V compiler toolchain (`toolchain-riscv-xpack`)
4. **PlatformIO Compilation**: ESPHome invokes `platformio run` inside the build directory. PlatformIO downloads the required dependencies matching your CPU architecture (such as `linux_x86_64` or `linux_aarch64`) and links everything into the final `firmware.elf` binary.
