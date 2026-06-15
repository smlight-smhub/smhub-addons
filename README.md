# SMLIGHT SMHUB Home Assistant Add-ons

This repository contains custom Home Assistant Add-ons for the SMLIGHT SMHUB ecosystem.

## Add-ons Included

### 1. [ESPHome for SMHUB](./esphome-smhub/)
A custom fork of ESPHome Device Builder, pre-configured with the toolchain and SDK packages required to build and deploy firmware natively on the SG2000 C906L RTOS coprocessor.

## How to use in Home Assistant

1. Go to **Settings** > **Add-ons** > **Add-on Store**.
2. Click the three dots menu in the top-right corner, and click **Repositories**.
3. Paste the URL of this repository: `https://github.com/smlight-smhub/smhub-addons`
4. Click **Add**, then close the dialog.
5. The store will refresh, showing the **ESPHome for SMHUB** add-on under the **SMLIGHT SMHUB** category.
6. Install and start the add-on!

## Development & CI/CD

The Docker image is built using GitHub Actions whenever changes are made to the compiler fork. The images are published to:
`ghcr.io/smlight-smhub/esphome-smhub-{arch}`

To build the images locally:
```bash
docker buildx build --platform linux/amd64,linux/arm64 -t ghcr.io/smlight-smhub/esphome-smhub-amd64:latest -t ghcr.io/smlight-smhub/esphome-smhub-aarch64:latest --build-arg BUILD_FROM=ghcr.io/esphome/esphome-hassio-amd64:latest .
```
