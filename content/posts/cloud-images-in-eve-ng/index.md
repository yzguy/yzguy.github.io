+++
title = 'Cloud Images in EVE-NG'
date = 2023-09-24T23:30:15-04:00
+++

EVE-NG has [documentation](https://www.eve-ng.net/index.php/documentation/howtos/howto-create-own-linux-host-image/) on how set up Linux nodes, but provides prepared images.
This is helpful, but the latest Ubuntu image is 20.04.2 and we don't know the origins of these images.

Fortunately, it's easy to get Ubuntu and Debian cloud images working in EVE-NG
### Create Custom Template

We will need to create a custom template as we will want some options that aren't specified in the `Linux` one

```
# linux-cloud.yml
---
type: qemu
description: Linux Cloud Image
name: Linux Cloud Image
cpulimit: 1
icon: Server.png
cpu: 1
ram: 1024
ethernet: 1
console: telnet
shutdown: 1
qemu_arch: x86_64
qemu_version: 2.12.0
qemu_nic: virtio-net-pci
qemu_options: -machine type=pc,accel=kvm -nographic -usbdevice tablet -boot order=cd -cpu host -serial mon:stdio
...
```

After saving this, we can copy it to the AMD & Intel directories where templates are kept

```
cp linux-cloud.yml /opt/unetlab/html/templates/amd
cp linux-cloud.yml /opt/unetlab/html/templates/intel
```

Finally, we just need to register the custom template so EVE-NG knows it exists

```
# /opt/unetlab/html/includes/custom_templates.yml
---
custom_templates:
  - name: linux-cloud
    listname: Linux Cloud
...
```

### Create Image Folder(s)

You will need to create a folder on your EVE-NG machine for each cloud image version you want to use.

{{< admonition type=note >}}
Image folders must be prefixed with `linux-cloud-` for EVE-NG to pick them up under the template created in previous step, the text after the hyphen can be whatever you want
{{< /admonition >}}

```
mkdir /opt/unetlab/addons/qemu/linux-cloud-ubuntu-24.04
mkdir /opt/unetlab/addons/qemu/linux-cloud-debian-12
```

### Download Image(s)

Both Ubuntu and Debian provide cloud images for download, the location for these can be found below

* [Ubuntu Cloud Images](https://cloud-images.ubuntu.com/)
    * eg. `noble-server-cloudimg-amd64.img`
* [Debian Cloud Images](https://cloud.debian.org/images/cloud/)
    * eg. `debian-12-genericcloud-amd64.qcow2`

We will download these to our EVE-NG server into the image folder created above

**Ubuntu:**

```
cd /opt/unetlab/addons/qemu/linux-cloud-ubuntu-24.04
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img \
    -O virtioa.qcow2
```

**Debian:**

```
cd /opt/unetlab/addons/qemu/linux-cloud-debian-12
wget https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-genericcloud-amd64.qcow2 \
    -O virtioa.qcow2
```

#### Optional: Increase Disk Size

Each cloud image comes with ~2GB of space, this will fill very quickly.

```
-> qemu-img info virtioa.qcow2
image: virtioa.qcow2
file format: qcow2
virtual size: 2 GiB (2147483648 bytes)
disk size: 267 MiB
cluster_size: 65536
Format specific information:
    compat: 1.1
    lazy refcounts: false
    refcount bits: 16
    corrupt: false
```

We can increase the image size so that more space will be available

```
-> qemu-img resize virtioa.qcow2 20G
Image resized.
```

We can verify the new size

```
-> qemu-img info virtioa.qcow2
image: virtioa.qcow2
file format: qcow2
virtual size: 20 GiB (21474836480 bytes)
disk size: 267 MiB
cluster_size: 65536
Format specific information:
    compat: 1.1
    lazy refcounts: false
    refcount bits: 16
    corrupt: false
```

This will not expand the partition or filesystem, it will just simply increase the size of the disk. Cloud-init will take care of expanding to take up the remaining free space when it boots

#### Optional: Install Packages

It's super helpful to have VMs start off with all the useful packages for labbing, eg `mtr`, `netcat-openbsd`, etc.

With the help of `virt-customize` we can do this to our image so that any VM started from our image will already have these installed

```
-> virt-customize -a virtioa.qcow2 --install mtr,netcat-openbsd,inetutils-traceroute,iperf3,tcpdump,dnsutils,nmap,neovim,curl,wget
[   0.0] Examining the guest ...
[   4.6] Setting a random seed
virt-customize: warning: random seed could not be set for this type of
guest
[   4.7] Installing packages: mtr netcat-openbsd inetutils-traceroute iperf3 tcpdump dnsutils nmap neovim curl wget
[  11.6] Finishing off
```

### Create Cloud-Init ISO

Cloud-Init allows you to pass configuration to the operating system that will be executed on first boot.

In EVE-NG we need to create an `iSO` file that contains our cloud-init data, this will then be mounted by the VM and then executed when the node first boots

We will need to create two files: `meta-data` and `user-data`

##### meta-data

```
instance-id: node
```

##### user-data

```
#cloud-config
password: lab
chpasswd: { expire: False }
ssh_pwauth: True
```

Refer to Cloud-Init's [Module Reference](https://cloudinit.readthedocs.io/en/latest/reference/modules.html) and 
[Examples](https://cloudinit.readthedocs.io/en/latest/reference/examples.html) for more configuration options

#### Generate iSO

After both files are created, we can create the `iSO` file that will store them, and will be mounted by the VM

```
-> genisoimage -output cdrom.iso -volid cidata -joliet -rock user-data meta-data
I: -input-charset not specified, using utf-8 (detected in locale settings)
Total translation table size: 0
Total rockridge attributes bytes: 331
Total directory bytes: 0
Path table size(bytes): 10
Max brk space used 0
183 extents written (0 MB)
```

After the `cdrom.iso` is created, we can copy it to the image location

```
cp cdrom.iso /opt/unetlab/addons/qemu/linux-cloud-ubuntu-20.04.3
cp cdrom.iso /opt/unetlab/addons/qemu/linux-cloud-debian12
```

### Fix Permissions

As with all EVE-NG images, we need to run `fixpermissions`

```
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```

### Deploy Node

{{< admonition type=warning >}}
Each node should be connected to the internet in some way before starting them, the easiest being through Cloud0. Without this, they take a long time to boot because Cloud-init is unable to complete
{{< /admonition >}}

Return to your EVE-NG Lab and launch a node, you will find these cloud images under `Linux Cloud`, then in the `Image` dropdown you can select the one you want.

Once that's added to the toplogy, you can start it. Wait for it to come to the login prompt.

Each image will have a default username, so you'll need to use that along with the password you set in `user-data`, eg `ubuntu / lab`.

* **Ubuntu**: `ubuntu`
* **Debian**: `debian`
