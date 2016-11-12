---
layout: post
title:  "Using SR-IOV and VT-d with libvirt"
date:   2016-11-12 20:00:00 +0900
categories: jekyll update
---

SR-IOV is for network virtualization. Does it matter to me anyhow? With VT-d, I found it is useful for testing. Say, I want add a test server in a network I am using. Deploy a KVM instance like in [my previous post](https://jihuni.github.io/jekyll/update/2016/10/07/using-libvird-as-non-root-user.html) does not fully solve my problem. By default, KVM instances are running on a dedicated virtual network switch; i.e. they live in a separated network. With SR-IOV and VT-d, however, I can assign "real" NICs to KVM instances. They get IPs from the "real" network and KVM instances behave as if they are just like other server in the network. No more worry about virtual network configuration. Of course, SR-IOV is not for just testing; it gives almost "bare-metal" network performances for VMs.

SR-IOV populates virtual NICs. For host OS, it just looks like usual, PCI NICs. Then, using VT-d, I can assign them to KVM instances. 
Example server before SR-IOV :
```bash
$ lspci | grep Ethernet
03:00.0 Ethernet controller: Intel Corporation I350 Gigabit Network Connection (rev 01)
03:00.1 Ethernet controller: Intel Corporation I350 Gigabit Network Connection (rev 01)
$ lspci -n | grep 03:00
03:00.0 0200: 8086:1521 (rev 01)
03:00.1 0200: 8086:1521 (rev 01)
```
Let's enable SR-IOV. Note that `igb` is for Intel 1GbE. For Intel 10GbE, it is `ixgb`:
```bash
modprobe -r igb
modprobe igb max_vfs=2
lsmod | grep igb
#For persistant setup
echo "options igb max_vfs=2" >>/etc/modprobe.d/igb.conf
```
It populates 2 virtual NIC for each NICs:
```bash
$ lspci | grep Ethernet
03:00.0 Ethernet controller: Intel Corporation I350 Gigabit Network Connection (rev 01)
03:00.1 Ethernet controller: Intel Corporation I350 Gigabit Network Connection (rev 01)
03:10.0 Ethernet controller: Intel Corporation I350 Ethernet Controller Virtual Function (rev 01)
03:10.1 Ethernet controller: Intel Corporation I350 Ethernet Controller Virtual Function (rev 01)
03:10.4 Ethernet controller: Intel Corporation I350 Ethernet Controller Virtual Function (rev 01)
03:10.5 Ethernet controller: Intel Corporation I350 Ethernet Controller Virtual Function (rev 01)
#libvirt should see this, too:
export LIBVIRT_DEFAULT_URI=qemu:///system
virsh nodedev-list | grep 03
pci_0000_03_00_0
pci_0000_03_00_1
pci_0000_03_10_0
pci_0000_03_10_1
pci_0000_03_10_4
pci_0000_03_10_5
```
With VT-d, we can assign the populated NICs. To enable VT-d, one need to change BIOS setting. After it's done, let libvirt can manage populated virtual NICs. Instead of specifying each VFNIC, it is simpler to define a pool for them and use it:
```bash
virsh net-define passthrough.xml
virsh net-autostart passthrough
virsh net-start passthrough.
```
Example `passthrough.xml` :
```xml
<network>
  <name>passthrough</name>
  <forward mode='hostdev' managed='yes'>
    <pf dev='enp3s0f0'/>
  </forward>
</network>
```
Note that VT-d passthrough applies for a physical NIC; `enp3s0f0` in this example. Repeat this for each physical NIC you want. 

To use it when deploy KVM instances, add `--network network=passthrough,model=igb` option to use it:
```bash
virt-install --name fedora --ram 4096 --vcpus 4 \
    --disk path=node1.qcow2,bus=virtio \
    --os-type linux --os-variant generic \
    --network network=passthrough,model=igb \
    --graphics spice,listen=0.0.0.0 \
    --cdrom Fedora-LXDE-Live-x86_64-24-1.2.iso
```
For more details, check [libirt wiki](http://wiki.libvirt.org/page/Networking#Assignment_from_a_pool_of_SRIOV_VFs_in_a_libvirt_.3Cnetwork.3E_definition) and [CentOS7 SR-IOV](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Virtualization_Deployment_and_Administration_Guide/sect-SR_IOV-Using_SR_IOV.html) docuementation.


