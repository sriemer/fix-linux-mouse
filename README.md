# fix-linux-mouse howto

Everthing in this howto relates to openSUSE Leap 42.2 but is mostly applicable
to other Linux distros as well.

## USB mice

Many USB mice support using them as PS/2 mouse as well (e.g. with an adapter).
This is important to know as only these work on the text consoles/virtual
terminals (VT) as well. Currently there is support for pure USB mice only on
the X Window System on Linux.

### On X Window System

The driver package is called `xf86-input-mouse`. This just works on all Linux
distros.

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

This config is exactly what we need. Just enable the gpm service in your
services manager (e.g. with YaST2) and your PS/2 capable USB mouse will
work on your VTs.

The supported mouse protocols/types can be displayed with the following command
executed as root:
```
gpm -m /dev/input/mice -t help | less
```

For imps2 it shows:
```
* imps2    Microsoft Intellimouse (ps2)-autodetect 2/3 buttons,wheel unused
```

### USB auto-suspend

Mice usually don't work well with USB auto-suspend. It is safest to disable it
completely by the kernel boot option `usbcore.autosuspend=-1`.

It is also possible to blacklist certain devices. It depends if they are
controlled by the laptop-mode-tools or pm-utils. There are enough howtos on the
web for this.

### Dell USB mice

Let's look at a Dell MS116 optical USB mouse. This one is a pure USB mouse and
only works on the X Window System. The annoying bit is that it spams your VT
and the kernel log with USB disconnect messages every minute:
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
Disconnecting it physically everytime you use the VT is no good option. It is
missing a driver. So activate the gpm service and don't use the mouse there.
The mouse won't work but these annoying messages are gone. If this still annoys
you, then rather use a Logitech mouse. ;)

Of cause also USB auto-suspend doesn't work with these mice. The mouse only
wakes up if pressing a button. So disable USB auto-suspend.

### Logitech USB mice

The Logitech USB mice support PS/2 and USB. These usually always work fine.
The only issue with these is usually that they pretend to support USB
auto-suspend but actually don't. So disable it. These work fine on VTs with gpm.
