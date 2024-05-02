+++
title = 'PicOS-V in EVE-NG'
date = 2024-05-01T21:51:17-04:00
+++

PicaOS-V is the free virtual machine provided by [Pica8](https://www.pica8.com/) that allows you to experiment with their NOS.

Having virtual images for various NOS is hugely beneficial for learning.  PicOS is the one I've recently stumbled upon.

A GNS3 qcow2 image is provided as well as directions for loading it into GNS3. However nothing for EVE-NG is provided, but fortunately it's easy to create a custom template and get it loaded in.

### Obtain Image

You will need to obtain the PicOS-V image from [Pica8's website](https://www.pica8.com/picos-v/#try-picos-at-your-own-pace-with-no-commitments)

When presented with your image download options, select the GNS3 one. This will download a `qcow2` image.

At the time of writing, the latest version available is `4.3.2.2`

Transfer this image to your EVE-NG server

### Prepare Image Folder

Navigate to EVE-NGs image folder and create one for your PicOS-V image

```
cd /opt/unetlab/addons/qemu/
mkdir picos-4.3.2.2
cd picos-4.3.2.2
```

Move or copy your PicOS-V image into this folder as `sataa.qcow2`

```
mv ~/picos-4.3.2.2* ./sataa.qcow2
```

### Create Custom Template

```
# picos.yml
---
type: qemu
description: PicOS
name: picos
cpulimit: 1
icon: Router.png
cpu: 1
ram: 2048
ethernet: 5
eth_name:
- eth0
eth_format: swp{1}
console: telnet
qemu_arch: x86_64
qemu_version: 4.1.0
qemu_nic: virtio-net-pci
qemu_options: -machine type=pc,accel=kvm -nographic -rtc base=utc
...
```

After saving this, we can copy it to the AMD & Intel directories where templates are kept

```
cp picos.yml /opt/unetlab/html/templates/amd
cp picos.yml /opt/unetlab/html/templates/intel
```

Finally, we just need to register the custom template so EVE-NG knows it exists

```
# /opt/unetlab/html/includes/custom_templates.yml
---
custom_templates:
  - name: picos
    listname: PicOS
...
```

### Fix Permissions

As with all EVE-NG images, we need to run `fixpermissions`

```
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```

### Deploy Node

Return to your EVE-NG Lab and launch a node, you will find it `PicOS`, then in the `Image` dropdown you can select the version you want.

Once that's added to the toplogy, you can start it. Wait for it to come to the login prompt.

PicOS-V default has default credentials of `admin` / `pica8`. After authenticating it will have you change the default password


### References

You can refer to the [GNS3 Deployment guide](https://www.pica8.com/wp-content/uploads/PicOS-V-in-GNS3-User-Guide.pdf) as well as a general [Configuration Guide](https://pica8-fs.atlassian.net/wiki/spaces/PicOS433sp/overview)

### Tips & Tricks

#### Set Management IP

```
admin@PICOS> configure
admin@PICOS# set system management-ethernet eth0 ip-address IPv4 dhcp
admin@PICOS# commit
admin@PICOS# run show interface management-ethernet
eth0	Hwaddr: 50:00:00:01:00:00	State: UP
	Gateway  : 192.168.111.1
	Inet addr:
	           192.168.111.168/24
```

#### Simulate different hardware model

By default PicOS-V will simulate an **EdgeCore AS5812-54X**, but you can easily swap it to simulate a different model. This will change the interface naming, etc to match

To simulate a **Dell N3248P-ON**

```
admin@PICOS> bash "sudo /etc/reset-product.sh -p N3248P-ON"
Simulation Platform changed from 'as5812_54x' to 'N3248P-ON'.
Please reboot the virtual machine.
```

Additionally you will want to clear the following directories so previous configuration for the older model is not carried forward

```
admin@PICOS> bash "sudo rm /pica/config/*"
admin@PICOS> bash "sudo rm /backup/pica/config/*"
```

Finally, reboot the VM

```
request system reboot
```
