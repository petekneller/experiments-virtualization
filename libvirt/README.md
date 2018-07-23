# Prereqs

* qemu
* `apt-get install libvirt0`

* On my machine the 'default' network wasn't setup from installation

  * Create definition of network:

   ```
   cat > /usr/share/libvirt/networks/default.xml

   <network>
     <name>default</name>
     <bridge name="virbr0"/>
     <forward mode="nat"/>
     <ip address="192.168.122.1" netmask="255.255.255.0">
       <dhcp>
         <range start="192.168.122.2" end="192.168.122.254"/>
       </dhcp>
     </ip>
   </network>

   ```

  * `virsh net-define /usr/share/libvirt/networks/default.xml`
  * `virsh net-start default`
  * and optionally, if you want it started on boot: `virsh net-autostart default`

# Some basic examples

* Launch a LiveCD of Debian (the equivalent of that found in the QEMU readme)

  `virt-install \
              --connect qemu:///system \
              --hvm \
              --name debian-live \
              --memory 512 \
              --disk none \
              --livecd \
              --graphics vnc \
              --cdrom /home/pete/.qemu-vms/debian-live-9.5.0-amd64-xfce.iso`

  In the above:
  * the `disk` parameter is required (since we're trying to install an OS) but can be explicitly disabled with `none`
  * the `graphics vnc` will create a VNC session and connect the guest graphic console to it
  * `hvm` specifies that hardware virtualization is desired
  * `cdrom` (or one of a few other params) is required to specify the install media
  * `livecd` specifies that the install medium (cdrom in this case) is intended to be connected and set as boot permanently

  This will 'define' the domain (guest) and start it. Afterwards the domain is considered to have been previously defined, so if you run the above again you will get an `ERROR Guest name 'debian-live' is already in use`.

  Try:
  * `virsh list --all` to see the stopped guest
  * `virsh dominfo debian-live`
  * `virsh domstats debian-live`
  * `virsh domiflist debian-live`
  * `virsh domblklist debian-live`
  * `virsh net-list`
  * `virsh pool-list`
  * `virsh vol-list .qemu-vms` (where `.qemu-vms` is the pool ID from `pool-list`)
  * `virsh dumpxml debian-live` to see the config file

  To re-run the above guest:

  `virsh start debian-live` and then:

  * `virsh domblkstat debian-live`
  * `virsh domifstat debian-live vnet0` (`vnet0` is the interface name taken from `virsh domiflist`)
  * `virsh suspend`, `virsh resume`
  * `virsh shutdown` to shut it down gracefully or `virsh destroy` to shut it down forcefully

  While the domain is running:
  * `virt-viewer debian-live` to connect a VNC client (assuming using VNC graphics - which this example does)
  * `virsh console` to connect serial client (assuming serial client is available - which is not in this example)

* Install from a Debian LiveCD (equivalent of what I did with qemu)

  `virt-install \
              --connect qemu:///system \
              --hvm \
              --name debian-live \
              --memory 512 \
              --disk size=8 \
              --cdrom /home/pete/.qemu-vms/debian-live-9.5.0-amd64-xfce.iso`

  If we don't actually install the OS and instead just exit the installer, virt-install will exit and reboot (without the installation - cdrom - media connected) under the expectation that the installation succeeded.

  `virsh domblklist` and `virsh dumpxml debian-live` will now show only the hard disk and not the cdrom connected

* Create a domain for an already-created VM

  Using a debian disk image I'd previously created with QEMU:

  `virt-install \
              --connect qemu:///system \
              --hvm \
              --name debian-base \
              --memory 512 \
              --disk /home/pete/.qemu-vms/debian-base.img \
              --graphics vnc \
              --import`

  The `--import` option skips the installation phase (since we already have an installed disk) and just defines the domain and boots into the guest

# How NOT to do NAT'd, bridged guests with virtual ethernet devices

  I wanted to set up a number of guests, on a bridge so they could talk to one another, but also NAT'd to the outside world so they weren't exposed on the public network. I set up a bridge, call it `br0` onto which I added my first guest, which had a tap interface of `tap0`. And now, instead of adding the public interface to the bridge (which would have exposed everything at layer 2) I could have given the bridge its own IP (on its `br0` interface) and set up routing and NAT between the bridge and the public interface. But instead of that (because the bridge having an IP I find a little odd) I decided to create a veth pair, put one end in the bridge and give the other end an IP, using the free end as my routing/NAT interface.

  In order to use a linux bridge with a guest that had previously created using the default libvirt NAT networking I modified the definition of the network configuration in the guest:
  * `virsh edit debian-base`
  * in the network section - `<interface type="network">` - the  type goes to `bridge`
  * and the source gets identified - adding the `source` element so that the result looks like:
    ```
    <interface type='bridge'>
      <mac address='52:54:00:00:00:12'/>
      <source bridge='br0'/>
      <model type='rtl8139'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
    ```

  Except it didn't work. I could ping the guest from the host, and vice versa. But the guest couldn't ping the outside world. `tcpdump`ing the public interface showed that the packets were being forwarded out the public interface, but with the original source IP (no NAT), so the guest was not getting any replies.

  After a hell of a lot of frustration I found that you can add `LOG` targets to iptables to see when rules are hit, and the POSTROUTING chain of the nat table was *not* being hit. I then found you can add `TRACE` targets to iptables and I found that the nat chain was being hit, but as the packets crossed the bridge, and not in the veth -> public crossing. Looking at the output of `conntrack -E` confirmed that the connection was being initialised as the packets traversed the bridge and as a result the nat chain wasn't being hit after that. So with this setup I can only add the masquerade rules if they can be applied while the packet is traversing the bridge. Which I can't.

  The quick solution was to drop the veth pair and just go back to giving the bridge an IP. Which is a bit odd because I'd still expect the packet the traverse the kernel as both traverses the bridge and the bridge -> public. But it seems to work that way. I suspect it would work okay if instead of a linux bridge I used a non-kernel based one, like OVS.

# Open vSwitch

  The situation above - masquerading not working the way I wanted it to because the linux bridge implementation is in-kernel - can be remedied by using Open vSwitch instead of the linux bridges. As such I am going back to the idea of using leaving the bridge free of physical interfaces and instead attached a veth between the bridge and host.

  Assumptions:
  * I have a previously-created guest in which a static IP is assigned in the range 192.168.0.0/24, excluding .1 (the host)
  * OVS is installed: `apt-get install openvswitch-switch`

  Creation (very similar to creating a linux bridge):
  * `ovs-vsctl add-br br0`
  * `ip link add veth0 type veth peer name veth1`
  * `ovs-vsctl add-port br0 veth0`
  * `sudo ip addr add 192.168.0.1/24 dev veth1`
  * `ip link set br0 up`
  * `ip link set veth0 up`
  * `ip link set veth1 up`
  
  The guest needs to have its network configuration changed at this point so that libvirt knows it is using OVS rather than linux bridge:
  * `virsh edit debian-base` and change the network section to:
    ```
    <interface type='bridge'>
      <mac address='52:54:00:00:00:12'/>
      <source bridge='br0'/>
      <virtualport type="openvswitch"/>
      <model type='rtl8139'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
    ```
    
  Now when the guest starts (`virsh start debian-base`) the NIC will be attached to the OVS bridge. This be checked via `ovs-vsctl show` where there will be a new `vnet0` (or similar) port on the bridge.
  
  At this point the host and guest should be able to communicate. Packets from the guest will also be passed to the public interface but won't be responded to without setting up NAT.
