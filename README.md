# raspbian-f2fs

This script takes a .zip or .img file and either:
- Writes an F2FS installation directly to an SD card or USB flash device.
(The root partition is expanded to fill the device during write.)
- Creates a new F2FS image that can be flashed just like the original image.
(The root partition will be automatically expanded during first boot.)

In either case it boots it seems to boot and work well. I've been using F2FS in
several production Pi's for several months with no issues.

Syntax: raspbian-f2fs source.zip|img /dev/target|target.img
