# Prereqs

`sudo apt-get install qemu`

# Some basic QEMU examples

Taken from (and credits available at) https://wiki.qemu.org/Testing/System_Images. *NB* that qemu on my machine is version 2.5 which seems not to support the absolute latest cli switches

* A FreeDOS image run directly from floppy (booting from A drive)
  `qemu-system-x86_64 -boot a -fda odin1440.img`

* Running a Debian LiveCD
  From a USB device (seems _really_ slow compared to CDROM)
  `qemu-system-x86_64 -usb -usbdevice disk:debian-live-9.5.0-amd64-xfce.iso -m 512` (the default 128 MB seems too low to boot into the desktop)

  From CDROM
  `qemu-system-x86_64 -cdrom debian-live-9.5.0-amd64-xfce.iso -m 512` (could use `-boot d` to boot from CDROM but the BIOS will try this anyway when there is no bootable hdd)

  ... which is I think is the equivalent of specifying the TCG hardware acceleration:
  `qemu-system-x86_64 -cdrom debian-live-9.5.0-amd64-xfce.iso -m 512 -machine type=pc-q35-xenial,accel=tcg`

  Using KVM acceleration:
  `qemu-system-x86_64 -cdrom debian-live-9.5.0-amd64-xfce.iso -m 512 -machine type=pc-q35-xenial,accel=kvm` (need to specify the machine type to access the 'accel' parameter)

  ... which seems to be the equivalent of:
  `qemu-system-x86_64 -cdrom debian-live-9.5.0-amd64-xfce.iso -m 512 -enable-kvm`

  Passing through the host CPU spec:
  `qemu-system-x86_64 -cdrom debian-live-9.5.0-amd64-xfce.iso -m 512 -enable-kvm -cpu host`

  Adding a disk image for installation
  `qemu-img create -f qcow2 debian.img 4G`
  `qemu-system-x86_64 -hda debian.img -cdrom debian-live-9.5.0-amd64-xfce.iso -boot d -m 512` (need to specify the boot device so it doesn't try hda)

# Networking

* The following _should_ work but my qemu is too old to support the `-nic` option
