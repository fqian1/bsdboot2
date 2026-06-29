# Booting FreeBSD via coreboot with a Linux payload

This tutorial describes a chain‑loading method that uses a minimal Linux kernel as a coreboot payload. That kernel’s initramfs contains the FreeBSD boot loader (`loader.kboot`). When the machine starts, coreboot initialises the hardware, runs the Linux kernel, which immediately executes the FreeBSD loader. The loader then finds and boots your installed FreeBSD system from an NVMe or SATA drive.

You get the best of both worlds: coreboot’s fast, open‑source hardware initialisation and FreeBSD’s mature operating system, without needing an intermediate bootloader like GRUB.

## Overview

```
  coreboot
     │
     ▼
  Linux kernel (bzImage + embedded initramfs)
     │
     ▼
  /init  ⟹  FreeBSD loader.kboot
     │
     ▼
  FreeBSD kernel /boot/kernel/kernel
```

## Prerequisites

- A **FreeBSD machine** to build `loader.kboot` (or a cross‑build environment). You can also use a FreeBSD VM/install.
- A **Linux machine** (or the same machine with Linux in a chroot) to compile the minimal Linux kernel.
- **coreboot toolchain** – we will build two variants:
    1. Using **Libreboot’s** provided ROMs to extract Intel blobs, then building upstream coreboot.
    2. Using **Dasharo’s** coreboot fork (already includes blobs) for boards like MSI Z790‑P.
- An **external SPI flash programmer** (e.g. CH341A, FT232H) – flashing coreboot internally is often locked by the original firmware.
- Basic familiarity with the command line, kernel configuration, and editing config files.

---

## 1. Build the FreeBSD loader (`loader.kboot`)

On a FreeBSD system (or inside a FreeBSD build environment), fetch the source and build the standalone loader:

```bash
# Ensure source tree is present (e.g. in /usr/src)
cd /usr/src
make -C stand
```

After the build, the file we need is:

```
/usr/obj/usr/src/amd64.amd64/stand/loader.kboot
```

Copy this file to your Linux machine where you will build the kernel. You will later place it as `init` inside the initramfs directory.

---

## 2. Build the minimal Linux kernel (coreboot payload)

We will build a very small bzImage that contains an embedded initramfs. The kernel itself is stripped of almost everything not needed for loading the FreeBSD loader.

### 2.1 Obtain the kernel source

On your Linux build machine:

```bash
# Pick the latest stable release (check kernel.org)
wget https://cdn.kernel.org/pub/linux/kernel/v7.x/linux-7.1.2.tar.xz
tar xf linux-7.1.2.tar.xz
cd linux-7.1.2
```

### 2.2 Prepare the initramfs content

```bash
mkdir initfs
# Copy the previously built loader.kboot and rename it to 'init'
cp /path/to/loader.kboot initfs/init
chmod +x initfs/init
```

### 2.3 Create the kernel `.config`

Save the following content as a file named `defconfig` in the kernel source directory. It enables only the absolutely necessary hardware support: PCI, NVMe, SATA, EXT4, plus `devtmpfs` and `kexec` (used by the loader). ACPI, USB, input, graphics, and many other subsystems are disabled to keep the kernel tiny.

