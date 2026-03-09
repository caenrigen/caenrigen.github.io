# Apple bluetooth keyboards on Linux

## ANSI/ISO swapped keys

Available configs:

```bash
cat /sys/module/hid_apple/parameters/
fnmode iso_layout swap_ctrl_cmd swap_fn_leftctrl swap_opt_cmd
```

Swap "§" key with the "`" key to match the printed layout on some ANSI keyboards:

Add/modify `/etc/modprobe.d/hid_apple.conf` with:

```bash
options hid_apple iso_layout=0
```

## Bluetooth disconnects

Apple's A1314 Bluetooth keyboard suffers from random disconnects (while Linux still shows it as connected).

The following might help to prevent random disconnects for Apple Bluetooth keyboards.

Modify `/etc/bluetooth/input.conf` with:

```ini
UserspaceHID=persist
```

It did not seem to solve my spurious disconnect problem.

### Disable usb auto-suspend for bluetooth usb devices

Add/modify `/etc/modprobe.d/btusb-disable-autosuspend.conf` with:

```bash
options btusb enable_autosuspend=0
```

It did not seem to solve my spurious disconnect problem.
