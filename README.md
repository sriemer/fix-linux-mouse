![](https://raw.githubusercontent.com/sriemer/fix-linux-mouse/master/mouse-hammer.jpg)

# fix-linux-mouse howto

Everything in this howto relates to openSUSE Leap 42.2 but is mostly applicable
to other Linux distros as well.

## USB mice on Linux

Many USB mice support using them as PS/2 mouse as well (e.g. with an adapter).
They also understand the PS/2 protocol. This is important for using them on
the text consoles/virtual terminals (VT) as well. Currently there is support
for the USB protocol only on the X Window System on Linux.

### Linux kernel driver usbhid

All USB mice use the `usbhid` driver but an additional user-space driver is
required. usbhid devices usually use the USB interrupt transfer. So the default
behavior of the driver is to wait for interrupts. But this can cause buffers in
some devices to overflow. So the usbhid quirk `HID_QUIRK_ALWAYS_POLL` is often
required for USB mice to work properly without a user-space driver running.

The problem is that it can only use the USB vendor ID and product ID to identify
if a quirk is required. So often quirks for mice with the same chips but
different IDs are missing.

The quirks table `hid_blacklist` is located in `drivers/hid/usbhid/hid-quirks.c`
and the usbhid vendor/product IDs are located in `drivers/hid/hid-ids.h`
of the Linux kernel source.

A usbhid quirk can also be set by the kernel boot option `usbhid.quirks`.
E.g. `usbhid.quirks=0x413c:0x301a:0x00000400` sets HID_QUIRK_ALWAYS_POLL
(0x00000400) for the Dell MS116 mouse with idVendor 0x413c and idProduct 0x301a.
The USB IDs can be displayed with `lsusb -vvv`.

If you find out that a quirk is required for your device, then please report
that to the linux-usb(a)vger.kernel.org mailing list to get it fixed in the
upstream kernel.

### On X Window System

The driver package is called `xf86-input-mouse`. This just works on all Linux
distros with all USB mice.

Documentation:

```
man mousedrv
less /usr/share/doc/packages/xf86-input-mouse/README
```

### On text console/virtual terminal

If you want to use your PS/2 capable USB mouse on a VT as well, then you need
GPM (General Purpose Mouse) from package gpm. It provides a "gpm" systemd
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

Note: In some cases a reboot might be required after starting gpm.

The supported mouse protocols/types can be displayed with the following command
executed as root:
```
gpm -m /dev/input/mice -t help | less
```

For imps2 it shows:
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
really annoyed me that it spamed the virtual terminal and the kernel log with
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
Disconnecting it physically everytime you use the VT is no good option. Its
buffer overflows if it is not always polled. This can be fixed by activating the
gpm service and a reboot, or even better by the kernel boot option
`usbhid.quirks=0x413c:0x301a:0x00000400` as this is a usbhid bug.

Of cause I've sent [a patch](http://marc.info/?l=linux-usb&m=149675002229952&w=2)
for this to the linux-usb mailing list which got accepted.
