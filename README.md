![](https://raw.githubusercontent.com/sriemer/fix-linux-mouse/master/mouse-hammer.jpg)

# fix-linux-mouse howto

## Table of Contents

   * [USB mice on Linux](#usb-mice-on-linux)
      * [Linux kernel driver usbhid](#linux-kernel-driver-usbhid)
      * [On Wayland](#on-wayland)
      * [On X Window System](#on-x-window-system)
      * [On text console/virtual terminal](#on-text-consolevirtual-terminal)
      * [USB auto-suspend](#usb-auto-suspend-on-linux)
      * [USB mouse disconnects/reconnects every minute](#usb-mouse-disconnectsreconnects-every-minute-on-linux)
   * [USB mouse in virtual machines](#usb-mouse-in-virtual-machines)
   * [Wireless and Bluetooth mice](#wireless-and-bluetooth-mice)

## Introduction

Everything in this howto relates to openSUSE Leap 15.0 but is mostly applicable
to other Linux distros as well.

## License

Author: **Sebastian Parschauer**

This document has been created with the help of colleagues when I worked
at SUSE. Furthermore, this work is subject to continuous improvements.
**All contributions are welcome.**
Please just open issues on GitHub to start the discussion.

<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">
  <img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" />
</a> This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">
Creative Commons Attribution-ShareAlike 4.0 International License</a>.

## USB mice on Linux

USB optical mice are usually based on a single chip from PixArt Imaging.
An example is the
[PAN3511](https://datasheetspdf.com/pdf-file/776442/PixartImaging/PAN3511/1)
identifying itself with USB IDs `093a:2510`.

Many USB mice support using them as PS/2 mouse as well (e.g. with an adapter).
They also support a PS/2 legacy data report protocol. This is important for
using them on the text consoles/virtual terminals (VT) as well. Currently there
is support for the general USB report protocol only on the display servers on
Linux. See section 5 "USB Interface" in the PAN3511 datasheet for hardware
details.

### Linux kernel driver usbhid

All wired USB mice use the `usbhid` driver but an additional user-space driver
is required. usbhid devices usually use the USB interrupt transfer. So the
default behavior of the driver is to wait for interrupts. But this can cause
buffers in some devices to overflow. So the usbhid quirk fix
`HID_QUIRK_ALWAYS_POLL` is often required for USB mice to work properly without
a user-space driver running.

The problem is that it can only use the USB vendor ID and product ID to identify
if a quirk fix is required. And with more modern chips than the PAN3511, the USB
IDs can be modified. So often quirk fixes for mice with the same chips but
different IDs are missing.

Until `v4.15`, the quirks table `hid_blacklist` is located in
[drivers/hid/usbhid/hid-quirks.c](https://elixir.bootlin.com/linux/v4.15/source/drivers/hid/usbhid/hid-quirks.c#L28)
and the `usbhid` vendor/product IDs are located in
[drivers/hid/hid-ids.h](https://elixir.bootlin.com/linux/v4.15/source/drivers/hid/hid-ids.h#L20)
of the Linux kernel source. Another important kernel source file is
[include/linux/hid.h](https://elixir.bootlin.com/linux/v4.15/source/include/linux/hid.h#L331)
containing the quirk defines. It shows that `HID_QUIRK_ALWAYS_POLL` has the
value `0x00000400`.

A usbhid quirk can also be set by the kernel boot option `usbhid.quirks`.
E.g. `usbhid.quirks=0x413c:0x301a:0x00000400` sets `HID_QUIRK_ALWAYS_POLL`
for the Dell MS116 mouse with idVendor `0x413c` and idProduct `0x301a`.
Usually up to four usbhid quirks can be provided in a comma-separated list.
With `HID_QUIRK_IGNORE` (`0x00000004`) it is also possible to exclude a device.
The USB IDs can be displayed with `lsusb -vvv`.

If you find out that a quirk fix is required for your device, then please open
a GitHub issue here for discussion. Relevant mailing lists are **linux-usb**
and **linux-input** on **vger.kernel.org** to get it fixed in the upstream
kernel.

**Recent changes:**
   * `v4.16`: quirks moved to `hid_quirks` in
[drivers/hid/hid-quirks.c](https://elixir.bootlin.com/linux/v4.16/source/drivers/hid/hid-quirks.c#L29)

### On Wayland

Wayland compositors use `libinput`. On openSUSE Leap 15, the `gnome-shell`
processes are the ones loading it. It is the user-space driver for all input
devices.

Documentation: [libinput](https://wayland.freedesktop.org/libinput/doc/latest/)

**List available input devices:**
```
zypper install libinput-tools
libinput list-devices
```

**Test the mouse events**:
```
zypper install evtest
evtest --grab /dev/input/event23
```

Use the event device which you have found for your mouse. My Dell MS116 mouse
is at `/dev/input/event23` here. Typical events are `BTN_LEFT`, `BTN_RIGHT`,
`BTN_MIDDLE`, `REL_X`, `REL_Y`, and `REL_WHEEL`.

### On X Window System

Modern Linux systems use `libinput` with the `xf86-input-libinput` package for
the X server as well. This way it integrates nicely if the X server is running
on top of Wayland to provide compatibility.

Mouse support on the X Window System usually works fine on all Linux distros
with almost all USB mice.

Documentation:
   * `man 4 libinput`
   * [libinput](https://wayland.freedesktop.org/libinput/doc/latest/)

Alternatives are the packages `xf86-input-mouse` and `xf86-input-evdev`.

Use the following command to check which input driver is running:
```
xpid=$(pidof -s Xorg); if [ -z "$xpid" ]; then xpid=$(pidof -s X); fi; \
sudo cat /proc/$xpid/maps | grep input
```

The package `xf86-input-mouse` is usually only used on non-Linux systems.

Documentation:
   * `man mousedrv`
   * `less /usr/share/doc/packages/xf86-input-mouse/README`

In contrast to that, the package `xf86-input-evdev` provides another generic
Linux input driver.

Documentation:
   * `man evdev`
   * `less /usr/share/doc/packages/xf86-input-evdev/README`

### On text console/virtual terminal

If you want to use your PS/2 capable USB mouse on a VT as well, then you need
**GPM** (General Purpose Mouse) from package `gpm`. It provides a "gpm" systemd
service which is usually disabled by default. Its config is located at
`/etc/sysconfig/mouse`.

Default config:
```
MOUSEDEVICE="/dev/input/mice"
MOUSETYPE="imps2"
GPM_PARAM=""
GPM_REPEAT=""
```

This config is exactly what we need. Just enable and start the gpm service in
your services manager (e.g. with YaST2) and your PS/2 capable USB mouse should
work on your VTs.

**Note:** In some cases a reboot might be required after enabling and starting
gpm.

The supported mouse protocols/types can be displayed with the following command
executed as root:
```
gpm -m /dev/input/mice -t help | less
```

For `imps2` it shows:
```
* imps2    Microsoft Intellimouse (ps2)-autodetect 2/3 buttons,wheel unused
```

### USB auto-suspend on Linux

Mice often don't work well with USB auto-suspend. It is safest to disable it
completely by the kernel boot option `usbcore.autosuspend=-1` to check if the
mouse is affected.

It is also possible to blacklist certain devices. It depends if they are
controlled by the `laptop-mode-tools` or the kernel directly. There are enough
howtos on the web for this.

[openSUSE USB power management](https://en.opensuse.org/Powersaving#USB_power_management)

### USB mouse disconnects/reconnects every minute on Linux

Let's look at a Dell MS116 optical USB mouse. This is a PixArt OEM mouse. It
really annoyed me that it spammed the virtual terminal and the kernel log with
USB disconnect messages every minute without a user-space driver running:
```
[12334.243124] usb 3-14: USB disconnect, device number 12
[12335.748073] usb 3-14: new low-speed USB device number 13 using xhci_hcd
[12335.879685] usb 3-14: New USB device found, idVendor=413c, idProduct=301a
[12335.879689] usb 3-14: New USB device strings: Mfr=1, Product=2, SerialNumber=0
[12335.879691] usb 3-14: Product: Dell MS116 USB Optical Mouse
[12335.879696] usb 3-14: Manufacturer: PixArt
[12335.881821] input: PixArt Dell MS116 USB Optical Mouse as /devices/pci0000:00/0000:00:14.0/usb3/3-14/3-14:1.0/0003:413C:301A.000A/input/input19
[12335.882034] hid-generic 0003:413C:301A.000A: input,hidraw1: USB HID v1.11 Mouse [PixArt Dell MS116 USB Optical Mouse] on usb-0000:00:14.0-14/input0
```
Disconnecting it physically everytime you use the VT is no good option. **Its
buffer overflows if it is not always polled.** This can be fixed by activating
the `gpm` service and a reboot, or even better by the kernel boot option
`usbhid.quirks=0x413c:0x301a:0x00000400` as this is a `usbhid` bug.

The bit mask `0x00000400` activates `HID_QUIRK_ALWAYS_POLL`.

For details see: [Linux kernel driver usbhid](#linux-kernel-driver-usbhid)

-------

**Fixing the Upstream Kernel**

Of cause I've sent [a patch](http://marc.info/?l=linux-usb&m=149675002229952&w=2)
for this to the **linux-usb** mailing list which got accepted. As I've sent it
to the linux **stable** mailing list as well, this is fixed for all Linux
distributions now.

PixArt mice with this HW issue are known from vendors Chicony, Dell, HP,
Microsoft, PixArt, and Primax. Please let me know if your mouse is affected as
**issues often persist for years**.

There is a strong indication that PixArt chips use **Logitech firmware** with
that bug. Before integration, the sensor chips were usually coupled with a
Logitech USB mouse controller IC and PixArt ICs with that bug can be found in
Logitech mice as well. E.g. the Dell MS111-L is a Logitech PixArt mouse
requiring the quirk fix as well (IDs `046d:c077`, see
[#19](https://github.com/sriemer/fix-linux-mouse/issues/19)).

## USB mouse in virtual machines

It is most common in virtual machines that the mouse cursor is not located where
it should be. Windows VMs require **absolute** mouse movement and Linux VMs
require **relative** mouse movement. Make sure that this is properly set e.g.
in `virt-manager`.

With very old Linux distributions which still use GNOME 2 like e.g. SLES11, the
QEMU **EvTouch USB Graphics Tablet** emulation does not work properly. Remove it
and add a **Generic USB Mouse** instead.

## Wireless and Bluetooth mice

Do yourself and people around you a favor and avoid using those due to harmful
pulsed microwave radiation at **2.4 GHz**. The related topic is **electrosmog**.
It can be measured e.g. with the Acousticom 2.

See e.g.: https://swissharmony.com/what-is-electrosmog/<br/>
German: https://www.elektrosmog.com/forschungsergebnisse-zu-biologischen-wirkungen-von-mikrowellen

Especially **Logitech**/PixArt based devices are often found to be
**not optimized** in this regard and can cause **wrist pain** as the most
obvious effect.

With the long range those can also be a major
[security issue](https://www.theverge.com/2019/7/14/20692471/logitech-mousejack-wireless-usb-receiver-vulnerable-hack-hijack).
