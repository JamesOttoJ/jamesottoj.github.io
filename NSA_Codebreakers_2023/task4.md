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

Knowing that the USB is on /opt, I looked in there and found some relevant files: hostname, mount_part, and part.enc. hostname was just a text file with the word "crazyfence" int it. [mount_part](./static/mount_part) was more interesting because it looked to take the part.enc file and decrypt it with cryptsetup. Because it uses the hostname and id.txt for the password, I tried to find what might have been in id.txt using tools like autopsy to no avail. After exhausting my options, I moved onto bruteforce.

### Hashing and Catting
```shell
NAME=`hostname`
ID=`cat /private/id.txt` # id.txt is zeroed out

DATA="${NAME}${ID:0:3}"
echo "cryptsetup: opening $ENC_PARTITION"
echo -n $DATA | openssl sha1 | awk '{print $NF}' | cryptsetup open $ENC_PARTITION part
```

Above is an excerpt from the mount_part file that we care about. It starts by setting a variable, NAME, to the hostname. Then, it sets the variable ID to whatever is in id.txt. After that, it puts them together into a plaintext password using the hostname followed by the first three characters of the id.txt file. Onto the complicated line, `echo -n $DATA` just takes the Data variable and outputs it *without a newline*. That data is sent to `openssl sha1` to hash it using the sha1 algorithm. Because openssl adds some header information and outputs in the format "SHA1(stdin)= 20ae5c97cd9c3ec19fc1d5e01829622c6313b6cb", `awk '{print $NF}` is used to print out only the last "line". Side note, this is great for parsing the output of things like ps. Aat last, the hash is piped into cryptsetup as a password to decrypt the partition and mount it to /dev/mapper/part.

From this, we can gather a couple of things about the password. First, the hostname given to us is "crazyfence". Second, only 3 characters from ID are added to the hostname. This gives us a password of crazyfenceXXX with the Xs replaces by a character or number. The easiest path I saw from here was to generate plaintext passwords to fit in the DATA variable, run them through the hashing function they have, and use them in a wordlist to crack the partition password.

Starting with generating the plaintext passwords, I turned to hashcat. The command I landed on was `hashcat --stdout -a 6 -1?l?u?d ./hostname.txt ?1?1?1 > base_passwords.txt`. For more in-depth information on each of the arguments, check the [man pages](https://manpages.org/hashcat) or [wiki](https://hashcat.net/wiki/doku.php?id=hashcat). Just looking at this command, `--stdout` redirects all of the possible passwords into standard out. `-a 6` specifies an attack mode with 6 meaning a wordlist followed by a mask (all of the characters in a specified range). `-1?l?u?d` creates a custom mask called 1 that is a collection of all lowercase(?l), uppercase(?u), and number(?d) characters. `./hostname.txt ?1?1?1` then gives the hostname followed by every combination of alphanumeric characters. Lastly, `> base_passwords.txt` sends all of the standard out data to the base_passwords.txt file

Now that I had all of the plaintext passwords, I created a quick script to creat a wordlist will all the hashed versions of them:
```bash
while read LINE; do
  echo -n "$LINE" | openssl sha1 | awk '{print $NF}' >> passwords.txt
done <base_passwords.txt
```
This just goes through each line of base_passwords.txt, passes them through the hashing function, and appends them to passwords.txt. Once the real wordlist is aquired, it's time to start cracking

Looking at how cryptsetup works as well as how to bruteforce it, I found that it uses luks, and most likely luksv1. With that information, I was able to make this hashcat command: `hashcat -m 14600 -a 0 -w 3 part.enc passwords.txt -o luks_password.txt`. `-m 14600` just specifies that we're cracking luksv1 encryption. `-a 0` says that it is just using a normal wordlist and trying each of the entries as passwords. `-w 3` just sets consumption to high for faster cracking. The rest are our encrypted partition, password list, and output file for the found password. Keep in mind that this password will be the hashed version, so you'll need to decode it to get the id or plaintext for mounting on startup