```ini
# CONFIG_LOCALVERSION_AUTO is not set
CONFIG_KERNEL_XZ=y
CONFIG_DEFAULT_HOSTNAME="linuxboot"
# CONFIG_CROSS_MEMORY_ATTACH is not set
CONFIG_HIGH_RES_TIMERS=y
# CONFIG_PREEMPT_DYNAMIC is not set
CONFIG_LOG_BUF_SHIFT=12
CONFIG_BLK_DEV_INITRD=y
CONFIG_INITRAMFS_SOURCE="./initfs"
# CONFIG_RD_GZIP is not set
# CONFIG_RD_BZIP2 is not set
# CONFIG_RD_LZMA is not set
# CONFIG_RD_LZO is not set
# CONFIG_RD_LZ4 is not set
# CONFIG_RD_ZSTD is not set
CONFIG_CC_OPTIMIZE_FOR_SIZE=y
CONFIG_EXPERT=y
# CONFIG_MULTIUSER is not set
# CONFIG_SGETMASK_SYSCALL is not set
# CONFIG_FHANDLE is not set
# CONFIG_BUG is not set
# CONFIG_PCSPKR_PLATFORM is not set
# CONFIG_FUTEX is not set
# CONFIG_EPOLL is not set
# CONFIG_SIGNALFD is not set
# CONFIG_TIMERFD is not set
# CONFIG_EVENTFD is not set
# CONFIG_SHMEM is not set
# CONFIG_AIO is not set
# CONFIG_IO_URING is not set
# CONFIG_ADVISE_SYSCALLS is not set
# CONFIG_MEMBARRIER is not set
# CONFIG_RSEQ is not set
# CONFIG_CACHESTAT_SYSCALL is not set
# CONFIG_KALLSYMS is not set
CONFIG_KEXEC=y
CONFIG_KEXEC_FILE=y
# CONFIG_CRASH_DUMP is not set
# CONFIG_X86_EXTENDED_PLATFORM is not set
# CONFIG_SCHED_OMIT_FRAME_POINTER is not set
CONFIG_HYPERVISOR_GUEST=y
CONFIG_PVH=y
# CONFIG_DMI is not set
# CONFIG_X86_MCE is not set
# CONFIG_PERF_EVENTS_AMD_UNCORE is not set
# CONFIG_X86_IOPL_IOPERM is not set
# CONFIG_MTRR is not set
# CONFIG_X86_UMIP is not set
# CONFIG_RELOCATABLE is not set
CONFIG_CMDLINE_BOOL=y
# CONFIG_MODIFY_LDT_SYSCALL is not set
# CONFIG_X86_BUS_LOCK_DETECT is not set
# CONFIG_CPU_MITIGATIONS is not set
# CONFIG_SUSPEND is not set
# CONFIG_ACPI is not set
# CONFIG_VIRTUALIZATION is not set
# CONFIG_SECCOMP is not set
# CONFIG_STACKPROTECTOR is not set
# CONFIG_RANDOMIZE_KSTACK_OFFSET is not set
# CONFIG_BLOCK_LEGACY_AUTOLOAD is not set
# CONFIG_COREDUMP is not set
CONFIG_SLUB_TINY=y
# CONFIG_COMPACTION is not set
# CONFIG_ZONE_DMA is not set
# CONFIG_VM_EVENT_COUNTERS is not set
# CONFIG_SECRETMEM is not set
CONFIG_PCI=y
CONFIG_DEVTMPFS=y
CONFIG_DEVTMPFS_MOUNT=y
# CONFIG_PREVENT_FIRMWARE_BUILD is not set
# CONFIG_FW_LOADER is not set
# CONFIG_ALLOW_DEV_COREDUMP is not set
# CONFIG_FIRMWARE_MEMMAP is not set
CONFIG_BLK_DEV_NVME=y
CONFIG_NVME_MULTIPATH=y
CONFIG_BLK_DEV_SD=y
CONFIG_ATA=y
# CONFIG_INPUT is not set
# CONFIG_SERIO is not set
# CONFIG_VT is not set
# CONFIG_UNIX98_PTYS is not set
# CONFIG_LEGACY_PTYS is not set
# CONFIG_LEGACY_TIOCSTI is not set
CONFIG_SERIAL_8250=y
# CONFIG_HW_RANDOM is not set
# CONFIG_DEVMEM is not set
# CONFIG_DEVPORT is not set
# CONFIG_HWMON is not set
# CONFIG_USB_SUPPORT is not set
# CONFIG_VIRTIO_MENU is not set
# CONFIG_VHOST_MENU is not set
# CONFIG_SURFACE_PLATFORMS is not set
# CONFIG_X86_PLATFORM_DEVICES is not set
# CONFIG_IOMMU_SUPPORT is not set
CONFIG_EXT4_FS=y
# CONFIG_FILE_LOCKING is not set
# CONFIG_DNOTIFY is not set
# CONFIG_INOTIFY_USER is not set
# CONFIG_MISC_FILESYSTEMS is not set
# CONFIG_CRC_OPTIMIZATIONS is not set
# CONFIG_XZ_DEC_POWERPC is not set
# CONFIG_XZ_DEC_ARM is not set
# CONFIG_XZ_DEC_ARMTHUMB is not set
# CONFIG_XZ_DEC_ARM64 is not set
# CONFIG_XZ_DEC_SPARC is not set
# CONFIG_XZ_DEC_RISCV is not set
# CONFIG_SYMBOLIC_ERRNAME is not set
# CONFIG_DEBUG_MISC is not set
CONFIG_FRAME_WARN=1280
# CONFIG_SECTION_MISMATCH_WARN_ONLY is not set
# CONFIG_FTRACE is not set
# CONFIG_X86_VERBOSE_BOOTUP is not set
# CONFIG_EARLY_PRINTK is not set
# CONFIG_X86_DEBUG_FPU is not set
CONFIG_UNWINDER_GUESS=y
# CONFIG_RUNTIME_TESTING_MENU is not set
```

Now apply the configuration and build the kernel:

```bash
cp defconfig .config
make olddefconfig
make -j$(nproc)
```

If successful, the resulting payload is:

```
arch/x86_64/boot/bzImage
```

This file is a self‑contained Linux kernel with the FreeBSD loader inside.

---

## 3. Build coreboot with the Linux payload

The final coreboot image must contain the Intel firmware blobs (descriptor, ME, GbE) and our `bzImage` as the payload. We cover two common approaches: using Libreboot’s prebuilt ROMs to extract blobs, and using the Dasharo fork.

