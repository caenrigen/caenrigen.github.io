# StarTech AV53C1 USB Bluetooth dongle support on linux

This dongle uses a Realtek 8761 chip inside.
For bluetooth to work reliably the linux kernel needs to know that this devices must be treated according to the Realtek chips quirks. A [patch exists][patch_source] but it will take a long time to make it into the Debian backported kernels.
As of February 2026, the latest backported Debian Linux kernel (based on linux kernel `6.18.5`) does not ship with the necessary patch. Building a custom patched kernel from source should not be necessary at some point in the future.

The info below is a reference for debugging the `btusb` driver issues and patching the linux kernel to support new Product IDs. Everything was tested on Debian 13 "trixie" on a x86_64/amd64 machine and an arm64 VM (Apple Silicon).

Many procedures below require to disable Secure Boot (UEFI BIOS settings).

## Verify if the current kernel supports our Product ID

Install latest backported kernel:

```bash
# For x86_64:
sudo apt install -t trixie-backports linux-image-amd64 linux-headers-amd64
# For arm64 (e.g., QEMU VM on Apple Silicon):
# sudo apt install -t trixie-backports linux-image-arm64 linux-headers-arm64
```

Install the "firmware" files for the Realtek chips:

```bash
sudo apt install -t trixie-backports firmware-realtek
```

Plug the dongle into the computer (unplug first if it was already plugged in).

```bash
sudo dmesg -T -w # ctrl-c to exit
```

If you see no `RTL: loading` firmware file being loaded then the kernel is missing the patch. Below is a detailed confirmation of this.

```bash
# Check the dongle Vendor and Product ID
lsusb -v -t
# /:  Bus 003.Port 001: Dev 001, Class=root_hub, Driver=xhci_hcd/16p, 480M
#     ID 1d6b:0002 Linux Foundation 2.0 root hub
#     |__ Port 003: Dev 013, If 0, Class=Wireless, Driver=btusb, 12M
#         ID 2c0a:8761
#     |__ Port 003: Dev 013, If 1, Class=Wireless, Driver=btusb, 12M
#         ID 2c0a:8761

# Check which driver is being used for the USB device
BUS=3 PORT=3; for d in /sys/bus/usb/devices/${BUS}-${PORT}:*/driver; do [ -e "$d" ] && echo "$d -> $(readlink -f "$d")"; done

# These are part of `firmware-realtek` package.
# Check to which package these files belong too.
# `bu` suffix is for USB-based devices
dpkg -S /lib/firmware/rtl_bt/rtl8761bu*

# Check if btusb knows about our device
sudo modinfo -F alias btusb | grep '8761'
# Check if firmware might be built in (unlikely)
lsinitramfs /boot/initrd.img-$(uname -r) | grep '8761'
```

### Check kernel source code

```bash
# `-t trixie-backports` because we tried out the latest backported debian kernel, we need to look at the corresponding source
sudo apt install -t trixie-backports linux-source
cd /usr/src
# Modify this to unpack it elsewhere and not require sudo for inspecting the source files
tar -xf linux-source-*.tar.*
cd linux-source-*/

# Check if this kernel version was compiled with support for our Product ID,
# no matches means the patch is not available.
grep -RIn --line-number '0x2c0a' drivers/bluetooth/btusb.c
```

## Build a patched Debian backported Linux kernel

