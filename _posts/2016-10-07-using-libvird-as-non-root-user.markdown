---
layout: post
title:  "Using libvirt as non-root user"
date:   2016-10-07 21:38:53 +0900
categories: jekyll update
---
There are subtle things to remember for using libvirt as non-root user. 

* Check `virsh uri` is `qemu:///system`.
* The directory for qemu-img should be accessible for `libvirt` group.

{% highlight bash %}
#Add my account to libvirt group. Need re-login for it becomes effective.
sudo usermod -a -G libvirt jihuni
#Make directory for libvirt group.
sudo mkdir /opt/libvirt
sudo chgrp -R libvirt /opt/devops/
{% endhighlight %}


Let's test it.
{% highlight bash %}
#Set URI virsh uses.
export LIBVIRT_DEFAULT_URI=qemu:///system
#Check it.
virsh uri

#Create a new virtual switch.
virsh net-define switch.xml
virsh net-autostart vntest
virsh net-start vntest
#Check it
virsh net-list

#Create a new image file
cd /opt/libvirt
qemu-img create -f qcow2 nixos.qcow2 5G
#Launch a KVM instance!
virt-install --name nixos --ram 2048 --vcpus 4 \
    --disk path=nixos.qcow2,bus=virtio \
    --os-type linux --os-variant generic \
    --network network=vntest,model=virtio \
    --graphics spice,listen=0.0.0.0 \
    --cdrom nixos-minimal-16.09.680.4e14fd5-x86_64-linux.iso
#Check it
virsh list
#Check a port number for spice and connect to it.
virsh dumpxml nixos
remote-viewer spice://localhost:5900
{% endhighlight %}

The 'switch.xml' file I used:
{% highlight xml %}
<network>
  <name>vntest</name>
  <bridge name="vn1"/>
  <forward/>
  <ip address="192.168.142.1" netmask="255.255.255.0">
    <dhcp>
      <range start="192.168.142.2" end="192.168.142.254"/>
    </dhcp>
  </ip>
</network>
{% endhighlight %}

