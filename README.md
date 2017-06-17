# Configuration of spud, a Pi Zero running FreeBSD 11

Initial configuration done manually; subsequent config done using Ansible.

**WORK IN PROGRESS**

## Prepare the MicroSD card

1. Download a FreeBSD 11 disk image for the Raspberry Pi / Pi Zero plus checksums:

        cd ~/Downloads/
        wget FreeBSD-11.0-STABLE-arm-armv6-RPI-B-20170510-r318134.img.xz ]
        wget CHECKSUM.SHA512-FreeBSD-11.0-STABLE-arm-armv6-RPI-B-20170510-r318134.xz

1. Verify the downloaded image:

        sha512sum -c CHECKSUM.SHA512-FreeBSD-11.0-STABLE-arm-armv6-RPI-B-20170510-r318134.xz

1. Blast the image onto a particular MicroSD card (my card was seen as `/dev/mmcblk0` on my Linux laptop):

        unxz -c FreeBSD-11.0-STABLE-arm-armv6-RPI-B-20170510-r318134.img.xz | sudo dd of=/dev/mmcblk0

## First boot

1. Insert the MicroSD card into the Pi
1. Attach a HDMI monitor and a USB + Ethernet hub
1. Attach an Ethernet cable and USB keyboard to the hub
1. Power on the Pi
1. Post-boot, log in as `root` (password: `root`)
1. The `sshd` service is enabled by default; bring it down until user passwords have been changed from their defaults:

        service sshd stop

1. Change the `root` and `freebsd` (default non-privileged user) users' passwords:

        for u in root freebsd; do
            passwd $u
        done

1. Rename the `freebsd` user as something more sensible (`sa_will`)

        vipw  # then replace freebsd with sa_will
        vigr  # then replace freebsd with sa_will

        mv /home/{freebsd,sa_will}
        mv /var/mail/{freebsd,sa_will}

1. Ensure that the DHCP client syncronously picks up an address for the `ue0` interface at boot (NB `sysrc` safely updates `/etc/rc.conf`):

        sysrc ifconfig_ue0="SYNCDHCP"
        service netif restart

1. Check that an address has been acquired via DHCP and DNS name resolution is working:

        ifconfig  # shows Ethernet interface as device `ue0`
        ping -c 1 www.last.fm

1. Set the hostname and keymap:

        sysrc hostname="spud"
        sysrc keymap="uk"

1. SSHD config: uncomment the `Port=` line and set the port to something other than 22.
   Let's refer to that port number as `$SSH_PORT` from now on.

1. Start the `sshd` service again

        service sshd start

1. Check if can ssh to the machine by trying to copy a SSH public key to the `sa_will` user's list of authorized keys:

        # From my laptop machine
        ssh-copy-id -i ~/.ssh/id_rsa.pub -p $SSH_PORT sa_will@spud
        ssh -p $SSH_PORT sa_will@spud

1. Disable password authentication by ensuring that the following lines are present in `/etc/ssh/sshd_config`: 

    * `ChallengeResponseAuthentication no`
    * `PasswordAuthentication no`

1. Restart the SSH daemon:

        service sshd restart 

1. Install the `pkgng` package management utility and update the package database:

        env ASSUME_ALWAYS_YES=yes pkg bootstrap
        pkg update

1. Install Python (required as we later want to use Ansible for configuration management):

       pkg install python34

1. Timezone: set appropriately

        tzsetup /usr/share/zoneinfo/Europe/London

1. NTP: set up time syncing at boot

        sysrc ntpd_enable="YES"
        sysrc ntpd_sync_on_start="YES"
        service ntpd start

        # Check the system time
        date

1. Install bash:

        pkg install bash

        # Required to use bash on FreeBSD (see https://forums.freebsd.org/threads/50214/)
        echo 'fdesc /dev/fd  fdescfs  rw 0 0' >> /etc/fstab
        mount -a

1. Set the `sa_will` user's shell to bash:

        chpass -s /usr/local/bin/bash sa_will

        # Check that this has worked
        su - sa_will -c 'echo $SHELL'

1. Install sudo (needed for remote config using Ansible; couldn't get `become_method=su` to work)

        pkg install sudo

1. Ensure that `sa_will` can become root and can run anything with sudo (after supplying a password):

        pw usermod sa_will -G wheel

        visudo  # Ensure that '%wheel ALL=(ALL) ALL' is not commented out

1. Reboot!

## Ansible

**TODO**

Testing using

```
ansible-playbook --inventory=inventory.ini --ask-become-pass playbook.yml
```

and an `inventory.ini` containing

```
[home-servers]
spud ansible_connection=ssh ansible_host=spud ansible_port=65000 become_method=sudo become=true ansible_user=sa_will
```
