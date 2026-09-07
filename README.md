# Custom UEFI Secure Boot Lab

A hands-on Secure Boot lab that builds a custom UEFI application, signs it with a lab certificate, enrolls a PK/KEK/db trust chain in OVMF, boots it under Secure Boot, and proves signature enforcement with a tamper test.

## Architecture

```text
                    ┌──────────────────────────────┐
                    │      OVMF Secure Boot        │
                    │  OVMF_CODE_4M.secboot.fd     │
                    └──────────────┬───────────────┘
                                   │
                                   ▼
                         ┌─────────────────┐
                         │       PK        │
                         │ Platform Key    │
                         └────────┬────────┘
                                  │
                                  ▼
                         ┌─────────────────┐
                         │      KEK        │
                         │ Key Exchange    │
                         └────────┬────────┘
                                  │
                                  ▼
                         ┌─────────────────┐
                         │       db        │
                         │ Allowed Signers │
                         └────────┬────────┘
                                  │ verifies
                                  ▼
                       ┌──────────────────────┐
                       │ EFI/BOOT/BOOTX64.EFI│
                       │   Signed by db key  │
                       └──────────┬───────────┘
                                  │
                         valid signature?
                           ┌──────┴──────┐
                          YES            NO
                           │              │
                           ▼              ▼
                       Execute        Reject
```

## Build and Test Sequence

1. Write the UEFI application in `src/main.c`.
2. Compile it with GNU-EFI.
3. Link it into `build/bootloader.so`.
4. Convert it to an EFI PE/COFF application: `build/BOOTX64.EFI`.
5. Generate lab PK, KEK and db certificates/keys.
6. Sign `BOOTX64.EFI` using `keys/db.key`.
7. Create a FAT32 ESP image.
8. Place the signed loader at `EFI/BOOT/BOOTX64.EFI`.
9. Create a clean OVMF variable store.
10. Enroll PK, KEK and db and enable Secure Boot.
11. Boot QEMU with `OVMF_CODE_4M.secboot.fd`.
12. Confirm the signed loader executes.
13. Modify the signed image.
14. Replace the ESP bootloader with the tampered image.
15. Confirm Secure Boot rejects it.
16. Restore the signed image and confirm boot succeeds again.

## Current Project Layout

```text
secureboot-lab/
├── src/
│   └── main.c
├── keys/
│   ├── PK.key
│   ├── PK.crt
│   ├── KEK.key
│   ├── KEK.crt
│   ├── db.key
│   └── db.crt
├── build/
│   ├── main.o
│   ├── bootloader.so
│   ├── BOOTX64.EFI
│   ├── BOOTX64-SIGNED.EFI
│   ├── esp.img
│   └── OVMF_VARS_SECURE.fd
└── README.md
```

> **Security:** Never commit `*.key`, OVMF variable stores containing private trust material, or other secrets to Git.

## Build Commands

```bash
cd ~/secureboot-lab

gcc \
  -I/usr/include/efi \
  -I/usr/include/efi/x86_64 \
  -fpic \
  -ffreestanding \
  -fno-stack-protector \
  -fno-stack-check \
  -fshort-wchar \
  -mno-red-zone \
  -DEFI_FUNCTION_WRAPPER \
  -c src/main.c \
  -o build/main.o

ld \
  -nostdlib \
  -znocombreloc \
  -T /usr/lib/x86_64-linux-gnu/elf_x86_64_efi.lds \
  /usr/lib/x86_64-linux-gnu/crt0-efi-x86_64.o \
  build/main.o \
  -shared \
  -Bsymbolic \
  -L/usr/lib/x86_64-linux-gnu \
  -lefi \
  -lgnuefi \
  -o build/bootloader.so

objcopy \
  -j .text \
  -j .data \
  -j .rodata \
  -j .dynamic \
  -j .dynsym \
  -j .rela \
  -j .reloc \
  -O efi-app-x86_64 \
  build/bootloader.so \
  build/BOOTX64.EFI
```

## Signing

```bash
sbsign \
  --key keys/db.key \
  --cert keys/db.crt \
  --output build/BOOTX64-SIGNED.EFI \
  build/BOOTX64.EFI
```

Verify:

