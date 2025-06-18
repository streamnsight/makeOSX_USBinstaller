# Make a USB bootable Mac OS Installer on linux or a newer mac

The downloadable installer to create a USB requires to run on a computer that supports the OS you're trying to install. That makes it hard to recover from a failing recovery disk. On old MacBooks, the Internet recovery also fails due to some files not being reachable, so unless you have a spare MBP, you are out of luck. 

But you can do this on Linux or a newer Mac, using this process:

Linux shell script that creates bootable USB flash drive with OS X installer.

OS X installer application contains a disk image "InstallESD.dmg" that can be
used to create a bootable USB flash drive. The procedure is well described in
the media, see references below.

This script automates process on Linux platform, doing essentially the
following:

    apt-get install hfsprogs e2fsprogs dmg2img rsync

    # create folders for temporary mounts
    mkdir -p /mnt/OSX_InstallESD /mnt/OSX_BaseSystem /mnt/usbstick

    # convert installer disk image to raw format
    dmg2img "Install OS X <Version>.app/Contents/SharedSupport/InstallESD.dmg" InstallESD.img
    # create device mapping for the InstallESD image
    kpartx -a InstallESD.img
    # mount the volume to /mnt/OSX_InstalledESD
    mount /dev/mapper/loop0p2 /mnt/OSX_InstallESD

    # convert base system disk image to raw format
    dmg2img /mnt/OSX_InstallESD/BaseSystem.dmg BaseSystem.img
    # create device mapping for the BaseSystem image
    kpartx -a BaseSystem.img
    # mount the volume to /mnt/OSX_BaseSystem
    mount /dev/mapper/loop1p1 /mnt/OSX_BaseSystem

    # partition the USB flash drive, /dev/sdX
    # clear the partition table
    sgdisk -o /dev/sdX
    # create new partition 1:0:0 (partnum:start(default=0):end(all=0)) -t 1:AF00 (partition type hfs+) -c name -A 1:set:2 (make bootable) /dev/sdX (device) 
    sgdisk -n 1:0:0 -t 1:AF00 -c 1:"disk image" -A 1:set:2 /dev/sdX
    # format filesystem with type hfsplus
    mkfs.hfsplus -v "OS X Base System" /dev/sdX1
    # mount the resulting volume to /mnt/usbstick
    mount /dev/sdX1 /mnt/usbstick

    # copy installer files
    # copy the Base System boot files
    rsync -aAEHW /mnt/OSX_BaseSystem/ /mnt/usbstick/
    # remove symlink to Packages
    rm -f /mnt/usbstick/System/Installation/Packages
    # copy install file packages to /System/Installation/Packages
    rsync -aAEHW /mnt/OSX_InstallESD/Packages /mnt/usbstick/System/Installation/
    # copy chunklist
    rsync -aAEHW /mnt/OSX_InstallESD/BaseSystem.chunklist /mnt/usbstick/
    # copy BaseSystem.dmg to root of stick (used for recovery volume?)
    rsync -aAEHW /mnt/OSX_InstallESD/BaseSystem.dmg /mnt/usbstick/
    # be sure to sync before unmounting the stick
    sync

Usage: `./mkosxinstallusb.sh </dev/sdX> "Install OS X <Version>.app"`, where
`</dev/sdX>` is a block device for target USB flash drive, e.g. `/dev/sdb`.

Known problems:
* High Sierra installers are not supported yet, 
* Korean localization can be omitted due to Linux/hfsplus bug, see
  https://ubuntuforums.org/showthread.php?t=1422374

References:
* https://www.afp548.com/2012/05/30/understanding-installesd-recovery-hd-internet-recovery/
* https://www.macworld.co.uk/how-to/mac-software/how-make-bootable-mavericks-install-drive-3475042/
* https://www.macworld.com/article/2367748/os-x/how-to-make-a-bootable-os-x-10-10-yosemite-install-drive.html