### A. Using Libreboot’s release ROM (extract blobs, then build upstream coreboot)

This method works for many older Intel boards supported by Libreboot.

#### Step A1: Get the Libreboot ROM

```bash
git clone https://codeberg.org/libreboot/lbmk.git
cd lbmk
./mk dependencies

# Download the appropriate release archive for your board.
# Replace [board] with your board’s codename (e.g., x230, t440p, ...)
wget https://rsync.libreboot.org/stable/26.01rev1/roms/libreboot-26.01rev1_[board].tar.xz
```

#### Step A2: Inject a random MAC address and extract the ROM

```bash
./mk inject libreboot-26.01rev1_[board].tar.xz setmac
tar xf libreboot-26.01rev1_[board].tar.xz
# The Libreboot ROM is now in bin/ or a similarly named directory.
cp bin/*/libreboot-26.01rev1*.rom libreboot.rom
```

#### Step A3: Extract the Intel firmware blobs

```bash
git clone --depth 1 https://review.coreboot.org/coreboot.git
cd coreboot
make crossgcc-i386 CPUS=$(nproc)
make -C util/ifdtool

# Place the Libreboot ROM in the coreboot directory
cp ../lbmk/libreboot.rom .

# Extract the regions
util/ifdtool/ifdtool -x libreboot.rom
mkdir binaries
mv flashregion_0_flashdescriptor.bin binaries/ifd.bin
mv flashregion_2_intel_me.bin    binaries/me.bin
mv flashregion_3_gbe.bin         binaries/gbe.bin
```

#### Step A4: Create a coreboot config

Copy your `bzImage` into the coreboot root directory:

```bash
cp /path/to/bzImage .
```

Now create a `defconfig` file with the following content, adjusting `CONFIG_VENDOR_*` and `CONFIG_BOARD_*` to match your board exactly (use `make menuconfig` first to see the exact names). The critical line is `CONFIG_PAYLOAD_LINUX=y` which tells coreboot to treat `./bzImage` as a Linux payload.

```ini
CONFIG_COMPRESS_RAMSTAGE_ZSTD=y
CONFIG_VENDOR_<VENDOR>=y             # e.g. CONFIG_VENDOR_LENOVO=y
CONFIG_DISABLE_HECI1_AT_PRE_BOOT=y
CONFIG_IFD_BIN_PATH="binaries/ifd.bin"
CONFIG_ME_BIN_PATH="binaries/me.bin"
CONFIG_GBE_BIN_PATH="binaries/gbe.bin"
CONFIG_HAVE_IFD_BIN=y
CONFIG_BOARD_LENOVO_<BOARD>=y        # e.g. CONFIG_BOARD_LENOVO_X230=y
CONFIG_USE_INTEL_FSP_MP_INIT=y
CONFIG_HAVE_ME_BIN=y
CONFIG_HAVE_GBE_BIN=y
CONFIG_NO_GFX_INIT=y
# CONFIG_TPM2 is not set
CONFIG_PAYLOAD_LINUX=y               # coreboot looks for ./bzImage
```

Apply and build:

```bash
cp defconfig .config
make olddefconfig
make -j$(nproc)
```

The final coreboot image is `build/coreboot.rom`. Flash it with an external programmer.

---

### B. Using Dasharo coreboot (for newer Intel boards, e.g. MSI Z790‑P DDR5)

Dasharo provides a coreboot fork with mainboard support and pre‑packaged blobs.

#### Step B1: Clone Dasharo coreboot and fetch blobs

```bash
git clone https://github.com/Dasharo/coreboot.git -b msi_ms7e06_v0.9.4
cd coreboot
# The build script will download necessary submodules and blobs
./build.sh z790p_ddr5   # this may fail or produce a ROM, it only initialises the tree
```

*Note: The `build.sh` script ensures all blobs are placed under `3rdparty/dasharo-blobs/`. You can stop it after it has fetched the blobs if you don’t want the default ROM.*

#### Step B2: Customise the configuration

Copy your `bzImage` into `payload/bzImage` (or any path, but we’ll use `payload/bzImage` as in the defconfig):

```bash
mkdir -p payload
cp /path/to/bzImage payload/bzImage
```

Now create a `defconfig` that includes Dasharo‑specific options (EFI variable store, SMMSTORE, vboot FMAP, etc.) and points to the Linux payload. Adjust `CONFIG_VENDOR_*` and `CONFIG_BOARD_*` accordingly.

