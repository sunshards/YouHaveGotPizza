# Build and Publish

This repository builds **You Have Got Pizza** as a normal uploadable PRG32
cartridge. The portable build uses the C cartridge entry prefix
`you_have_got_pizza_c`.

## Prerequisites

- A local checkout of `riscv-prg32/PRG32`.
- A PRG32 checkout from the portable ABI-table tooling branch or newer.
- A PRG32 resident firmware build, for example `PRG32/build/PRG32.elf`, only
  when building the legacy absolute-import format.
- The RISC-V toolchain used by `PRG32/tools/prg32_game.py`.
- A running CartridgeStore instance when publishing.

Set `PRG32_REPO` if the PRG32 repository is not next to this repository:

```bash
export PRG32_REPO=/path/to/PRG32
```

## Build for ESP32-C6 Hardware

```bash
export PRG32_ARCHITECTURE=esp32c6
scripts/build.sh
```

The script writes `dist/you-have-got-pizza-esp32c6.prg32`. By default this is
a portable ABI-table cartridge and is not tied to one firmware ELF.

## Build for QEMU

```bash
export PRG32_ARCHITECTURE=qemu
scripts/build.sh
```

The script writes `dist/you-have-got-pizza-qemu.prg32`.

## Build the Legacy Absolute-Import Format

Use this only for firmware images that do not yet support portable ABI-table
cartridges:

```bash
export PRG32_PORTABLE=0
export PRG32_ARCHITECTURE=esp32c6
scripts/build.sh "$PRG32_REPO/build/PRG32.elf"
```

## Upload to a Board

```bash
python3 "$PRG32_REPO/tools/prg32_game.py" upload \
  dist/you-have-got-pizza-esp32c6.prg32 \
  --url http://192.168.4.1
```

Use the board address shown in PRG32 setup mode when the board is on classroom
Wi-Fi.

## Stage in QEMU

```bash
python3 "$PRG32_REPO/tools/prg32_game.py" upload-qemu \
  dist/you-have-got-pizza-qemu.prg32 \
  --flash "$PRG32_REPO/build-qemu/flash_image.bin" \
  --partitions "$PRG32_REPO/partitions_prg32.csv"
```

## Pack a CartridgeStore Bundle

Build both architecture variants first, then pack the flat Store bundle:

```bash
export PRG32_ARCHITECTURE=esp32c6
scripts/build.sh

export PRG32_ARCHITECTURE=qemu
scripts/build.sh

scripts/pack-store-bundle.sh
```

The bundle is `dist/you-have-got-pizza-store-bundle.zip`.

## Publish to CartridgeStore

```bash
python3 "$PRG32_REPO/tools/prg32_game.py" publish-bundle \
  dist/you-have-got-pizza-store-bundle.zip \
  --store-url http://192.168.1.42:5080 \
  --token "$PRG32_STORE_TOKEN"
```

The store URL and token may also be kept in `~/.prg32/config.json`.
