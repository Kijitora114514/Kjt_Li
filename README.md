Kjt_Li

This repository is basically a storage space for my various projects.

For now, it mainly contains an experimental out-of-tree ZMK/Zephyr board module for the FDK HY0020 nRF52832 BLE module.

Other modules, features, or random things may be added here in the future.

## What Is HY0020?

HY0020 is a compact BLE module based on the Nordic nRF52832 QFAA. This repository provides a reusable ZMK board definition for HY0020 so keyboard projects can use it as a split peripheral or other BLE-only ZMK target.

## Purpose

This repository is intended to be added to a user's `zmk-config` as a west module. It provides only the HY0020 board definition and does not include any keyboard-specific matrix, keymap, shield, or dongle configuration.

The board definition is based on a real ZMK keyboard build that has been tested on HY0020 hardware, including BLE reconnection after peripheral reboot.

## Using HY0020 In ZMK

Add this repository as a module in your `zmk-config/config/west.yml`.

```yaml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
    - name: Kjt_Li
      url-base: https://github.com/Kjt_Li

  projects:
    - name: zmk
      remote: zmkfirmware
      revision: main
      import: app/west.yml
    - name: Kjt_Li
      remote: Kjt_Li
      revision: main

  self:
    path: config
```

Then use `hy0020/nrf52832` as the board in your ZMK build matrix.

```yaml
include:
  - board: hy0020/nrf52832
    shield: your_keyboard_left
    artifact-name: your_keyboard_left-hy0020
    cmake-args: -DCONFIG_ZMK_SPLIT=y -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n

  - board: hy0020/nrf52832
    shield: your_keyboard_right
    artifact-name: your_keyboard_right-hy0020
    cmake-args: -DCONFIG_ZMK_SPLIT=y -DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n
```

HY0020 has no USB interface in this board definition, so it is intended for BLE builds and SWD flashing.

If you build HY0020 firmware with GitHub Actions, make sure your `zmk-config` workflow asks ZMK's reusable workflow to keep `.hex` output. This module only provides the board definition; it does not control artifact packaging in repositories that consume it. The build file should be somewhere around `.github/workflows/build.yml`.

```yaml
jobs:
  build:
    uses: zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3
    with:
      build_matrix_path: build.yaml
      fallback_binary: hex
```

Without `fallback_binary: hex`, another repository may be able to build HY0020 successfully but still not include the `.hex` file in the downloaded artifact ZIP.

## Flash Layout

HY0020 uses the nRF52832 QFAA 512 KiB internal flash.

| Partition | Address | Size | Purpose |
| --- | ---: | ---: | --- |
| `app_partition` | `0x00000000` | `0x00078000` / 480 KiB | Direct SWD application image |
| `storage_partition` | `0x00078000` | `0x00008000` / 32 KiB | ZMK/Zephyr settings and BLE bond storage |

There is no bootloader partition in this layout. Firmware starts at address `0x00000000`.

## BLE Bond And Settings

This board definition uses Zephyr Settings with the NVS backend:

```conf
CONFIG_FLASH=y
CONFIG_FLASH_PAGE_LAYOUT=y
CONFIG_FLASH_MAP=y
CONFIG_NVS=y
CONFIG_SETTINGS_NVS=y
CONFIG_MPU_ALLOW_FLASH_WRITE=y
```

The devicetree also explicitly routes Settings storage to the final 32 KiB flash partition:

```dts
zephyr,settings-partition = &storage_partition;
```

These settings are important for BLE bond persistence. Without a persistent Settings backend, a ZMK build may end up with:

```conf
CONFIG_SETTINGS=y
CONFIG_BT_SETTINGS=y
CONFIG_SETTINGS_NONE=y
```

That configuration can pair initially, but BLE bond data will not be restored after power loss or reset. Do not change this module to another Settings backend unless you have tested BLE bond persistence on real HY0020 hardware.

## Flashing With SWD

Program the generated `.hex` file through SWD using your preferred nRF52-compatible probe and tool, such as a J-Link, CMSIS-DAP probe, pyOCD, or Nordic tooling.

Typical SWD signals required:

- SWDIO
- SWCLK
- GND
- VTref / target voltage reference
- RESET, if available

Because this layout has no bootloader, erase and program the application image directly to internal flash.

## Continuous Integration

This repository includes a minimal GitHub Actions workflow that builds:

```yaml
board: hy0020/nrf52832
shield: settings_reset
```

The CI is intended to catch breakage in board discovery and basic ZMK compilation. It does not replace testing with a real keyboard shield and real HY0020 hardware.

## Status

This is experimental support. The board definition has been tested with a specific HY0020-based ZMK split peripheral setup, including BLE reconnection after peripheral reboot, but broader hardware variations and workflows may still need validation.

Real-device test reports, fixes, and issues are welcome.

## Maintenance

I may update this project when I feel like it. Or I may not.

Hopefully, someone smarter than me will come along and improve it.

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.