```bash
# Install the latest backported kernel, headers and firmware if not already installed
sudo apt install -t trixie-backports linux-image-amd64 linux-headers-amd64
sudo apt install build-essential devscripts fakeroot quilt
sudo apt build-dep -t trixie-backports linux

# ! Inside a VM don't do this inside a shared (with the host OS) folder, does not work
# ! unless you take care of user/group ids which is a bit more elaborate.
mkdir -p ~/repos/kernel && cd ~/repos/kernel
# Get the exact version of the kernel we want to patch
apt-cache showsrc linux | grep '^Version:'
apt-get source -t trixie-backports linux=6.18.5-1~bpo13+1
# ls -lAh
# total 152M
# drwxrwxr-x 28 ilab ilab 4.0K Feb 17 20:08 linux-6.18.5/
# -rw-r--r--  1 ilab ilab 1.4M Feb  5 14:53 linux_6.18.5-1~bpo13+1.debian.tar.xz
# -rw-r--r--  1 ilab ilab 136K Feb  5 14:53 linux_6.18.5-1~bpo13+1.dsc
# -rw-r--r--  1 ilab ilab 151M Jan 16 06:21 linux_6.18.5.orig.tar.xz

cd linux-6.18.5
dpkg-parsechangelog -SVersion # E.g., 6.18.5-1~bpo13+1
quilt import -P btusb-rtl8761bu-device-id.patch ~/repos/caenrigen.github.io/notes/linux/rtl8761bu.patch
quilt push
dch --local +rtl8761bu "Add support for StarTech AV53C1 USB Bluetooth dongle"
dpkg-parsechangelog -SVersion # 6.18.5-1~bpo13+1+rtl8761bu1

# Do not build with debug symbols, makes it faster and smaller
export DEB_BUILD_PROFILES='pkg.linux.nokerneldbg pkg.linux.nokerneldbginfo'
export DEB_BUILD_OPTIONS="parallel=$(nproc) nocheck"

# The following command takes a while on a consumer laptop (e.g., 1.5h on a 7-core 8GB RAM VM).
# By default it builds multiple flavours (amd64, cloud-amd64, rt-amd64, test) as defined in
# debian/config/amd64/defines.toml. We only need the normal amd64 flavour, you can remove/comment
# the cloud-amd64, rt-amd64, and test `[[flavour]]` blocks in that file to speed up the compilation.

# -b is for binary packages
# -us and -uc is for building unsigned
dpkg-buildpackage -b -uc -us # will show an error, expected
dpkg-buildpackage -b -uc -us # needs to be run twice due to debian's CI protection

# .deb files are in parent directory
cd ..

# Simulate install
sudo apt -s install ./linux-base-6.18+unreleased-amd64_6.18.5-1~bpo13+1+rtl8761bu1_amd64.deb ./linux-image-6.18+unreleased-amd64-unsigned_6.18.5-1~bpo13+1+rtl8761bu1_amd64.deb ./linux-headers-6.18+unreleased-amd64_6.18.5-1~bpo13+1+rtl8761bu1_amd64.deb ./linux-headers-6.18+unreleased-common_6.18.5-1~bpo13+1+rtl8761bu1_all.deb ./linux-kbuild-6.18+unreleased_6.18.5-1~bpo13+1+rtl8761bu1_amd64.deb
# Install if everything looks good
sudo apt install ./linux-base-6.18+unreleased-amd64_6.18.5-1~bpo13+1+rtl8761bu1_amd64.deb ./linux-image-6.18+unreleased-amd64-unsigned_6.18.5-1~bpo13+1+rtl8761bu1_amd64.deb ./linux-headers-6.18+unreleased-amd64_6.18.5-1~bpo13+1+rtl8761bu1_amd64.deb ./linux-headers-6.18+unreleased-common_6.18.5-1~bpo13+1+rtl8761bu1_all.deb ./linux-kbuild-6.18+unreleased_6.18.5-1~bpo13+1+rtl8761bu1_amd64.deb

# Install again, just in case, because it hooks into the initramfs
sudo apt install -t trixie-backports firmware-realtek
```

Reboot, in the GRUB menu select to boot using this newly installed kernel, plug in the dongle and test it. Bluetooth should work reliably.

### Make the custom kernel the default GRUB boot option

Identify the exact name of the boot option via:

```bash
sudo cat /boot/grub/grub.cfg | grep -i "menuentry "
```

which might look like `"Debian GNU/Linux, with Linux 6.18+unreleased-amd64"`.

Edit these variables in `/etc/default/grub` (requires sudo):

```bash
GRUB_DEFAULT=saved
# to not have to specify the nested advanced menu
GRUB_DISABLE_SUBMENU=true
```

Save the desired default:

```bash
sudo grub-set-default "Debian GNU/Linux, with Linux 6.18+unreleased-amd64"
sudo update-grub # apply changes
sudo grub-editenv list # confirm boot option was saved
```

Reboot to confirm.

## Monkey-patch kernel module for quick testing

This section is for reference only.

**⚠️ WARNING:** This procedure might break the kernel! Always have a backup stable unmodified kernel installed before rebooting (so that it can be selected in the GRUB menu before linux boots up).

This was used as a quick and "dirty" test that confirmed the patch worked.

```bash
sudo apt install firmware-realtek
sudo apt install -t trixie-backports linux-source
cp linux-source-6.18.tar.xz ~/repos/
cd ~/repos/
tar -xf linux-source-*.tar.*
cd ~/repos/linux-source-6.18

# Patch btusb.c manually

make -C /lib/modules/6.18.5+deb13-amd64/build/ M=$PWD/drivers/bluetooth modules

sudo systemctl stop bluetooth

btusb_path="$(sudo modinfo -n btusb)"
sudo cp -a "$btusb_path" "${btusb_path}.bak"
target_dir="$(dirname "$btusb_path")"
sudo install -m 0644 drivers/bluetooth/btusb.ko "$target_dir/btusb.ko"
# Only specific compression is supported inside the kernel
sudo xz -z -f --check=crc32 btusb.ko

sudo depmod -a
sudo update-initramfs -u -k "$(uname -r)"
sudo modprobe -r btusb
sudo modprobe btusb
sudo systemctl start bluetooth
```

Test the dongle and Bluetooth connection to see if the problem is solved. After that revert the changes. E.g.:

```bash
sudo systemctl stop bluetooth

sudo cp -a "${btusb_path}.bak" "$btusb_path"

sudo depmod -a
sudo update-initramfs -u -k "$(uname -r)"
sudo modprobe -r btusb
sudo modprobe btusb
sudo systemctl start bluetooth
```

[patch_source]: https://www.spinics.net/lists/linux-bluetooth/msg125730.html