```ini
CONFIG_LOCALVERSION="v0.9.4"
CONFIG_LTO=y
CONFIG_VENDOR_<VENDOR>=y
CONFIG_FMDFILE="src/mainboard/$(CONFIG_MAINBOARD_DIR)/vboot-rwab.fmd"
CONFIG_ONBOARD_VGA_IS_PRIMARY=y
# CONFIG_PCIEXP_ASPM is not set
# CONFIG_PCIEXP_L1_SUB_STATE is not set
# CONFIG_PCIEXP_CLK_PM is not set
CONFIG_IFD_BIN_PATH="3rdparty/dasharo-blobs/$(MAINBOARDDIR)/descriptor.bin"
CONFIG_ME_BIN_PATH="3rdparty/dasharo-blobs/$(MAINBOARDDIR)/me.bin"
CONFIG_CONSOLE_CBMEM_BUFFER_SIZE=0x100000
CONFIG_PCIEXP_DEFAULT_MAX_RESIZABLE_BAR_BITS=37
CONFIG_HAVE_IFD_BIN=y
CONFIG_BOARD_<BOARD>=y                     # e.g. CONFIG_BOARD_MSI_Z790P_DDR5=y
CONFIG_POWER_STATE_OFF_AFTER_FAILURE=y
CONFIG_INCLUDE_HSPHY_IN_FMAP=y
CONFIG_SOC_INTEL_COMMON_OC_WDT_ENABLE=y
CONFIG_ENABLE_EARLY_DMA_PROTECTION=y
CONFIG_HAVE_ME_BIN=y
CONFIG_PCIEXP_SUPPORT_RESIZABLE_BARS=y
CONFIG_PCIEXP_LANE_ERR_STAT_CLEAR=y
# CONFIG_RESOURCE_ALLOCATION_TOP_DOWN is not set
CONFIG_DRIVERS_EFI_VARIABLE_STORE=y
CONFIG_DRIVERS_EFI_FW_INFO=y
CONFIG_DRIVERS_EFI_MAIN_FW_GUID="58301b3d-5b6d-4562-9ec1-fbce40f15253"
CONFIG_DRIVERS_EFI_MAIN_FW_VERSION=0x00090480
CONFIG_DRIVERS_EFI_MAIN_FW_LSV=0x00090480
CONFIG_DRIVERS_EFI_UPDATE_CAPSULES=y
CONFIG_SMMSTORE=y
CONFIG_SMMSTORE_V2=y
CONFIG_BOOTMEDIA_LOCK_CONTROLLER=y
CONFIG_BOOTMEDIA_LOCK_WPRO_VBOOT_RO=y
CONFIG_DEFAULT_CONSOLE_LOGLEVEL_0=y
# CONFIG_CONSOLE_USE_LOGLEVEL_PREFIX is not set
# CONFIG_CONSOLE_USE_ANSI_ESCAPES is not set
CONFIG_PAYLOAD_LINUX=y
CONFIG_PAYLOAD_FILE="payload/bzImage"
# CONFIG_COMPRESS_SECONDARY_PAYLOAD is not set
```

Apply the configuration and build:

```bash
cp defconfig .config
make olddefconfig
# Optional: run 'make menuconfig' to verify settings
make -j$(nproc)
```

The final ROM is `build/coreboot.rom`.

---

## 4. Flashing the firmware

**Warning:** Flashing a bad coreboot image can brick your machine. Always have a backup of the original firmware and a known‑good external programmer.

1. Connect your external SPI programmer to the BIOS chip (pomona clip or soldered wires).
2. Read the existing flash contents at least twice and compare checksums.
3. Write the new `coreboot.rom`:
   ```bash
   flashrom -p <programmer> -w build/coreboot.rom
   ```
4. Verify the write.

On the first boot, coreboot will run the Linux payload, which immediately starts `loader.kboot`. The FreeBSD loader will scan for a bootable UFS/ZFS filesystem on NVMe or SATA drives and proceed to load the FreeBSD kernel.

---

## Troubleshooting

- **No boot after flashing**: Double‑check that `loader.kboot` was built for the correct architecture (amd64) and that it is executable.
- **FreeBSD loader cannot find kernel**: Make sure the root filesystem is on a supported storage controller (NVMe / SATA AHCI). The minimal Linux kernel does not enable USB or VirtIO.
- **Coreboot does not recognise the payload**: The bzImage must be present in the exact path specified by `CONFIG_PAYLOAD_FILE` (or in the coreboot root if using the default). Verify the file’s integrity.
- **Missing blobs**: If coreboot complains about missing IFD/ME/GBE, ensure the paths in the defconfig point to valid binary files. Use `ifdtool -x` on a working ROM to extract them.

---

## Final notes

This method provides a lightweight, self‑contained boot chain that puts FreeBSD directly on the metal after a minimal hardware initialisation. It is especially useful for headless servers or appliances where every megabyte of firmware space counts. Modify the kernel configuration only if you need additional drivers (e.g., network booting) – the current setup targets local NVMe/SATA drives with an EXT4 `/boot` partition or a UFS/ZFS root that the FreeBSD loader understands.
