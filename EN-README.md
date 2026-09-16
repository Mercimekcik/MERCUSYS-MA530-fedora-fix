# MERCUSYS MA530 BLUETOOTH 5.3 ADAPTER — FEDORA FIX GUIDE

Tested fix for the Mercusys MA530 USB Bluetooth 5.3 dongle not being detected by any device (phone, headset, controller, etc.) on Fedora / other Linux distros.

## THE PROBLEM

The Mercusys MA530 (USB ID: `2c4e:0115`) uses a Realtek RTL8761BUV chipset. On Fedora's kernel (and most other distros as of this writing), this device ID is **missing** from the `btusb` driver's `quirks_table`. As a result:

- The device shows up in `lsusb`, `hci0` is created, and `power on` works fine
- **But** the firmware (`rtl8761bu_fw.bin`) never actually loads
- End result: Bluetooth *looks* like it's working, but it can't discover/scan any devices at all (TX seems to work, RX effectively doesn't)

This is a known gap upstream. Several people submitted patches independently to add this ID (Michal Piernik — Feb 2025, elespink — Aug 2025, Hrvoje Nuic — Apr 2026). Hrvoje Nuic's patch was the one officially merged (upstream commit: `ce21a5cf3d1fd92b84ea9ad2b7c7240aff2162d2`) and was backported to older stable kernel branches (6.18.y, 6.12.y, 6.1.y) in August 2026 via AUTOSEL. However, whether your specific distro's kernel package has picked it up is **not guaranteed** — you need to check.

## STEP 1: CONFIRM THE ISSUE

Check whether you're actually hitting this bug:

```bash
sudo dmesg | grep -i RTL
```

If the output does **not** contain lines like these (only an ethernet `r8169` line, for example), the firmware isn't loading and you should continue with the steps below:

```
RTL: examining hci_ver=...
RTL: loading rtl_bt/rtl8761bu_fw.bin
```

You can also check whether the compiled kernel module recognizes this ID (informational only — not conclusive proof, since `quirks_table` isn't part of `MODULE_DEVICE_TABLE`, so it won't show up in the alias list even if patched):

```bash
modinfo btusb | grep -i 2c4e
```

Check that the firmware files exist on your system (usually already present via the `linux-firmware` package):

```bash
ls -la /usr/lib/firmware/rtl_bt/ | grep 8761bu
```

Check `rfkill` (should say "no" if not blocked):

```bash
rfkill list
```

## STEP 2: TRY A KERNEL UPDATE FIRST (EASIEST PATH)

Your distro may have picked up the fix by now — try this first:

```bash
sudo dnf update kernel
sudo reboot
```

After updating, re-run the `dmesg` command from Step 1. If the RTL lines show up, **the issue is fixed** and you don't need to go further.

## STEP 3: IF STILL MISSING — MANUAL PATCH VIA DKMS (WORKING FIX)

This method was tested and **works** (confirmed on `6.19.10-300.fc44.x86_64`).

**3.1) Install required packages:**

```bash
sudo dnf install -y kernel-devel-$(uname -r) dkms make gcc curl
```

**3.2) Download the source files** (`btusb.c` plus its sibling header files — **all of them are required**, `btusb.c` alone is not enough):

```bash
mkdir -p ~/btusb-mercusys && cd ~/btusb-mercusys
curl -o btusb.c "https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/plain/drivers/bluetooth/btusb.c?h=v6.19.10"
curl -o btintel.h "https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/plain/drivers/bluetooth/btintel.h?h=v6.19.10"
curl -o btrtl.h "https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/plain/drivers/bluetooth/btrtl.h?h=v6.19.10"
curl -o btbcm.h "https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/plain/drivers/bluetooth/btbcm.h?h=v6.19.10"
curl -o btmtk.h "https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/plain/drivers/bluetooth/btmtk.h?h=v6.19.10"
```

> **Note:** Replace `v6.19.10` with your own running kernel version (the major version from `uname -r`). For example, if you're on kernel `6.20.x`, try `v6.20`. Verify the download worked correctly:
>
> ```bash
> head -5 btintel.h
> ```
>
> The output should be real C code starting with `#ifndef`, **not** an HTML/error page. If it's wrong, you need to find the correct tag/version name.

