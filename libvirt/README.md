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
