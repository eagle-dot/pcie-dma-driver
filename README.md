# PCI DMA Driver

A custom PCI DMA kernel driver and matching QEMU device model for Linux driver
development on **Jetson Orin Nano** (JetPack 6.2). Includes a full sysfs
interface (`dma_stats/` and `dma_info/`) and a userspace test program that
validates H2D and D2H transfers at ~960–1692 MB/s depending on the unit.

The device is currently conventional PCI, not PCIe, and x16 upgrade is a future step.



```
/sys/bus/pci/devices/0000:00:03.0/
├── dma_stats/
│   ├── bytes_written
│   ├── bytes_read
│   ├── irq_count
│   ├── last_latency_us
│   ├── avg_latency_us
│   └── reset_stats
└── dma_info/
    ├── buf_size
    ├── buf_phys_addr
    ├── bar0_start
    ├── bar0_size
    ├── vendor_id
    └── device_id
```

---

## Hardware and Software Requirements

| | Requirement |
|--|-------------|
| **Host hardware** | Jetson Orin Nano (tested on JetPack 6.2 / L4T R36.4.x) |
| **Host OS** | Ubuntu 22.04 |
| **Host kernel** | 5.15.x-tegra |
| **KVM** | Must be enabled (`/dev/kvm` present) |
| **QEMU** | Built from source v8.2.0 (instructions below) |
| **Guest OS** | Ubuntu 22.04 cloud image (arm64) |
| **Guest kernel** | 5.15.0-179-generic |
| **Disk space** | ~5 GB free |
| **Time** | ~20 minutes end to end |

> This project has been tested on three Jetson Orin Nano units. It will not
> work on x86 hosts without modifications to the VM launch script and driver.

---

## Repository Layout

```
pcie-dma-driver/
├── device/
│   └── pcie_dma.c          QEMU device model (inject into QEMU source tree)
├── driver/
│   ├── pcie_dma_driver.c   Linux kernel module
│   └── Makefile
├── test/
│   └── dma_test.c          Userspace validation test
└── scripts/
    └── start-vm.sh         QEMU VM launch script
```

---

## Register Map

The device exposes 8 MMIO registers in BAR0:

| Offset | Name | Description |
|--------|------|-------------|
| `0x00` | `REG_DMA_SRC` | Host DMA source address (low 32 bits) |
| `0x04` | `REG_DMA_SRC_HI` | Host DMA source address (high 32 bits) |
| `0x08` | `REG_DMA_DST` | Device buffer destination offset |
| `0x0C` | `REG_DMA_LEN` | Transfer length in bytes |
| `0x10` | `REG_DMA_CMD` | Command: `0x01`=H2D, `0x02`=D2H |
| `0x14` | `REG_DMA_STATUS` | Status: `0`=idle, `1`=busy, `2`=done, `3`=error |
| `0x18` | `REG_DMA_IRQ_MASK` | `1` = enable MSI on completion |
| `0x1C` | `REG_DMA_IRQ_ACK` | Write `1` to clear interrupt |

**Vendor ID:** `0x1234` — **Device ID:** `0xA100`

---

## Step 1 — Verify KVM on the host

```bash
ls /dev/kvm && echo "KVM ready"
zcat /proc/config.gz | grep "CONFIG_KVM="
uname -r
# Expected: 5.15.148-tegra
```

---

## Step 2 — Install host dependencies

```bash
sudo apt update
sudo apt install -y \
  qemu-system-aarch64 qemu-utils qemu-efi-aarch64 \
  cloud-image-utils git libglib2.0-dev libpixman-1-dev \
  ninja-build flex bison libslirp-dev \
  pkg-config libcap-ng-dev build-essential

sudo usermod -aG kvm $USER
newgrp kvm
```

---

## Step 3 — Set up workspace and firmware

```bash
mkdir -p ~/vm-driver-dev/{vm,firmware,driver_share/{driver,test,qemu-dma-dev}}

cp /usr/share/qemu-efi-aarch64/QEMU_EFI.fd ~/vm-driver-dev/firmware/QEMU_EFI.img
truncate -s 64M ~/vm-driver-dev/firmware/QEMU_EFI.img
dd if=/dev/zero of=~/vm-driver-dev/firmware/varstore.img bs=1M count=64
```

---

## Step 4 — Create the guest disk image

```bash
cd ~/vm-driver-dev/vm
wget -c https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-arm64.img
qemu-img resize jammy-server-cloudimg-arm64.img 10G

cat > user-data << 'EOF'
#cloud-config
hostname: vmguest
users:
  - name: dev
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    lock_passwd: false
    plain_text_passwd: 'dev123'
ssh_pwauth: true
EOF

printf "instance-id: orin-vm01\nlocal-hostname: vmguest\n" > meta-data
cloud-localds seed.img user-data meta-data
```

---

## Step 5 — Copy driver files from this repo

```bash
cp driver/pcie_dma_driver.c ~/vm-driver-dev/driver_share/driver/
cp driver/Makefile           ~/vm-driver-dev/driver_share/driver/
cp test/dma_test.c           ~/vm-driver-dev/driver_share/test/
cp scripts/start-vm.sh       ~/vm-driver-dev/start-vm.sh
chmod +x ~/vm-driver-dev/start-vm.sh
```

---

## Step 6 — Build QEMU with the pcie-dma device