**3.3) Add the Mercusys ID into `btusb.c`:**

```bash
sed -i '/Additional Realtek 8761BUV Bluetooth devices/a\	{ USB_DEVICE(0x2c4e, 0x0115), .driver_info = BTUSB_REALTEK |\n\t\t\t\t\t\t     BTUSB_WIDEBAND_SPEECH },' btusb.c
```

Verify it:

```bash
grep -A2 "0x2c4e, 0x0115" btusb.c
```

**3.4) Create the `Makefile`:**

```bash
cat > Makefile << 'EOF'
obj-m += btusb.o
KDIR := /lib/modules/$(shell uname -r)/build
PWD := $(shell pwd)
default:
	$(MAKE) -C $(KDIR) M=$(PWD) modules
clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
EOF
```

**3.5) Create `dkms.conf`:**

```bash
cat > dkms.conf << 'EOF'
PACKAGE_NAME="btusb-mercusys"
PACKAGE_VERSION="1.0"
BUILT_MODULE_NAME[0]="btusb"
DEST_MODULE_LOCATION[0]="/kernel/drivers/bluetooth"
AUTOINSTALL="yes"
EOF
```

**3.6) Check Secure Boot status:**

```bash
mokutil --sb-state
```

If it says `SecureBoot enabled`, `dkms install` will ask you to set a MOK password, and afterward you'll need to **reboot** and select "Enroll MOK" on the blue screen and enter that password. Skipping this step means the module won't load. If it says `SecureBoot disabled`, don't worry about this at all.

**3.7) Add to DKMS and build:**

```bash
sudo mkdir -p /usr/src/btusb-mercusys-1.0
sudo cp -r ~/btusb-mercusys/* /usr/src/btusb-mercusys-1.0/
sudo dkms add -m btusb-mercusys -v 1.0
sudo dkms build -m btusb-mercusys -v 1.0
```

You should see `Building module(s)... done.`. If you get an error (e.g. `fatal error: btintel.h: No such file or directory`), you forgot to download the header files from step 3.2 — check the log:

```bash
cat /var/lib/dkms/btusb-mercusys/1.0/build/make.log
```

**3.8) Install it:**

```bash
sudo dkms install -m btusb-mercusys -v 1.0
```

**3.9) Unload the old module, load the new one:**

```bash
sudo systemctl stop bluetooth
sudo modprobe -r btusb
sudo modprobe btusb
sudo systemctl start bluetooth
```

**3.10) Verify — the RTL lines should now appear:**

```bash
sudo dmesg | grep -i RTL
```

Expected output:

```
Bluetooth: hci0: RTL: examining hci_ver=0a hci_rev=000b lmp_ver=0a lmp_subver=8761
Bluetooth: hci0: RTL: rom_version status=0 version=1
Bluetooth: hci0: RTL: loading rtl_bt/rtl8761bu_fw.bin
Bluetooth: hci0: RTL: loading rtl_bt/rtl8761bu_config.bin
Bluetooth: hci0: RTL: fw version 0xdfc6d922
```

**3.11) Test Bluetooth scanning:**

```bash
bluetoothctl
scan on
```

(Wait ~30 seconds, then type `devices` — your devices should now show up in the list.)

## PERSISTENCE NOTE

The DKMS registration is **permanent**. You don't need to do anything after a reboot. When the kernel is updated (e.g. via `dnf update`), DKMS will **automatically** rebuild and reinstall the module for the new kernel in the background (via `dkms.service`). If you reinstall/reformat your system, you'll need to redo steps 1–3 from scratch, since the DKMS registration lives on disk and gets wiped.

If this ID is eventually added to your distro's official kernel, there's no conflict — the DKMS-built version does the same thing, so nothing manual is needed either way.

## Hardware info reference

```
Vendor=2c4e ProdID=0115
Product: Mercusys MA530 Adapter
Chipset: Realtek RTL8761BUV
```

## Contributing

If you got this working on a different distro or kernel version, feel free to open a PR or issue with your notes.
