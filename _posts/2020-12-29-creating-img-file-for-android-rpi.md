---
title: "Creating AOSP img for android-rpi"
layout: post
---

> Imagination will often carry us to worlds that never were. But without it we go nowhere. - Carl Sagan

[Android-rpi](https://github.com/android-rpi) is an awesome port of the [Android
Open Source Project](https://source.android.com/) for the Raspberry Pi. Its community is full of great 
developer and most of your doubts can be easily solved by looking at their 
[Google Forum group](https://groups.google.com/forum/m/#!forum/android-rpi).
However, there is no information on how to create a
distributable img file from your compiled operating system. That's the goal of
this tutorial, teach you one way of generating an img file that you can easily
distribute.

The first step we will take is, generate an empty img file. This can be easily accomplished using 
the dd command that comes with most GNU/Linux distribution, and I am sorry about not mentioning earlier
this is all done on Ubuntu 20.04 and should work on macOS.

For this example I will create a 6 GB image, this value needs to be larger than the minimum size for
the boot, system and vendor partitions that you will be using on your device. I recommend, 128 MB for
boot, 1024 MB for both system and vendor and the rest of the SD card for the data partition. Therefore,
our img has to be bigger than 2.2 GB. To add some slack I decided to go with 6 GB.

The dd command works with an input file (if), in our case zeros, and sends data information to an
output file (of) which will be our image. The block size will be given in Megabytes (bs) and we
guarantee that we pass a full 1 MB block (bs) from the input every time we write to the output file.

{% highlight bash %}
$ dd if=/dev/zero of=android.img iflag=fullblock bs=1M count=6144 && sync
{% endhighlight %}

After creating the image we will need to map it to a loop device using the losetup command. First of
all, make sure you use a name that is not in use, for that you can check the loop devices by running
losetup with no arguments.

{% highlight bash %}
$ losetup
{% endhighlight %}

Once you decide the number of the loop device map the img to the desired device, for this example
I will mount it at loop99.

{% highlight bash %}
$ sudo losetup loop99 android.img
{% endhighlight %}

Once the disk is properly loaded on the kernel - you can check with lsblk - you will need to create
the partitions on the loop device, to make it easier I will provide you my script. It automatically
creates and add the filesystem to these partitions.

{% highlight bash %}
#!/bin/bash

sudo parted -a optimal -s $1 -- mklabel msdos \
    mkpart primary fat32 1MiB 129MiB  \
    mkpart primary ext4 129MiB 1154MiB \
    mkpart primary ext4 1154MiB 2179MiB \
    mkpart primary ext4 2179MiB 100%

sudo mkfs.vfat -F 32 -n BOOT "$1p1"
sudo mkfs.ext4 -F -O ^64bit -L system "$1p2"
sudo mkfs.ext4 -F -O ^64bit -L vendor "$1p3"
sudo mkfs.ext4 -F -O ^64bit -L data "$1p4"

sudo parted -a optimal -s $1 -- set 1 boot on \
    set 1 lba on

{% endhighlight %}

The partitions are created using the parted command. For this particular device we will need to
label the device as msdos in order to add both the boot and lba flags to the boot partition. The
partitions are created with the mkpart option of the parted utility where we can descripe starting
and ending position, type and format. Once the partitions are written the mkfs.* suite can be used
to create the filesystem for every partition. To finish the creation on the disk we write the flags
to the boot partition using parted, this has to be done afterwards because the filesystem has to be
created in order to have the flags available.

To execute the script save the code above to a .sh file, add the execution permission using chmod and
run it as shown below. Remember to change the loop device to whatever you have used.

{% highlight bash %}
$ ./name_of_the_script.sh /dev/loop99
{% endhighlight %}

The next step required is to mount the partitions to actual directories on your machine. I recommend
that you create a folder named "mnt" on your home folder and inside it add the 4 partitions on the
disk + overlays. Like so:

{% highlight bash %}
$ mkdir ~/mnt
$ mkdir ~/mnt/boot
$ mkdir ~/mnt/boot/overlays
$ mkdir ~/mnt/system
$ mkdir ~/mnt/vendor
$ mkdir ~/mnt/data
{% endhighlight %}

With the directories created we can mount the partitions of the loopback device to the directories.

{% highlight bash %}
sudo mount "/dev/loop99p1" ~/mnt/boot
sudo mount "/dev/loop99p2" ~/mnt/system
sudo mount "/dev/loop99p3" ~/mnt/vendor
sudo mount "/dev/loop99p4" ~/mnt/data
{% endhighlight %}

Once mounted you can follow the default copy of the files as described by the folks at
android-rpi.

Write system & vendor partition:

{% highlight bash %}
$ cd out/target/product/rpi4
$ sudo dd if=system.img of=/dev/loop99p2 bs=1M
$ sudo dd if=vendor.img of=/dev/loop99p3 bs=1M
{% endhighlight %}
  
Copy kernel & ramdisk to BOOT partition:
{% highlight bash %}
$ cd out/target/product/rpi4
$ cp device/arpi/rpi4/boot/* ~/mnt/boot
$ cp kernel/arpi/arch/arm64/boot/Image.gz ~/mnt/boot
$ cp kernel/arpi/arch/arm64/boot/dts/broadcom/bcm2711-rpi-4-b.dtb ~/mnt/boot
$ cp kernel/arpi/arch/arm64/boot/dts/overlays/vc4-kms-v3d-pi4.dtbo ~/mnt/boot/overlays
$ cp out/target/product/rpi4/ramdisk.img ~/mnt/boot
{% endhighlight %}

Next up, unmount the loop device from the directories and remove the mnt folder if you no longer
neet to mount the img.

{% highlight bash %}
$ sudo umount ~/mnt/boot
$ sudo umount ~/mnt/system
$ sudo umount ~/mnt/vendor
$ sudo umount ~/mnt/data

#Below is optional
rm -rf ~/mnt
{% endhighlight %}

And to conclude unload the loopback device from the kernel.

{% highlight bash %}
sudo losetup -d /dev/loop99
{% endhighlight %}

Now you can burn the .img with whatever tool you like to use, I personally enjoy [Balena Etcher](https://www.balena.io/etcher/). Plug
the SD card on the Raspberry Pi and enjoy your custom img!

