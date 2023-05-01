# WIP WIP WIP WIP WIP
# **Redesigned Volumio initramfs scripting**

This readme gives a quick overview of the files involved and how to integrate them into **volumio3-os**
The idea is to replace the different init scripts (init, init.nextarm, init86 etc) by one and the same script collection.
Instead of moving just one "init" script into initramfs, the new design inserts a series of scripts.
The scripts can have device-specific extensions, this decision is made by the board recipe designer.
Extensions can override and extend existing script functions.

First an explanation of the files involved.

## **scripts/functions**
These are basically taken unmodified from `/usr/share/initramfs-tools/scripts` (Debian 10 from spring 2020, after the release of Volumio 3).
Only a few of the functions are used. Still missing is plymouth preparation, as there were crash difficulties with plymouth at that particular time.

## **scripts/volumio-functions**
These volumio-specific functions and/ or overrides are placed in script file volumio-functions

## **Using Breakpoints**
Breakpoints are designed to let initramfs stop at pre-defined (but optional/ configurable) locations in initramfs.
When a breakpoint is reached, initramfs jumps to a shell. 
It is not an endpoint,leaving the shell continues the initramfs flow. This is different from the way debugging is done with the current version, which stops initramfs alltogether.

Valid breakpoints are:
```
cmdline, modules, backup-gpt, srch-firmw-upd, srch-fact-reset, kernel-rollb, kernel-upd, resize-data, mnt-overlayfs, modfstab, switch-root
```

### Configuring a breakpoint
A kernel cmdline parameter `break=` with a comma-separated list, e.g.

```
break=modules,kernel-upd
```

When reaching a listed breakpoint, `initramfs` will drop to a temporary shell at the defined location.
Here you can inspect/ modify parameters
Using `exit` will return you to the normal initramfs script flow.

## WIP WIP WIP

This documentation is still work in progress.
The initramfs script collection has been thoroughly tested with x86 and arvm7 (Odroid N2).
It has not been verified with an RPi or Primo, but this *should* be just verification, there should be no more functional differences.
(Most of them are obslete as support has stopped for old armv7 boards).

MP1 was added to Volumio later and has also not been verified.  
There is some work to be done for mp1, as this is the only device which received ```init``` modifications for preparing kernel 6.1y. 

# **Quick Edit Initramfs**

Rebuilding an image just for testing initramfs is not very efficient, it is easier just to decompress, edit the script(s) or anything else and then compress again.
Below is a sample script,


## Example script


```bash
#!/bin/bash
TMPBOOT=$HOME/tmpboot
TMPWORK=$HOME/tmpwork
HELP() {
  echo "

Help documentation for initrd editor

Basic usage: edit-initramfs.sh -d /dev/sdx -f volumio.initrd

  -d <dir>	Device with the flashed volumio image
  -f <name>	Name of the volumio initrd file
  -a <arm|arm64> Either 32 or 64bit arm, default arm64

Example: ./edit.initramfs.sh -d /dev/sdb -f volumio.initrd -a arm

Notes:
The script will try to determine how the initrd was compressed and unpack/ repack accordingly
It currently uses 'pluma' as the editor, see variable 'EDITOR'

"
  exit 1
}

EDITOR=pluma
EXISTS=`which $EDITOR`
if [ "x" = "x$EXISTS" ]; then
	echo "This script requires text editor '${EDITOR}'"
    echo "Please install '$EDITOR' or change the current value of variable EDITOR"
	exit 1
fi

ARCH="arm64"
NUMARGS=$#
if [ "$NUMARGS" -eq 0 ]; then
  HELP
fi

while getopts d:f:a: FLAG; do
  case $FLAG in
    d)
      DEVICE=$OPTARG
      ;;
    f)
      INITRDNAME=$OPTARG
      ;;
    a)
      ARCH=$OPTARG
      ;;

    h)  #show help
      HELP
      ;;
    /?) #unrecognized option - show help
      echo -e \\n"Option -${BOLD}$OPTARG${NORM} not allowed."
      HELP
      ;;
  esac
done

if [ -z $DEVICE ] || [ -z $INITRDNAME ]; then
	echo ""
	echo "$0: missing argument(s)"
	HELP
	exit 1
fi


[ -d $TMPBOOT ] || mkdir $TMPBOOT
if [ -d $TMPWORK ]; then
	echo "Workarea exists, cleaning it..."
	rm -r $TMPWORK/*
else
	mkdir $TMPWORK
fi

echo "Mounting volumio boot partition..."
mounted=`mount | grep -o ${DEVICE}1`
if [ ! "x$mounted" = "x" ]; then
	echo "Please unmount this device first"
	exit 1
fi
mount ${DEVICE}1 $TMPBOOT
if [ ! $? = 0 ]; then
	exit 1
fi

pushd $TMPWORK > /dev/null 2>&1
if [ ! $? = 0 ]; then
	exit 1
fi

echo "Making $INITRDNAME backup copy..."
cp $TMPBOOT/$INITRDNAME $TMPBOOT/$INITRDNAME".bak"

FORMAT=`file $TMPBOOT/$INITRDNAME | grep -o "RAMDisk Image"`
if [ "x$FORMAT" = "xRAMDisk Image" ]; then
	echo "Unpacking RAMDisk image $INITRDNAME..."
	dd if=$TMPBOOT/$INITRDNAME bs=64 skip=1 | gzip -dc | cpio -div
	pluma init
	echo "Creating a new $INITRDNAME, please wait..."
	find . -print0 | cpio --quiet -o -0 --format=newc | gzip -9 > $TMPBOOT/$INITRDNAME.new
	mkimage -A ${ARCH} -O linux -T ramdisk -C none -a 0 -e 0 -n uInitrd -d $TMPBOOT/$INITRDNAME.new $TMPBOOT/$INITRDNAME
	rm $TMPBOOT/$INITRDNAME.new
else
	echo "Unpacking gzip compressed $INITRDNAME..."
	zcat $TMPBOOT/$INITRDNAME | cpio -idmv > /dev/null 2>&1
	echo "Starting $EDITOR editor..."
	pluma init
	echo "Creating a new $INITRDNAME, please wait..."
	find . -print0 | cpio --quiet -o -0 --format=newc | gzip -9 > $TMPBOOT/$INITRDNAME
fi

echo "Done."
sync
popd > /dev/null 2>&1
umount ${DEVICE}1
if [ ! $? ]; then
	exit 1
fi

rmdir $TMPBOOT
rm -r $TMPWORK
```
