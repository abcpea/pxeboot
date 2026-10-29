
# Setting up a PXE boot server for a diskless Debian client

## Preface

This is the setup I used to create an environment for a single client to run entirely diskless  
over a direct network connection.  The client uses a separate connection for connecting to the internet.

I'm writing this after being frustrated at trying to find good resources online.  
This is primarily for my own future reference.


## Overview

The following components were used:

- Debian Trixie (Host and client)
- Dnsmasq for DHCP and TFTP
- iPXE Bootloader
- NFS Kernel Server

DHCP provides the location of the bootloader  
TFTP hosts the bootloader  
Persistent Debian OS is installed onto NFS server via debootstrap  

Host IP is 192.168.1.50  
TFTP root folder is /srv/tftp  
Client NFS folder is /srv/nfs/debian-client  


## DHCP and TFTP setup

Install `dnsmasq`

Create the TFTP root folder, set ownership to nobody:nogroup

Put the following in `/etc/dnsmasq.conf` (edit as needed):

    port=0
    interface=enp0s31f6
    enable-tftp
    tftp-root=/srv/tftp
    dhcp-range=192.168.1.100,192.168.1.200,12h

    dhcp-match=set:bios,option:client-arch,0
    dhcp-match=set:efi64,option:client-arch,7
    dhcp-match=set:efi64,option:client-arch,9
    dhcp-match=set:ipxe,175

    dhcp-boot=tag:bios,undionly.kpxe
    dhcp-boot=tag:efi64,ipxe.efi
    dhcp-boot=tag:ipxe,boot.ipxe
---
> You can get the name of the interface with `ip addr`.
>
> Also, make sure it has the right static IP address  
> (192.168.1.50 in this guide, can be set with `nmtui` or w/e)

---
Restart the dnsmasq service


## NFS setup

Install `nfs-kernel-server`

Create the `/srv/nfs/debian-client` directory

Stick this into /etc/exports:  
`/srv/nfs/debian-client  192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash)`

Run `exportfs -ra` as superuser


## Bootloader setup

Install `ipxe`

Copy `ipxe.efi` and `undionly.kpxe` from `/usr/lib/ipxe/` to `/srv/tftp/`

Create file `/srv/tftp/boot.ipxe` containing:

    #!ipxe
    
    set server_ip 192.168.1.50
    set nfs_path /srv/nfs/debian-client
    
    kernel nfs://${server_ip}${nfs_path}/vmlinuz root=/dev/nfs nfsroot=${server_ip}:${nfs_path},rw ip=dhcp
    initrd nfs://${server_ip}${nfs_path}/initrd.img
    
    boot || read void


## Client OS installation
---
> A lot of trial and error here, some steps may be missing or unnecessary

---
Run `# debootstrap stable /srv/nfs/debian-client http://deb.debian.org/debian/`

To chroot into the system, I used the following in a script (ran as superuser):

    #!/bin/bash
    
    mount --bind /dev /srv/nfs/debian-client/dev
    mount --bind /proc /srv/nfs/debian-client/proc
    mount --bind /sys /srv/nfs/debian-client/sys

    chroot /srv/nfs/debian-client

    umount /srv/nfs/debian-client/dev
    umount /srv/nfs/debian-client/proc
    umount /srv/nfs/debian-client/sys
---
> This is convenient when troubleshooting. 
> 
> Using a plain linux terminal is recommended.
>
> If you need to enter chroot AFTER successfully connecting the client,
> you will need to copy /etc/resolv.conf from the host if you want internet.

---


### In the chroot

Uncomment `/etc/locale.gen` appropriately and run `locale-gen`

Edit some stuff:

#### /etc/fstab:

    proc    /proc   proc    defaults    0   0
    sysfs   /sys    sysfs   defaults    0   0

#### /etc/hosts:

    127.0.0.1	localhost
    127.0.1.1	debian-client
    ::1		localhost ip6-localhost ip6-loopback
    ff02::1		ip6-allnodes
    ff02::2		ip6-allrouters

#### /etc/hostname:

    debian-client

#### /etc/apt/sources.list:

    deb https://deb.debian.org/debian trixie contrib main non-free non-free-firmware
    deb https://deb.debian.org/debian trixie-updates contrib main non-free non-free-firmware
    deb https://security.debian.org/debian-security trixie-security contrib main non-free non-free-firmware

Update apt cache and install:

`linux-image-amd64 locales console-setup nfs-common dhcpcd sudo`

Install whatever else you need: firmware, desktop environment, etc

Set root password, add a user, setup sudo privileges, and whatever else you need to do.

Exit the chroot and unmount the the pseudo filesystems (script does this automatically on exit)

## Connecting the client

You plug in the client computer and boot PXE

# END?
