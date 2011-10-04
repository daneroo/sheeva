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
    ctrl-a k to kiill screen session

-goedel:

    screen /dev/ttyUSB2 115200
    screen /dev/ttyUSB1 115200
    ctrl-a k to kiill screen session

# Low level Install (uBoot, Debian installer)
When we moved from the included Jaunty ubuntu distro to standard (maintained) debian, we had to install a new boot manager (uBoot)

## Uboot Needs an upgrade
We must note the ethaddress because it will be lost by this upgrade:  
  http://www.cyrius.com/debian/kirkwood/sheevaplug/uboot-upgrade.html

    Marvell>> version
    U-Boot 1.1.4 (Mar 19 2009 - 16:06:59) Marvell version: 3.4.16
    Marvell>> print ethaddr
    ethaddr=00:50:43:01:D1:DB

Use tftp raher that USB for uImage,uInitrd (uBoot, but we already did that)  
  OS X: sudo /sbin/service tftp start (put files in /private/tftpboot/)

**Install with console on goedel, cause I get garbled text from dirac (in the debian installer)!?

Install debian following: (from usb FAT --or-- tftp)
http://www.cyrius.com/debian/kirkwood/sheevaplug/install.html

-- will try this later see below:
http://plugcomputer.org/plugwiki/index.php/Installing_Debian_To_Flash

## Install `node.js`:
apt-get update
apt-get install scons make build-essential libssl-dev curl
cd /root/node-source
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

uname -a; node --version; npm --version
Linux miraplug001 2.6.32-5-kirkwood #1 Sat Sep 10 09:17:13 UTC 2011 armv5tel GNU/Linux
v0.4.12
1.0.30


Convert internal flash root partition to UBIFS
http://plugcomputer.org/plugwiki/index.php/Installing_Debian_To_Flash

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