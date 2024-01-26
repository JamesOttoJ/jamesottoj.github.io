---
layout: default
title: Task 4
parent: NSA Codebreakers 2023
nav_order: 5
---

# TASK 4
{: .no_toc}
- TOC
{:toc}

### Task Description
>We were able to extract the device firmware, however there isn't much visible on it. All the important software might be protected by another method.
>
>There is another disk on a USB device with an interesting file that looks to be an encrypted filesystem. Can you figure out how the system decrypts and mounts it? Recover the password used to decrypt it. You can emulate the device using the QEMU docker container from task 3.

### Files Given
sd.img.bz2
usb.img.bz2
kernel8.img.bz2
bcm2710-rpi-3-b-plus.dtb.bz2

### Digging for differences
After starting everything up with the readme, a basic linux shell pops up. I always start with running `ls -al` /home and /root, but both of those were empty this time. From there, I noticed an uncommon directory called "pivate". Inside is a couple of ssh keys and a file called id.txt that is empty. To see where the sd and usb devices ended up, I looked at the output of `mount` next.

```
/dev/mmcblk0p1 : Mounted to the (/) root directory and is on the SD card
/dev/mmcblk0p2 : Mounted to the /boot directory and is on the SD card
/dev/sda1 : 
	On the USB
	mounted to /opt
/dev/sda2 :
	On the usb
	mounted to /private
```

Knowing that the USB is on /opt, I looked in there and found some relevant files: hostname, mount_part, and part.enc. hostname was just a text file with the word "crazyfence" int it. [mount_part](./static/mount_part) was more interesting because it looked to take the part.enc file and decrypt it with cryptsetup. Because it uses the hostname and id.txt for the password, I tried to find what might have been in id.txt using tools like autopsy to no avail. After exhausting my options, I moved onto bruteforce

### Hashing and Catting
```shell
NAME=`hostname`
ID=`cat /private/id.txt` # id.txt is zeroed out

DATA="${NAME}${ID:0:3}"
echo "cryptsetup: opening $ENC_PARTITION"
echo -n $DATA | openssl sha1 | awk '{print $NF}' | cryptsetup open $ENC_PARTITION part
```