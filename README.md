# Sheevaplug installation notes

Original document is in Google Docs: _Shevaplug Installation Notes_.

Required (uBoot,uImage,etc..) files are .gitignored in the checkout repo on dirac.

_objective_: try to install node.js + couchdb

*Jaunty (the original distributed OS) is end-of-life: gonna re-install debian*
## References
* [Plug Computer Community site](http://plugcomputer.org/)
* [Plug Computer Community Wiki](http://plugcomputer.org/plugwiki/index.php?title=Main_Page)
* [Intall uBoot](http://www.cyrius.com/debian/kirkwood/sheevaplug/uboot-upgrade.html)
* [FTDI Drivers OSX](http://www.plugcomputer.org/plugwiki/index.php/Serial_terminal_program)
* [Debian install to USB of SD/MMC](http://www.cyrius.com/debian/kirkwood/sheevaplug/install.html)


## Serial console acces (From dirac)
Talking to the sheevaplug, especially to interact with the boot manager requires serial access through the miniusb port.

### OSX Driver
Download FTDI Driver from : http://www.ftdichip.com/Drivers/VCP.htm  
`ref` from http://www.plugcomputer.org/plugwiki/index.php/Serial_terminal_program

        Patch /System/Library/Extensions/FTDIUSBSerialDriver.kext/Contents/Info.plist
        make sure perms are correct.
        chown root.wheel /System/Library/Extensions/FTDIUSBSerialDriver.kext/Contents/Info.plist
        ## to test, you can see if line below complains
        sudo kextload /System/Library/Extensions/FTDIUSBSerialDriver.kext

Talk to serial port with:

-dirac:

    screen /dev/tty.usbserial-00002006B 115200
    # ctrl-a k to kiill screen session

-goedel:

    screen /dev/ttyUSB2 115200
    screen /dev/ttyUSB1 115200
    # ctrl-a k to kiill screen session

# Low level Install (uBoot, Debian installer)
When we moved from the included Jaunty ubuntu distro to standard (maintained) debian, we had to install a new boot manager (uBoot)

## Uboot Needs an upgrade
We must note the ethaddress because it will be lost by this upgrade:  
  http://www.cyrius.com/debian/kirkwood/sheevaplug/uboot-upgrade.html

    Marvell>> version
    U-Boot 1.1.4 (Mar 19 2009 - 16:06:59) Marvell version: 3.4.16
    Marvell>> print ethaddr
    ethaddr=00:50:43:01:D1:DB

Use tftp rather than USB for uImage,uInitrd (uBoot, but we already did that)  

-start tftp

    sudo scp u-boot.bin-3.4.19  /private/tftpboot/uboot.bin
    sudo /sbin/service tftp start

-update uBoot tftp from dirac

    setenv ipaddr 192.168.3.250
    setenv serverip 192.168.3.140
    tftpboot 0x0800000 uboot.bin
    nand erase 0x0 0xa0000
    nand write 0x0800000 0x0 0xa0000
    reset
   
    setenv ethaddr 00:50:43:01:D1:DB
    saveenv
    reset

## Install a new Kernel into Flash
*Will redo this with 2.6.32.5 to match the version installed with debian installer*
Get a kernel from sheeva.with-linux.com.

    wget http://sheeva.with-linux.com/sheeva/2.6/2.6.32/2.6.32.45/sheeva-2.6.32.45-uImage
    sudo scp sheeva-2.6.32.45-uImage /private/tftpboot
    
This wont be used just yet, but it will be usefull for later when we want to boot from this kernel

    setenv ipaddr 192.168.3.250
    setenv serverip 192.168.3.140
    tftpboot 0x2000000 sheeva-2.6.32.2-uImage
    iminfo
    nand erase 0x100000 0x400000
    nand write 0x2000000 0x100000 0x400000
   
    setenv mainlineLinux yes
    setenv arcNumber 2097
    saveenv
    reset

## Debian installer
I choose stable images (as opposed to dailys).

    wget ftp://ftp.debian.org/debian/dists/stable/main/installer-armel/current/images/kirkwood/netboot/marvell/sheevaplug/uImage
    wget ftp://ftp.debian.org/debian/dists/stable/main/installer-armel/current/images/kirkwood/netboot/marvell/sheevaplug/uInitrd

    sudo scp uImage uInitrd  /private/tftpboot

*Install with console on goedel, cause I get garbled text from dirac (in the debian installer)!?*  
*`export LANG=C` got me the first screen of the installer but garbled again...

    setenv ipaddr 192.168.3.250
    setenv serverip 192.168.3.140
    tftpboot 0x0400000 uImage
    tftpboot 0x0800000 uInitrd
    setenv bootargs console=ttyS0,115200 base-installer/initramfs-tools/driver-policy=most
    bootm 0x0400000 0x0800000
    
I didn't cathc the debian installer option to skip the kernel installation, so we do have a kernel in a boot partition

## Boot from MMC (similar for USB)
Just a test. 
Don't save the setting this is just to confirm that we can boot from the kernel installed on the sd/mmc card.
There was a typo in the instructions: `mmc init` -> `mmcinit`.
This works and is confirmed with `uname` reporting `Linux miraplug001 2.6.32-5-kirkwood` which is the kernel installed into mmc:/boot by the debian installer.


    setenv bootargs_console console=ttyS0,115200
    setenv bootcmd_mmc 'mmcinit; ext2load mmc 0:1 0x00800000 /uImage; ext2load mmc 0:1 0x01100000 /uInitrd'
    setenv bootcmd 'setenv bootargs $(bootargs_console); run bootcmd_mmc; bootm 0x00800000 0x01100000'
    # NOT permanent so dont - saveenv
    run bootcmd

This is the USB version

    setenv bootargs_console console=ttyS0,115200
    setenv bootcmd_usb 'usb start; ext2load usb 0:1 0x00800000 /uImage; ext2load usb 0:1 0x01100000 /uInitrd'
    setenv bootcmd 'setenv bootargs $(bootargs_console); run bootcmd_usb; bootm 0x00800000 0x01100000'
    # NOT permanent so dont - saveenv
    run bootcmd
    
## Boot with MMC/SD (USB) rootfs, and nand flash kernel
Now we want to boot with the previously installed (nand-flash kernel), but root fs on USB/MMC.
This works and is confirmed with `uname` reporting `Linux miraplug001 2.6.32.45` which is the kernel we installed into nand-flash above.
The befault (uboot) bootargs has to be overriden to specify root param.
We will not persist this either becaus we will convert to flash in the next step..

    setenv bootargs console=ttyS0,115200 mtdparts=nand_mtd:0xa0000@0x0(u-boot),0x400000@0x100000(uImage),0x1fb00000@0x500000(rootfs) root=/dev/mmcblk0p2 rootwait rw
    boot
    
This is the USB version

    setenv bootargs console=ttyS0,115200 mtdparts=nand_mtd:0xa0000@0x0(u-boot),0x400000@0x100000(uImage),0x1fb00000@0x500000(rootfs) root=/dev/sda2 rootwait rw
    boot

    ln -fs /proc/mounts /etc/mtab   # prevent a stale mtab from confusing things later on...

## Install to internal flash
    
### Convert internal flash root partition to UBIFS 

    apt-get install mtd-utils

    ubiformat /dev/mtd2 -s 512
    ubiattach /dev/ubi_ctrl -m 2 

    ubimkvol /dev/ubi0 -N rootfs -m
    mount -t ubifs ubi0:rootfs /mnt

### Clone USB rootfs to internal flash

    mkdir /tmp/rootfs
    mount -o bind / /tmp/rootfs/
    cd /tmp/rootfs
    sync
    cp -a . /mnt/

Fix fstab:

    cat <<EOF > /mnt/etc/fstab
    /dev/root  /               ubifs   defaults,noatime,rw                      0 0
    tmpfs      /var/run        tmpfs   size=1M,rw,nosuid,mode=0755              0 0
    tmpfs      /var/lock       tmpfs   size=1M,rw,noexec,nosuid,nodev,mode=1777 0 0
    tmpfs      /tmp            tmpfs   defaults,nosuid,nodev                    0 0
    EOF

Now set the boot

    setenv bootargs 'console=ttyS0,115200 ubi.mtd=2 root=ubi0:rootfs rootfstype=ubifs'
    saveenv
    reset
    
## Updating the kernel
In the original instructions the debian installer was to skip installing the kernel, we did install it.
In any case

As we installed the nand flasshed kernel 2.6.32.45 does not match the debian installed version, so we will use thi `upgrade` to re-install the 2.6.32.5 kernel. This is the same method that would be used to update the kernel.


You need to specify the version of the kernel to update to. e.g.:

    wget http://sheeva.with-linux.com/sheeva/README-PLUG-UPDATE.sh
    chmod +x README-PLUG-UPDATE.sh 
    ./README-PLUG-UPDATE.sh 2.6.32.5 --nandkernel
    
After a reboot `uname` confirms: `Linux miraplug001 2.6.32.5 #1 PREEMPT`


# Configuration (Software/Services)

So as to not pollute the flash, we will remount the SD, and build node there!
Remember /dev/mmcblk0p1 was /boot, /dev/mmcblk0p2 was /.
`df` report 243M used before we start; 309M after build-essentials et al.; and 321M after node install

    mkdir /mnt
    mount /dev/mmcblk0p2 /mnt
    cd /mnt/home/daniel

## Install `node.js`:
    apt-get update
    apt-get install scons make build-essential libssl-dev curl
    cd /mnt/home/daniel
    wget http://nodejs.org/dist/node-v0.4.12.tar.gz
    tar zxvf node-v0.4.12.tar.gz
    cd node-v0.4.12
    nano +139 deps/v8/SConstruct
    'CCFLAGS':      ['$DIALECTFLAGS', '$WARNINGFLAGS', '-march=armv5t'],

    # worked took 40 minutes
    ./configure
    make
    make install

## Install `npm`
    apt-get install curl
    curl http://npmjs.org/install.sh | sh

# Tadaa !
    uname -a; node --version; npm --version
    Linux miraplug001 2.6.32-5-kirkwood #1 Sat Sep 10 09:17:13 UTC 2011 armv5tel GNU/Linux
    v0.4.12
    1.0.30


Convert internal flash root partition to UBIFS  
http://plugcomputer.org/plugwiki/index.php/Installing_Debian_To_Flash

Hint for ubifs boot loading:  
http://www.plugcomputer.org/plugforum/index.php?topic=5894.0

Now that we have booted from USB rootfs (and installed node!), we want to burn the USB fs onto the internal flash:

apt-get install mtd-utils

ubiformat /dev/mtd2 -s 512
ubiattach /dev/ubi_ctrl -m 2

ubimkvol /dev/ubi0 -N rootfs -m
mount -t ubifs ubi0:rootfs /mnt


Stop - rootfs not big enough! -OR- maybe it is with compression
root@miraplug001:/# df -h / /boot
Filesystem            Size  Used Avail Use% Mounted on
/dev/sda2             6.8G  788M  5.7G  12% /
/dev/sda1             228M   15M  202M   7% /boot
root@miraplug001:/# df -h /mnt/
Filesystem            Size  Used Avail Use% Mounted on
ubi0:rootfs           462M   16K  458M   1% /mnt

cleanup - need 326M (before compression)
/var/cache/apt: 51M
/root: 102M
/usr/share/locale 65M

rm -rf /root/node-source /root/ax-dn : 690M left on /
# apt-get install localepurge : no effect, remove

apt-get clean: 667M left

GO, try copying, see what compression does:
mkdir /tmp/rootfs
mount -o bind / /tmp/rootfs/
cd /tmp/rootfs
sync

cp -a . /mnt/
whoa: compression helps!
root@miraplug001:/tmp/rootfs# df -h /mnt
Filesystem            Size  Used Avail Use% Mounted on
ubi0:rootfs           462M  295M  163M  65% /mnt

Fix fstab, or forever suffer the ignominy of a failed mount:
cat <<EOF > /mnt/etc/fstab
/dev/root  /               ubifs   defaults,noatime,rw                      0 0
tmpfs      /var/run        tmpfs   size=1M,rw,nosuid,mode=0755              0 0
tmpfs      /var/lock       tmpfs   size=1M,rw,noexec,nosuid,nodev,mode=1777 0 0
tmpfs      /tmp            tmpfs   defaults,nosuid,nodev                    0 0
EOF

shutdown -r now # cross fingers... nah, it'll work, what could go wrong?

# following is on one line
setenv bootargs 'console=ttyS0,115200 ubi.mtd=2 root=ubi0:rootfs rootfstype=ubifs'

saveenv
reset

Now we have booted with rootfs on flash, but kernel is still on usb.
bootcmd ....

update kernel ??
wget http://sheeva.with-linux.com/sheeva/README-PLUG-UPDATE.sh

bash ./README-PLUG-UPDATE.sh  --nandkernel
bash ./README-PLUG-UPDATE.sh  2.6.32 --nandkernel

Hmm how dowe boot from nand/ ubifs ? Hints here ?
  http://www.plugcomputer.org/plugforum/index.php?topic=5894.0

#WIP::::::    do this once
setenv bootargs 'console=ttyS0,115200 mtdparts=orion_nand:512k(uboot),4m@1m(kernel),507m@5m(rootfs) rw';


setenv bootcmd 'setenv bootargs $(bootargs_console); run bootcmd_usb; bootm 0x00800000 0x01100000'



-=-=-= Try re-install with mmc/sd