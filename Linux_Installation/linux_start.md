## Linux startup
Power On

  ↓

BIOS

  ↓

MBR (sector 0)

  ↓

GRUB (1–2047 gap + /boot/grub)

  ↓

Kernel + initramfs

  ↓

Temporary RAM root

  ↓

Switch to real root

  ↓

systemd

  ↓

Login


### BIOS + MBR + GRUB

Sector 0        → MBR (512 bytes)

Sector 1–2047   → GRUB core.img

Sector 2048     → First partition (1MB aligned)

MBR (first 512 bytes):

Contains tiny boot code (446 bytes)

Contains partition table

Loaded by BIOS

Its only job: Load GRUB stage 1.5 from sectors 1–2047

It cannot load Linux directly.


GRUB Stages

Stage 1 → in MBR

Stage 1.5 → in sector 1–2047 gap

Stage 2 → in /boot/grub/

GRUB Stage 2:

Shows boot menu

Loads: vmlinuz initrd.img

Passes kernel parameters

initramfs is: A temporary mini Linux system loaded into RAM.

### Kernel stage

1. Mount initramfs as temporary root
2. Run /init
3. Load needed drivers
4. Find real root filesystem
5. Mount it
6. switch_root
7. Execute /sbin/init (systemd)


### Systemd stage
1. mount file system
2. start services (different level, init 3 -> cli, init 5 -> graphical)