```bash
sbverify --list build/BOOTX64-SIGNED.EFI
```

## ESP

```bash
rm -f build/esp.img
dd if=/dev/zero of=build/esp.img bs=1M count=64
mkfs.fat -F 32 build/esp.img

mkdir -p /tmp/secureboot-esp
sudo mount -o loop build/esp.img /tmp/secureboot-esp
sudo mkdir -p /tmp/secureboot-esp/EFI/BOOT

sudo cp build/BOOTX64-SIGNED.EFI \
  /tmp/secureboot-esp/EFI/BOOT/BOOTX64.EFI

sudo umount /tmp/secureboot-esp
```

## OVMF Secure Boot

Create a fresh variable store:

```bash
cp /usr/share/OVMF/OVMF_VARS_4M.fd \
   build/OVMF_VARS_CUSTOM.fd
```

Enroll the lab trust chain with `virt-fw-vars` according to the installed version:

```text
PK  → Platform Key
KEK → Key Exchange Key
db  → Allowed signing certificate
SB  → Secure Boot enabled
```

Use the Secure Boot firmware:

```text
/usr/share/OVMF/OVMF_CODE_4M.secboot.fd
```

## QEMU Boot

```bash
qemu-system-x86_64 \
  -machine q35 \
  -m 512M \
  -drive format=raw,file=build/esp.img \
  -drive if=pflash,format=raw,readonly=on,file=/usr/share/OVMF/OVMF_CODE_4M.secboot.fd \
  -drive if=pflash,format=raw,file=build/OVMF_VARS_SECURE.fd
```

Expected output:

```text
====================================
     My Secure Bootloader v0.1
====================================
UEFI bootloader successfully started!
Firmware: UEFI
```

## Tamper Test

Do **not** start with the tampered image. First prove the correctly signed image boots.

Create a tampered copy:

```bash
cp build/BOOTX64-SIGNED.EFI \
   build/BOOTX64-TAMPERED.EFI

printf '\x90' | dd \
  of=build/BOOTX64-TAMPERED.EFI \
  bs=1 \
  seek=1000 \
  count=1 \
  conv=notrunc
```

Replace the ESP bootloader:

```bash
sudo mount -o loop build/esp.img /tmp/secureboot-esp

sudo cp build/BOOTX64-TAMPERED.EFI \
  /tmp/secureboot-esp/EFI/BOOT/BOOTX64.EFI

sudo umount /tmp/secureboot-esp
```

Boot QEMU again.

Expected result:

```text
Signed image:
  Signature valid → db trusts signer → EXECUTE

Tampered image:
  Image changed → signature invalid → REJECT
```

Restore the signed image afterward:

```bash
sudo mount -o loop build/esp.img /tmp/secureboot-esp

sudo cp build/BOOTX64-SIGNED.EFI \
  /tmp/secureboot-esp/EFI/BOOT/BOOTX64.EFI

sudo umount /tmp/secureboot-esp
```

## Learning Objectives

- UEFI application development
- PE/COFF EFI image construction
- Authenticode-style EFI signing
- PK / KEK / db trust hierarchy
- OVMF Secure Boot testing
- QEMU-based firmware experimentation
- Secure Boot signature verification
- Tamper detection and rejection

## Important Lab Notes

- This is a QEMU/OVMF lab. Do not experiment with custom PK/KEK/db enrollment on a physical machine until you understand recovery procedures.
- Keep private keys offline or protected with restrictive permissions.
- The ESP image is disposable and can be recreated.
- The OVMF variable store is part of the lab state; keep a clean backup before experimenting.
- Secure Boot validates the signed EFI image. It does not by itself guarantee that every component loaded later by the bootloader is trustworthy; the subsequent chain of trust must also be designed.

## Roadmap

- [ ] Boot signed custom loader
- [ ] Confirm Secure Boot state inside the UEFI environment
- [ ] Tamper and rejection test
- [ ] Restore and re-test
- [ ] Add a second-stage bootloader
- [ ] Verify second-stage signatures
- [ ] Add measurement / TPM concepts
- [ ] Add reproducible build scripts
- [ ] Add CI signing workflow using protected signing material
- [ ] Document key rotation and revocation
