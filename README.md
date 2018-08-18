# raspbian-f2fs - Flash Friendly Filesystem for the Raspberry Pi

Special thanks to *d-a-v/raspbian-f2fs* for creating the original script.
I started out making only a few changes to his original script, but an evening later I had almost completely rewritten it.
Changes Include:
* Better error handling.
* More verbosity.
* F2FS filesystem expansion for image targets.
* Creates an entirely new partition table instead of copying it from the source image.
* Uses `parted` instead of `fdisk` (because it has built-in scripting support).
* Now uses `losetup` to automatically determine partition offsets of source and target images.
* Progress indicators when transferring files from source to target (via `pv`).
* Determines if required packages are installed on the host and, if not, installs them.
* Uses a `chroot` environment to make changes to the target.
* Automatically installs `f2fs-tools` into the target.
* And more!

This script takes a Raspbian/Raspbian Lite .zip or .img file and either:
* Writes an F2FS installation directly to an SD card or USB flash device.
(The root partition is expanded to fill the device during creation.)
* Creates a new F2FS image that can be flashed just like the original image.
(The root partition will be automatically expanded during first boot.)

In either case it boots fine and seems to work well. I've been using F2FS in
a couple of production Pi's for several months with basically no issues.

Known Working: **Raspbian 2018-06-27**

## Installation
* Elevate to root.
`sudo su -`
* Install _f2fs-tools_ and _pv_.
`apt-get f2fs-tools pv`
* Download the script.
`wget https://raw.githubusercontent.com/timothybrown/raspbian-f2fs/master/raspbian-f2fs`
* Make it executable.
`chmod +x raspbian-f2fs`

## Usage
The script *must* be run as root! (`sudo su -`)

`raspbian-f2fs source target`

Where _source_ is either:
* If the file extension is _zip_ we assume a compressed file containing the raw image of Raspbian/Raspbian Lite.
Exactly what you'd download from *www.RaspberryPi.org*.
Example: `2018-06-27-raspbian-stretch.zip`
* If the file extension is _img_ we assume a raw image of Raspbian/Raspbian Lite.
This is simply what the above zip file contains.
Example: `2018-06-27-raspbian-stretch.img`

And _target_ is either:
* A physical device, such as a USB SD card reader or flash drive. You can find the device path with the `lsblk` command.
If you're booted off an external USB device or network then the internal SD card reader is also a valid target.
Example: `/dev/sda`
* The path and filename of a raw image to create.
This can later be flashed to a device just like the original image.
Example: `/root/f2fs-rpi.img`

## How It Works
In the following list, items prefixed with **Device** apply to _physical device_ targets, like flash drives and SD cards; conversly items prefixed with **Image** only apply to _raw image_ targets.
1. We mount the source disk image to a loop device, then mount the source _root_ and _boot_ partitions.
2. First we check to see if the target is a device (like a flash drive) or a raw image.
   * **Device:** We wipe the partition table.
   * **Image:** We create a new image sized 200MiB larger than the source image and mount it onto a loop device. This allows us to treat our image as a device.
3. We partition the target device as follows:
   * MBR Partition Map
   * P1: 64MiB _FAT32(LBA)_
   * P2: Remaining Disk Space _Linux filesystem_
4. We mount the target boot partition and perform the following actions:
   * Format it as _FAT32_ and mount it.
   * Copy the boot files from _source_ to _target_.
   * Modify the target `/boot/cmdline.txt` file and change the _rootfstype_ and _root=PARTUUID_ lines to match our new configuration.
   * **Device:** Modify `boot/cmdline.txt` to remove *init=/usr/lib/raspi-config/init_resize.sh* which prevents the partition resize script from running on first boot.
5. We mount the target root partition and perform the following actions:
   * Format it as _F2FS_ and mount it.
   * Copy the root files from _source_ to _target_.
   * Modify the target `/etc/fstab` with the new _boot_ and _root_ UUIDs.
   * **Image:** Create _hook_ and _boot_ scripts to include `resize.f2fs` in our custom initramfs.
   * **Image:** Modify `/usr/lib/raspi-config/init_resize.sh` to boot into our custom initramfs after expanding the target partition.
   * **Image:** Generate a cleanup script to run after the partition and filesystem have been expanded.
6. Chroot into the target and:
   * Disable the automatic _ext4_ resize script `resize2fs_once`.
   * Install the _f2fs-tools_ package.
   * **Image:** Generate a custom initramfs to run `resize.f2fs` on first boot.
7. Sync, unmount and we're done!

## Target Image: Expanding an F2FS Root Partition on First Boot
The default image of Raspbian uses a two-step process to automatically expand the root partition to fill your entire SD card on the first boot.
1. The line *init=* in `boot/cmdline.txt` calls `/usr/lib/raspi-config/init_resize.sh` early in the boot process. This script uses `fdisk` to expand the partition to fill the entire SD card, removes the *init=* line from `cmdline.txt` then reboots.
2. *Systemd* loads an old-style *rc.d* entry which calls a script named `resize2fs_once`. This script runs `resize2fs`, which expands the ext4 filesystem to fill the entire partition. Afterwards the *rc.d* entry is removed and the system continues booting.

They're able to do this because resize2fs can work on a mounted volume. Unfortunately the F2FS resize tool (`resize.f2fs`) **requires** the volume to be unmounted to work. Because of the way *systemd* works it's very, very difficult to actively unmount the root partition once it's running. So, we have to find a way to run `resize.f2fs` *after* the kernel loads, but *before* the root filesystem is mounted. It is possible to create a *systemd* service that can run before filesystems are mounted, however then we'd have no way to run `resize.f2fs`, which resides on the root partition!

Fortunately, there's an easy way to do this. It's called *initramfs*, or the *Initial RAM Filesystem*. Basically, an *initramfs* is a compressed file that contains various libraries, scripts, binaries, kernel modules and a limited shell (*busybox*). Once loaded, the *initramfs* should provide a fully working, minimally capable system. So, what we do is create a custom *initramfs* with:
* The `resize.f2fs` tool along with the required libraries.
* A *boot script* that runs as part of the *init-premount hook*. This script runs `resize.f2fs` before the root filesystem is mounted.

This is how we put it all together:
1. `/usr/lib/raspi-config/init_resize.sh` is modified so that it adds *initramfs initrd.img followkernel* to `/boot/config.txt` after it performs the partition expansion.
2. After partition expansion the system will reboot and load our custom *initramfs*.
3. Our custom *initramfs* runs `resize.f2fs` on the root filesystem before it's mounted, which expands the filesystem. Afterwards the system continues the normal boot process.
4. Towards the end of the boot process *systemd* executes `/etc/rc.local`, which in turn runs `/etc/resizef2fs_cleanup`.
5. `/etc/resizef2fs_cleanup`:
   * Restores the original `../../../init_resize.sh` file.
   * Deletes our custom *initramfs*.
   * Restores the original `../rc.local` file.
   * Removes the *initramfs* line from `/boot/config.txt`.
   * Deletes itself.

Then we're left with a *F2FS* filesystem that's been expanded to fill the entire SD card, just like the default image of Raspbian does to *ext4* filesystems.
