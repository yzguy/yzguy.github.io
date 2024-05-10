+++
title = 'EVE-NG in LXC on Proxmox' 
date = 2024-05-10T20:14:00-04:00
+++

EVE-NG has multiple deployment methods with mostly references to starting from an iSO or OVF. You can also start from a fresh Ubuntu 22.04 install, then run the installation script to install EVE-NG. You can see these instructions in the [Pro](https://www.eve-ng.net/index.php/documentation/professional-cookbook/) or [Community](https://www.eve-ng.net/index.php/documentation/community-cookbook/) Cookbook.

The ability to start from an Ubuntu install and then install EVE-NG on top means we can install it inside an LXC container! However, it's not as simple due to the fact a LXC container is NOT a virtual machine. We will need to tweak the configuration to allow nesting and to allow cgroups in the LXC container.

Additionally, we will be deploying our EVE-NG installation on top of Proxmox VE 8.x so that we can use the same system for other purposes if necessary. These instructions will work for both EVE-NG Pro and EVE-NG Community.

## Prerequisites

* Ubuntu 22.04 LXC Template

### Create LXC Container

1. Click Create CT button Proxmox
2. Generate
    1. Enter a hostname
    2. Uncheck `Unprivileged Container`
    3. Enter/Confirm password
    4. (Optional) Enter an SSH Public Key
3. Template
    1. Select `ubuntu-22.04-standard_22.04-1_amd64.tar.gz` template
4. Disks
    1. Set desired disk size
5. CPU
    1. Set desired cores
6. Memory
    1. Set desired memory
    2. Set desired swap
7. Network
    1. (Optional) Set VLAN Tag
    2. Uncheck Firewall
    3. Set IPv4 to Static or DHCP
    3. Set IPv6 to Static, DHCP, SLAAC
8. DNS
    1. Set DNS domain as desired (or leave to use host settings)
    2. Set DNS servers as desired (or leave to use host settings)
9. Confirm
    1. Do **NOT** check "Start after created"
    2. Click Finish

You will now have an Ubuntu LXC, but there are a few more modifications to do before we can start it and install EVE-NG

1. Select your LXC from the navigation
2. (Optional) Add additional network interfaces (eth1, eth2, etc.)
    * For additional interfaces, leave IPv4 and IPv6 set to Static, but do not enter an IP
3. Under Options > Features
    1. Check Nesting
    2. Check FUSE

This will allow the the LXC to launch additional containers, this is necessary for EVE-NG to function. We will need to do an additional step on the CLI

1. SSH to the Proxmox Node
2. Open the LXC's configuration file
    1. `/etc/pve/lxc/<ID>.conf`, eg `/etc/pve/lxc/111.conf`
3. At the end of the file add the following

    ```
    lxc.cgroup2.devices.allow: a
    lxc.mount.entry: /dev/net dev/net none bind,create=dir
    lxc.apparmor.profile: unconfined
    lxc.cap.drop:
    ```

    * Allows LXC access to all devices
    * Mounts the hosts /dev/net into the LXC
    * Sets the apparmor profile to unconfined (open)
    * Gives LXC all capabilities

    [Reference](https://linuxcontainers.org/lxc/manpages/man5/lxc.container.conf.5.html)

4. Save and quit

### Upgrade Ubuntu

1. Start your LXC
2. From the console or SSH
    1. Run `apt-get update && apt-get upgrade -y`
    2. Install `gnupg2` dependency via `apt-get install gnupg2 -y`
3. Reboot

### Install EVE-NG

Finally, we have all the bits in place to install EVE-NG. We will need to use the installation script as we are installing on top of an existing Ubuntu installation.

* EVE-NG Pro

    ```
    wget -O - https://www.eve-ng.net/jammy/install-eve-pro.sh | bash -i
    ```

* EVE-NG Community

    ```
    wget -O - https://www.eve-ng.net/jammy/install-eve.sh | bash -i
    ```

After installation completes, you will need to reboot. Once then system is booted again, you will be starting from the same point as if you started from the iSO or OVF.

You will want to install the remaining dependencies

```
apt-get update && apt-get install eve-ng-dockers -y
```

### Troubleshooting

#### Consoles do not work

`guacd` will listen on `::1`, but is expected to be listening on `127.0.0.1`, you can confirm this is happening by running below

```
root@eve:~# ss -lntp | grep guacd
LISTEN 0      5                   [::1]:4822          [::]:*    users:(("guacd",pid=1196,fd=4))
```

To fix this follow the process below

1. Create `/etc/guacamole/guacd.conf` with the following contents

    ```
    [server]
    bind_host = 127.0.0.1
    bind_port = 4822
    ```

2. Add the following properties to `/etc/guacamole/guacamole.properties`

    ```
    guacd-hostname: 127.0.0.1
    guacd-port: 4822
    ```

3. Restart `guacd` via `systemctl restart guacd`

4. Confirm it is now listening on `127.0.0.1`

    ```
    root@eve:/etc/guacamole# ss -lntp | grep guacd
    LISTEN 0      5               127.0.0.1:4822       0.0.0.0:*    users:(("guacd",pid=29290,fd=4))
    ```

### Caveats

Without a custom kernel for Proxmox, the linux bridges in LXC will be unable to forward several protocols (STP/RSTP/MSTP, LLDP, LACP, etc.) as a linux bridge is compliant with 802.1D and must filter frames for these protocols (they use IEEE 802.1D MAC Bridge Filtered MAC Group Addresses)

[Reference 1](https://interestingtraffic.nl/2017/11/21/an-oddly-specific-post-about-group_fwd_mask/), [Reference 2](https://blog.ipspace.net/2020/12/linux-bridge-lldp.html)

The EVE-NG installation in a VM/OVF already contains a modified kernel that allows these protocols, but is likely not suitable to be used as a proxmox kernel.

