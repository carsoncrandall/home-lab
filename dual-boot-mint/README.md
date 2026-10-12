# Dual Boot: Windows 10 and Linux Mint on Legacy BIOS Hardware

## Objective

With Windows 10 approaching end of support, I wanted to learn Linux before committing to a full switch. I set up a dual boot on my laptop so I could use Linux Mint day to day while keeping Windows 10 available. (Microsoft later announced Extended Security Updates for Windows 10, which removed the urgency.)

I ran the dual boot setup for about six months in 2025.

## Environment

| Component | Details |
|---|---|
| Laptop | Clevo N155SD |
| Firmware | Legacy BIOS only (no UEFI) |
| Storage | 240 GB SSD (single drive), MBR partition table |
| RAM | 8 GB |
| Graphics | Hybrid Intel integrated + NVIDIA GTX 960M |
| Existing OS | Windows 10 |
| New OS | Linux Mint 21.x Cinnamon |
| Tools | Rufus, GParted, Linux Mint installer, Boot-Repair, NVIDIA X Server Settings |

## Process

### 1. Prepared the installation media

- Downloaded the Linux Mint 21.x Cinnamon ISO from the official Linux Mint site.
- Verified the ISO's SHA256 checksum in PowerShell against the official checksum file:
  ```
  Get-FileHash .\linuxmint-21.x-cinnamon-64bit.iso -Algorithm SHA256
  ```
- Wrote the ISO to a USB drive with Rufus, configured for the laptop's Legacy BIOS:
  - Partition scheme: **MBR**
  - Target system: **BIOS (or UEFI-CSM)**
  - Write mode: **ISO Image mode**

### 2. Booted the live session

- Entered the BIOS with **F2** at startup and moved the USB drive to the top of the boot order.
- Booted into the Linux Mint live session.
- Checked that the keyboard, touchpad, display, Wi-Fi and sound worked before changing anything on disk.

### 3. Freed space for Linux

- Opened GParted from the live session and selected the internal SSD (`/dev/sda`).
- Shrank the Windows 10 NTFS partition to free roughly 100 GB.
- Left the freed space unallocated for the installer.

### 4. Installed Mint alongside Windows

- Launched the Mint installer and chose **Something else** at the installation type step, to control partitioning manually and leave Windows untouched.
- In the unallocated space, created:
  - An **ext4** partition mounted at `/`
  - A **Linux swap** partition
- Set the bootloader location to the **whole disk (`/dev/sda`)**, not the Windows partition, so GRUB was written to the MBR.

Resulting layout (approximate):

```
/dev/sda  240 GB SSD (MBR)
├── Windows system/recovery partition
├── Windows 10 (NTFS)
├── Linux Mint (ext4, mounted at /)
└── Linux swap
```

### 5. Verified the dual boot

- Booted into Linux Mint and confirmed it worked normally.
- Rebooted, selected Windows 10 from GRUB and confirmed the original installation was intact.

The GRUB menu was hidden by default. Holding **Shift** during startup displayed it, with entries for Windows 10 and Linux Mint. With no input, GRUB booted the default entry.

```
Legacy BIOS
   ↓
MBR on /dev/sda
   ↓
GRUB (hidden; hold Shift to show menu)
   ├── Windows 10
   └── Linux Mint Cinnamon
```

## Issues and Troubleshooting

### Issue 1: Windows 10 disappeared from the GRUB menu

**Symptom:** Both operating systems initially appeared in GRUB. Later, the Windows 10 entry was gone, while Linux Mint still booted normally.

**Troubleshooting:**
1. Ran `sudo update-grub` from Linux Mint to regenerate the GRUB configuration. Windows was still missing.
2. Installed and ran **Boot-Repair**, then ran `sudo update-grub` again.

**Resolution:** GRUB detected Windows 10 again and the entry was restored.

**Root cause:** Not confirmed. Next time I would check whether os-prober (the tool GRUB uses to detect other operating systems) was enabled in `/etc/default/grub` before reaching for Boot-Repair.

### Issue 2: Constant screen flickering

**Symptom:** After installation, the Linux Mint desktop flickered constantly.

**Troubleshooting:**
1. Assumed a driver problem and tried several of the compatible NVIDIA drivers listed in Mint's **Driver Manager**. The flickering continued.
2. Since the laptop uses hybrid Intel + NVIDIA graphics, I looked at the GPU handoff instead. In **NVIDIA X Server Settings**, I switched the PRIME profile from **On-Demand** to **Performance Mode**.

**Resolution:** With the GTX 960M driving the display full time, the flickering stopped.

**Why it worked:** In On-Demand mode, the Intel GPU renders the desktop and the NVIDIA GPU is only used when requested. Performance Mode bypasses that switching entirely, which points to the hybrid graphics handoff as the source of the instability rather than the NVIDIA driver itself.

**Tradeoff:** Higher power draw, more heat and shorter battery life, since the NVIDIA GPU stays active for all desktop work.

## Outcome

The dual boot ran for about six months. A later attempt to resize partitions led to corrupted files, and I restored the laptop to a Windows 10 only configuration. I had not taken a backup beforehand, so data from that period was lost.

## What I'd Do Differently

- **Back up before touching partitions.** Resizing live partitions without a backup is what turned a fixable problem into data loss.
- **Upgrade the hardware first.** A larger SSD and 16 GB of RAM would have left room for both operating systems plus virtual machines.
- **Be more patient.** Several problems came from moving quickly. Slower, step-by-step changes would have been easier to troubleshoot and undo.

## Skills Practiced

- Bootable media creation and ISO integrity verification
- Legacy BIOS boot configuration and MBR partitioning
- Disk partitioning with GParted (NTFS resize, ext4, swap)
- GRUB bootloader installation and repair (`update-grub`, Boot-Repair)
- Linux driver management and hybrid GPU (NVIDIA PRIME) troubleshooting
- Isolating root causes by ruling out the obvious suspect first
