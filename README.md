# Configuration of spud, a Pi Zero running FreeBSD 11

Initial configuration done manually; subsequent config done using Ansible.

**WORK IN PROGRESS**

## Prepare the MicroSD card

1. Download FreeBSD 11 disk image for Raspberry Pi / Pi Zero plus checksum
    ```csh
    cd ~/Downloads/
    for f in FreeBSD-11.0-STABLE-arm-armv6-RPI-B-20170510-r318134.img.xz CHECKSUM.SHA512-FreeBSD-11.0-STABLE-arm-armv6-RPI-B-20170510-r318134.xz; do 
        wget ftp://ftp.freebsd.org/pub/FreeBSD/snapshots/arm/armv6/ISO-IMAGES/11.0/$f
    done
    ```
1. Verify downloaded image
    ```csh
    sha512sum -c CHECKSUM.SHA512-FreeBSD-11.0-STABLE-arm-armv6-RPI-B-20170510-r318134.xz
    ```
1. Blast the image onto a particular MicroSD card (/dev/mmcblk0):
    ```csh
    unxz -c FreeBSD-11.0-STABLE-arm-armv6-RPI-B-20170510-r318134.img.xz | sudo dd of=/dev/mmcblk0
    ```

## First boot

1. Insert the MicroSD card into the Pi
1. Attach a HDMI monitor and a USB + Ethernet hub
1. Attach an Ethernet cable and USB keyboard to the hub
1. Power on the Pi
1. Post-boot, log in as `root` (password: `root`)
1. Rename the `freebsd` user as something more sensible (`sa_will`)
    ```csh
    vipw  # then replace freebsd with sa_will
    mv /home/freebsd /home/sa_will
    mv /var/mail/freebsd /var/mail/sa_will
    vigr  # then replace freebsd with sa_will
1. The `sshd` service is enabled by default; bring it down until user passwords have been changed from their defaults:
    ```csh
    service sshd stop
    ```
1. Change the `root` and `sa_will` users' passwords:
    ```csh
    for u in root sa_will; do
        passwd $u
    done
    ```
1. Ensure that the DHCP client syncronously picks up an address for the `ue0` interface at boot; the following needs to be in `/etc/rc.conf`:
    ```
    ifconfig_ue0="SYNCDHCP"
    service netif restart
    ```
1. Check that an address has been acquired via DHCP and DNS name resolution is working:
    ```csh
    ifconfig  # shows Ethernet interface as device `ue0`
    ping -c 1 www.last.fm
    ```
1. Set the hostname and keymap in `/etc/rc.conf` (safely using the `sysrc` tool):
    ```csh
    sysrc hostname="spud"
    keymap="uk"
    ```
1. SSHD config: uncomment the `Port=` line and set the port to something other than 22
1. Start the `sshd` service again
    ```csh
    service sshd start
    ```
1. Check if can ssh to the machine by trying to copy a SSH public key to the `sa_will` user's list of authorized keys:
    ```csh
    # From another machine
    ssh-copy-id -i ~/.ssh/id_rsa.pub -P ??? sa_will@spud
    ssh -P ??? sa_will@spud
    ```
1. Disable password authentication by ensuring that `ChallengeResponseAuthentication no` is present in `/etc/ssh/sshd_config` then
    ```csh
    service sshd restart 
    ```
1. Install the `pkgng` package management utility and update the package database:
    ```csh
    env ASSUME_ALWAYS_YES=yes pkg bootstrap
    pkg update
    ```
1. Install Python (required by Ansible; using most recent 3.x version)
   ```
   pkg install python34
   ```
1. Timezone: set appropriately
    ```csh
    tzsetup /usr/share/zoneinfo/Europe/London
    ```
1. NTP: set up time syncing at boot
    ```csh
    sysrc ntpd_enable="YES"
    sysrc ntpd_sync_on_start="YES"
    service ntpd start
    date
    ```
1. Install bash and set it as as the `sa_will` user's shell
    ```csh
    pkg install bash
    echo 'fdesc /dev/fd  fdescfs  rw 0 0' >> /etc/fstab
    mount -a
    chpass -s /usr/local/bin/bash sa_will
    su - sa_will -c 'echo $SHELL'
    ```
1. Install sudo (needed for remote config using Ansible; couldn't get `become_method=su` to work)
    ```csh
    pkg install sudo
    ```
1. Ensure that `sa_will` can become root and can run anything with sudo (after supplying a password):
    ```csh
    pw usermod sa_will -G wheel
    visudo
    # Ensure that '%wheel ALL=(ALL) ALL' is not commented out
    ```

## Ansible

**TODO**

Testing using

```
ansible-playbook --inventory=inventory.ini --private-key=${HOME}/.ssh/id_rsa --become --ask-become-pass --become-method=sudo --user=sa_will playbook.yml
```

and an `inventory.ini` containing

```
[home-servers]
spud ansible_connection=ssh ansible_host=????? ansible_become=true ansible_become_method=sudo ansible_port=?????
#ansible_connection=paramiko
```
