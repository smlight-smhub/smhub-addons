# SMLIGHT SMHUB Home Assistant Add-ons

[![Install the smhub esphome addon repository.](https://my.home-assistant.io/badges/supervisor_add_addon_repository.svg)](https://my.home-assistant.io/redirect/supervisor_add_addon_repository/?repository_url=https%3A%2F%2Fgithub.com%2Fsmlight-smhub%2Fsmhub-addons)

This repository contains custom Home Assistant Add-ons for the SMLIGHT SMHUB ecosystem.

## Add-ons Included

### 1. [ESPHome for SMHUB](./esphome-smhub/)
A custom fork of ESPHome Device Builder, pre-configured with the toolchain and SDK packages required to build and deploy firmware natively on the SG2000 C906L RTOS coprocessor.

---

## How to use in Home Assistant

1. Go to **Settings** > **Add-ons** > **Add-on Store**.
2. Click the three dots menu in the top-right corner, and click **Repositories**.
3. Paste the URL of this repository: `https://github.com/smlight-smhub/smhub-addons`
4. Click **Add**, then close the dialog.
5. The store will refresh, showing the **ESPHome for SMHUB** add-on under the **SMLIGHT SMHUB** category.
6. Install and start the add-on!

---

## Development and Standalone Compilation

For instructions on how to compile configurations or run the dashboard independently of Home Assistant (using the standalone Docker container or a local Python virtual environment), see [DEVELOPMENT.md](./DEVELOPMENT.md).