```bash
cd ~/vm-driver-dev
git clone https://gitlab.com/qemu-project/qemu.git \
  --depth=1 --branch v8.2.0 qemu-src
cd qemu-src

# Inject the device model
cp ~/vm-driver-dev/driver_share/qemu-dma-dev/pcie_dma.c hw/misc/
# OR from this repo:
# cp /path/to/pcie-dma-driver/device/pcie_dma.c hw/misc/

echo "system_ss.add(files('pcie_dma.c'))" >> hw/misc/meson.build

# Build (use system Python if conda is active)
PYTHON=/usr/bin/python3 ./configure \
  --target-list=aarch64-softmmu \
  --enable-kvm \
  --prefix=$HOME/vm-driver-dev/qemu-build

make -j$(nproc)
make install

# Verify device is registered
~/vm-driver-dev/qemu-build/bin/qemu-system-aarch64 \
  -device ? 2>&1 | grep pcie-dma
# Expected: name "pcie-dma", bus PCI, desc "Simple PCIe DMA Engine"
```

---

## Step 7 — Start the VM

```bash
~/vm-driver-dev/start-vm.sh
```

Login: `dev` / `dev123` — first boot takes ~90 seconds.

> **QEMU monitor shortcuts:**
> `Ctrl-A C` — toggle monitor | `Ctrl-A X` — kill VM | `quit` — graceful shutdown

---

## Step 8 — Inside the VM

```bash
# Mount the shared driver directory
sudo mkdir -p /mnt/drvshare
sudo mount -t 9p -o trans=virtio,version=9p2000.L drvshare /mnt/drvshare

# Persist across reboots
echo "drvshare /mnt/drvshare 9p trans=virtio,version=9p2000.L,nofail 0 0" \
  | sudo tee -a /etc/fstab

# Install build tools
sudo apt update && sudo apt install -y \
  linux-headers-$(uname -r) build-essential kmod

# Build and load the driver
cd /mnt/drvshare/driver
make
sudo insmod pcie_dma_driver.ko
sudo dmesg | tail -3
# Expected: pcie_dma 0000:00:03.0: ready - /dev/pcie_dma0 | sysfs: dma_stats/ dma_info/
```

---

## Step 9 — Run the test

```bash
gcc -O2 -o /tmp/dma_test /mnt/drvshare/test/dma_test.c
sudo /tmp/dma_test
# Expected: All tests PASSED, ~900+ MB/s
```

> `/tmp` is cleared on reboot — rebuild `dma_test` after each reboot.

---

## Step 10 — Read sysfs attributes

### Transfer statistics

```bash
SYSFS=/sys/bus/pci/devices/0000:00:03.0/dma_stats
for f in bytes_written bytes_read irq_count avg_latency_us last_latency_us; do
  printf "%-20s: %s\n" $f $(cat $SYSFS/$f)
done

# Reset counters
echo 1 | sudo tee $SYSFS/reset_stats
```

### Device info

```bash
for f in buf_size buf_phys_addr bar0_start bar0_size vendor_id device_id; do
  printf "%-15s: %s" "$f" \
    "$(cat /sys/bus/pci/devices/0000:00:03.0/dma_info/$f)"
done
```

---

## Development loop

```bash
# On HOST — edit driver source
nano ~/vm-driver-dev/driver_share/driver/pcie_dma_driver.c

# In VM — rebuild and reload
cd /mnt/drvshare/driver
make
sudo rmmod pcie_dma_driver 2>/dev/null || true
sudo insmod pcie_dma_driver.ko
sudo dmesg | tail -5
sudo /tmp/dma_test
```

---

## Benchmark results

| Unit | Hostname | L4T | Throughput |
|------|----------|-----|------------|
| 1 | mobileAi101 | R36.4.7 | ~960 MB/s |
| 2 | mobileAI2 | R36.4.7 | ~1309 MB/s |
| 3 | ubuntu134 | R36.4.0 | ~1692 MB/s |

Benchmark results vary across units due to differing background workloads. Future tests will be conducted after terminating all non-essential applications on each unit to ensure more consistent measurements
---

## Known Issues

| Symptom | Fix |
|---------|-----|
| Emergency mode on boot | `sed -i '/drvshare/d' /etc/fstab` → `exit` → re-add with `nofail` |
| QEMU image lock error | `pkill -f qemu-system-aarch64` then retry |
| `msi_init` compile error | Add `#include "hw/pci/msi.h"` to `pcie_dma.c` |
| QEMU crash on IRQ | Replace `pci_irq_assert` with `msi_notify(&s->parent_obj, 0)` |
| Kernel panic / alignment fault | Register map mismatch — verify `REG_DMA_DST=0x08`, `REG_DMA_IRQ_MASK=0x18`, `REG_DMA_IRQ_ACK=0x1C` |
| `distlib` error during QEMU configure | Use `PYTHON=/usr/bin/python3 ./configure ...` |
| Single-quoted includes in pcie_dma.c | `sed -i "s/#include '\(.*\)'/#include \"\1\"/" hw/misc/pcie_dma.c` |

---

## Roadmap / Future Work

PCIe x16 endpoint upgrade -
The device model currently declares itself as a conventional PCI device. Planned upgrade to a proper PCIe endpoint with x16 link width by changing the interface declaration to INTERFACE_PCIE_DEVICE, adding a PCIe capability structure via pcie_endpoint_cap_init(), and setting x16 link width via pcie_cap_lnkcap_set() in pcie_dma.c. The kernel driver requires no changes for this upgrade.

## License

GPL-2.0 — kernel modules must be GPL-compatible.
