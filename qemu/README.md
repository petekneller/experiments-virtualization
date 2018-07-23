# Prereqs

* `sudo apt-get install qemu`

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

  `qemu-img create -f qcow2 debian.img 8G`

  `qemu-system-x86_64 -hda debian.img -cdrom debian-live-9.5.0-amd64-xfce.iso -boot d -m 512 -enable-kvm` (need to specify the boot device so it doesn't try hda)

  Once the systems installed, use the disk image as copy-on-write backing store for each new VM disk

  `mv debian.img debian-base.img`

  `qemu-img create -f qcow2 -o backing_file=debian-base.img debian1.img` (`-b` defines the backing store, and there's no need to define the size as its taken from the backing)

  `qemu-system-x86_64 -hda debian1.img -m 512 -enable-kvm`

# Networking

* A default user-space network backend and e1000 NIC gets created if no networking options are set. These are _almost_ the equivalent of doing:

  `-netdev user,id=xyz -device e1000,netdev=xyz` ie. the 'user' backend is created and given an ID of `xyz`, an e1000 NIC is added to the VM and attached to the backend/'netdev' `xyz`

  Actually it looks like the above isn't quite the equivalent of the default, because if you run `info network` in the qemu monitor when default networking is being used then you see a `hub0` onto which are 2 interfaces - the e1000 NIC of the VM and a `user.0` which is presumably the user-mode backend on the host. This suggests that the default users an emulated hub and so is more equivalent to:

  `-netdev user,id=user1 -netdev hubport,id=hp1,hubid=0,netdev=user1 -netdev hubport,id=hp2,hubid=0 -device e1000,netdev=hp2` ie. create a user-mode backend called `user1` and connect that to the emulated hub `0` on port `hp1`, give the VM an e1000 NIC and connect it to the emulated hub `0` on port `hp2`

  Except that my qemu doesn't seem to support the `netdev=` option to `-netdev hubport` so it doesn't work correctly. Using legacy options `-net` it is:

  `-net user -net nic,model=e1000` (`-net` adds devices to the default emulated hub `0`)

  *NB* that if you do this for multiple VMs their user-space stacks are entirely separate. Another backend, eg. `tap` or `socket` is necessary for inter-VM comms

* Adding `-nic none` _should_ remove the network adapter from the guest and disable the default network backend (NAT'd) but my qemu seems too old (2.5) to support the `-nic` option. If you don't specify a backend then the user-space one gets added so in lieu of having `-nic none` I think the only way to disable networking is to used a noddy backend (like hubport) that isn't connected to anything. eg.

  `-netdev hubport,id=abc,hubid=0` - the use of `netdev hubport` overrides the default backend and without the default backend there is no default NIC added.

## Tap devices

* To not use the user-space networking and instead set it up explicitly, use the `tap` backend. *NB* this requires either running as root/sudo or by adding yourself to the `kvm` group.

  `-netdev tap,id=nd1,ifname=tap1,script=no,downscript=no`

  This will initialise the tap backend, creating a `tap1` device on the host. It won't add a NIC to the VM so obviously at this point there's no connectivity.

  Assuming `ip link` shows `tap1` as 'down' (which it should) and `ip addr` shows the device having no IP (which it shouldn't) then in order to get the device ready (from the host's POV):
  * `sudo ip link set tap1 up`
  * `sudo ip addr add 10.1.1.1/24 dev tap1`

  At this point, sending anything to `10.1.1.1` will be the equivalent of sending it to localhost. But sending anything to `10.1.1.2` or anything else in the `10.1.1.0/24` network will result in traffic being sent over tap1. Can see why this is the case by running `ip route` and seeing the route out through tap1. The software network is now live:

  Eg. Run from the host:
  * `sudo tcpdump -i tap1` and then from another terminal:
  * `ping 10.1.1.2`
  * `arping -i tap1 10.1.1.2` (Thomas Habet's arping)
  * `arping -i tap1 52:54:00:12:34:56` (where `52:54:00:12:34:56` is the default MAC addr given to the VM)

  Using each of these `tcpdump` should show some packets travelling over the tap inteface. Even without the (ar)pings I found traffic flowing over the interface (mDNS and other). There is no NIC attached to the VM yet, so the VM won't in any way respond, but this serves to prove that the interface is live.

  Now add a NIC to the VM to see it begin to respond over that interface:

  `-netdev tap,id=nd1,ifname=tap1,script=no,downscript=no -device e1000,netdev=nd1`

  Now when we `tcpdump` the tap interface we immediately see the VM trying to solicit a DHCP lease. If we give it an address:

  `sudo ip addr add 10.1.1.2/24 dev ens3` (on the guest, where `ens3` was the ethernet adapter on my guest)

  And now (ar)ping it we'll start to see the VM respond normally.

  At this point we've basically got a second network (a sofware-defined one) at 10.1.1.1/24 which is the equivalent of passing an ethernet crossover cable directly between the host and guest.

  If the host had IP forwarding turned on and no filtering in the way then packets from the guest that were destined for the outside world would be passed by the host:
  * on the host check 
    * `cat /proc/sys/net/ipv4/ip_forward` is 1 (or set it so)
    * `iptables --list -t filter` 'ACCEPT's packets on the FORWARD chain from the relevant interfaces
    * on the guest add the default route if none is available:
    * `ip route add default via 10.1.1.1 dev ens3`

  Now pings from the guest to, say '216.58.206.68' (www.google.com) should be passed via 10.1.1.1 (as the default gateway) and then via whatever the public adapter on the host is. They won't be responded to as the guest has an internal, unroutable, IP address. But if we do a `tcpdump -i tap1` we see the packets sent by the guest and a `tcpdump -i <outbound_iface>` then we'll see the same packets echo'd to the public network. And if we drop the address from the public interface with `ip addr del <xyz/nn> dev <outbound_iface>` while the guest is still pinging then we'll start to see the pings return 'network unreachable'.

  Set `/proc/sys/net/ipv4/ip_forward` to 0 while the ping is running and we'll now see the tcpdump on the external interface no longer showing the ping requests.

## Linux bridges

  I suppose you _could_ at this point just create tap interfaces and forward packets across the host machine, but there are a couple downsides:
* the kernel IP filtering/forwarding happens at the IP layer, so any guests would have to be given IP leases and DNS servers by the host (can't pass DHCP requests across the kernel to the outside network)
* whatever IP's you assign to the guests need to be routable from the outside world (ie. not on a private network) and compliant with any DHCP assignment from the external network

  Unless you're running NAT in which case this is probably exactly what you want.

  However, if you're trying to create publicly-visible, routable guests then a software bridge of some kind is likely necessary. On the other hand you can't NAT across a bridge, so bridges are _only_ applicable where the guests are to be visible.

 If we want to use (linux) bridges then this is how we do so. The setup of the guest doesn't differ at all, as we're only going to connect the tap interfaces to the outside world in a different fashion.

* create the bridge and connect the host so it has external connectivity:

  * `brctl addbr br0`
  * `ip addr del <addr> dev <external_iface>` (external interface must not have an address before being connected to the bridge - it will get one via the bridge)
  * `brct addif br0 <external_iface>`

  At this point the external interface has been added to the bridge. The bridge acts just like an old fashion switch - it just forwards ethernet packets from one 'port' to the others, keeping a port address table so it doesn't have to broadcast on all ports all the time. So any packets arriving on the external NIC (regardless of MAC) will be forwarded over the bridge and any packets leaving interfaces attached to bridge ports will be forwarded out (assuming their external bound) the port bound to the NIC and hence the NIC itself.

  `ip link` will now show that the bridge has the same MAC address as the external NIC. This is how/why the NIC has its address dropped before adding to the bridge:
  * `ip link set br0 up` and possibly...
  * `dhclient br0` (or equivalent, if the network manager doesn't automatically assign it)
  * `ip addr set default via <external_addr> dev <external_iface>`

  From the point of view of the host, the external NIC interface is no longer used within the normal network stack (it has no address). Instead it is attached to the bridge and the bridge provides a new software interface (with same name as the bridge) that masquerades as the same ethernet device as the real NIC, onto which we now assign/request an IP address.

  And finally, to give the guest some connectivity:

  * if the `tap1` interface still has an address associated, remove it with `ip addr del 10.1.1.1/24 dev tap1` otherwise packets will just be forwarded both on the bridge and by the kernel IP forwarding mechanism
  * `brctl addif br0 tap1`
  * inside the guest remove the static IP and let DHCP give it a new one (assuming the outside network provides DHCP):
  * `ip link set <iface> down` and `ip link set <iface> up`

  Now the guest should have an IP assigned by the public network and pings from the guest while tcpdump'ing from the host should show requests from the new IP and successful replys from the outside world.
