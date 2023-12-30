# 
|Date|Author|Change|
|---|---|---|
|20230501|gkkpch|Initial V1.0
|20230621||Starting the documention
|20230622||Adding the old (20210209) build environment as a reference
|20231224||Added first "Malaga" amendments
|||Documentation in progress

## ```##TODO```

|Task|Status|
|---|---|
|Check all init scripts for changes after december 2020|open|
|Test whether the overriding of functions works properly|open
|Board-specifics for mp1|open
|Verify RPi USB boot|open (find someone to do this)
|Pick ip the plymouth issue a default (debian futureprototype) seems to startup and hold until booted|open
|Backport Plymouth in case successfull|open


# **Volumio initramfs implementation**

This readme gives a quick overview of the files involved and how to integrate them into **volumio3-os**.  
This documentation is still WIP.

One of the main reasons for refactoring this was the lack of debugging capabilities in the current init script versions.  
Where most of the arm boards have the ability to add a console log to monitor/debug the boot process, x86 does not have that opportunity.
Also, using the chosen approach brings Volumio's init script into a structure closer to the original initramfs scripts as used with  Debian and Ubuntu.

During the development process, the currently maintained init scripts (init, init.nextarm, init86 etc.) were replaced by a single initramfs script set, covering the Raspberry PI, various other armv7 boards and x86.  
Replacing the Volumio init script with the original initramfs approach from Ubuntu and Debian also means that, instead of moving just one "init" script into initramfs, the new design now inserts a basic collection of initramfs scripts. 

Though the use of init "hooks" does not really apply to Volumio, the available initramfs general functions have been used as much as possible. Volumio-specific functions are added to a separate script module.
Volumio does not need all initramfs functions and therefore only uses a part of them. As mentioned, Volumio does not use hooks like the standard initramfs does, but implements something similar (without calling it a hook).

Volumio-specific scripts are not allowed to be modified and implement function extensions. This ought to be done in a separate function module (script) ```custom-functions``` by adding the modified function from ```volumio-functions``` (it willl override the standard one) or by adding a function (extension).  
 
An extension can be therefore be used to implement a device-specific requirement. 
Extensions are therefore board-specific and the board implementer's responsibility. Extensions are always a combination of an overridden volumio function and an addition.



  


# **Contents**
|File/ folder|Contents
|---|---|
|conf|Environment configuration	
|scripts|initramfs scripts (custom and standard functions)
|init|init script
|mkinitramfs-custom.sh|adapted script (replace standard initramfs script folder)


## Folder **scripts/functions**
This is basically an unmodified copy from ```/usr/share/initramfs-tools/scripts/functions```.  
Only a few of them are used.  
(Based on Debian 10 from spring 2020, after the release of Volumio 3, still valid). 
 
Known issue: Still missing is plymouth preparation, as there were crash difficulties with plymouth at that particular time.

## **scripts/volumio-functions**
Standard Volumio functions, used with all boards.

## **scripts/custom-functions**
Board-specific functions and/ or overrides are placed in script file ```custom-functions```. 

## **mkinitramfs-custom.sh**

The customized ```mkinitramfs``` build script for Volumio has been modified to replace the standard initramfs hook scripts by the above used scripts for volumio, see function ```build_initrd()```.

# Debugging

## **1. Using ```debug``` and ```use_kmsg``` cmdline parameters**
The use of ```debug``` will allow you to debug the init and functions scripts.  
Combined with the ```use_kmsg``` parameter, the debug messages will either be printed on the console (usekmsg=yes) or to the debug file in ```/run/initramfs/initramfs.debug```.  
The debug file can be accessed during initramfs processing ***and*** after the boot process has completed.

## 2. **Using Breakpoints**
Breakpoints are a big step forward for debugging, designed to let initramfs stop at optional locations in initramfs, the locations themselves are pre-defined, but can be configured with the ```break``` cmdline parameter.     
When a breakpoint is reached, initramfs jumps to a temporary busybox shell.  
Here you can check parameters, look at the kernel log (dmesg) or check the initramfs.debug file (when ```debug``` has been configure, combined with ```use_ksmsg=yes```, see above). 

You could add your own breakpoints, but that would mean a modification of the init script.   
Use ```do_break()``` for that and remove them when you are finished.


### Valid breakpoints are:
```
cmdline, modules, backup-gpt, srch-firmw-upd, srch-fact-reset, kernel-rollb, kernel-upd, resize-data, mnt-overlayfs, modfstab, switch-root
```

### Configuring a breakpoint
A kernel cmdline parameter `break=` with a comma-separated list, e.g.

```
break=modules,kernel-upd
```

Dropping to a shell will not cause a kernel panic, when leaving the shell with ```exit```.   
The initramfs flow will continue aftre the breakpoints, you can use as many breakpoints from the list as you need.  
This is different from the way debugging is done with the current version, which stops initramfs alltogether and then causes a kernel panic.  

## WIP WIP WIP

This documentation is still work in progress and will be completed soon with a comprehensive description of the components involved.  
The initramfs script collection has been thoroughly tested with x86 and arvm7 (Odroid N2).  
It has not been verified with an RPi or Primo, but this *should* be just verification, as there *should* be no more functional differences.  
(Most of the original differences are now obsolete as support has stopped for old armv7 boards).  

MP1 was added to Volumio later and has also not been verified.   
This is relevant, because some work needs to be done for mp1, it is the only device which received very specific ```init``` updates because of necessary modifications with preparing kernel 6.1y: it needs u-boot to be updated as well..  

# **Quick Edit Initramfs**

Rebuilding an image just for testing initramfs is not very efficient, it is easier just to decompress, edit the script(s) or anything else and then compress again.  
Below is a sample script


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
