# raspbian-f2fs

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
`raspbian-f2fs source.zip|img /dev/target|target.img`

Where _source_ is either:
* A zip file containing a raw image of Raspbian/Raspbian Lite. Exactly what you'd
download from *RaspberryPi.org*. If you use this option then the zip file **must**
have the same name as the _.img_ file it contains (as it does by default).
* A raw image of Raspbian/Raspbian Lite. This is simply what the above zip file contains.
There's no constraints on the filename, as long as the extension remains _.img_.

And _target_ is either:
* A physical device, such as a USB SD card reader or flash drive (typically /dev/sdX).
If you're booted off an external USB device or network then the internal SD card
reader is also a valid target (/dev/mmcblk0).
* The path and filename of a raw image to create. This can later be flashed to a device
just like the original image.
