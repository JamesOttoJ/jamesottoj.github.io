---
layout: default
title: Task 5
parent: NSA Codebreakers 2024
nav_order: 6
---

# TASK 5
{: .no_toc}
- TOC
{:toc}

### Task Description
>  Great job finding out what the APT did with the LLM! GA was able to check their network logs and figure out which developer copy and pasted the malicious code; that developer works on a core library used in firmware for the U.S. Joint Cyber Tactical Vehicle (JCTV)! This is worse than we thought!
> 
> You ask GA if they can share the firmware, but they must work with their legal teams to release copies of it (even to the NSA). While you wait, you look back at the data recovered from the raid. You discover an additional drive that you haven’t yet examined, so you decide to go back and look to see if you can find anything interesting on it. Sure enough, you find an encrypted file system on it, maybe it contains something that will help!
> 
> Unfortunately, you need to find a way to decrypt it. You remember that Emiko joined the Cryptanalysis Development Program (CADP) and might have some experience with this type of thing. When you reach out, he's immediately interested! He tells you that while the cryptography is usually solid, the implementation can often have flaws. Together you start hunting for something that will give you access to the filesystem.
> 
> What is the password to decrypt the filesystem? 
> 
> Prompt:
> - Enter the password (hope it works!)

### Files Given
- disk image of the USB drive which contains the encrypted filesystem (disk.dd.tar.gz)
- Interesting files from the user's directory (files.zip)
- Interesting files from the bin/ directory (bins.zip)

### Looking Through the Files
There are a lot more files this time around. Starting with the USB image, `file` shows: `disk.dd: DOS/MBR boot sector, code offset 0x58+2, OEM-ID "mkfs.fat", sectors/cluster 64, reserved sectors 64, Media descriptor 0xf8, sectors/track 63, heads 255, sectors 268435440 (volumes > 32 MB), FAT (32 bit), sectors/FAT 32768, serial number 0x6d316b65, label: "USB-128    "`. This tells us that it uses the fat32 file system, so we can mount it to our own system with:
```bash
sudo mkdir /mnt/USB-128
sudo losetup /dev/loop8 disk.dd
sudo mount -t vfat /dev/loop8 /mnt/USB-128/
```
This starts by setting up a loopback device that lets the OS access a file like a device. This is similar to how localhost works for networking where requests can be routed back to you without reaching an external service. This is needed because mount only works with devices. Normally, you should just be able to mount from a file, and it will set up the loop device automatically, but that wasn't working for me, so I did it manually. With this, I ran `ls -alR /mnt/USB-128` to get the following result:
```bash
┌─[jamesj@parrot]─[~/Documents/codebreaker_2024/task5]
└──╼ $ls -alR /mnt/USB-128/
/mnt/USB-128/:
total 192
drwxr-xr-x 5 root root 32768 Dec 31  1969 .
drwxr-xr-x 1 root root    14 Dec 31 09:36 ..
drwxr-xr-x 2 root root 32768 Aug  1  2024 .bin
drwxr-xr-x 2 root root 32768 Aug  1  2024 .data
drwxr-xr-x 2 root root 32768 Aug  1  2024 data
-rwxr-xr-x 1 root root    76 Aug  1  2024 lock
-rwxr-xr-x 1 root root    91 Aug  1  2024 unlock

/mnt/USB-128/.bin:
total 6080
drwxr-xr-x 2 root root   32768 Aug  1  2024 .
drwxr-xr-x 5 root root   32768 Dec 31  1969 ..
-rwxr-xr-x 1 root root 6132879 Aug  1  2024 gocryptfs

/mnt/USB-128/.data:
total 151840
drwxr-xr-x 2 root root    32768 Aug  1  2024 .
drwxr-xr-x 5 root root    32768 Dec 31  1969 ..
-rwxr-xr-x 1 root root 59289586 Aug  1  2024 2YvrwHWTn1671UdBF_1DPQ
-rwxr-xr-x 1 root root      398 Aug  1  2024 gocryptfs.conf
-rwxr-xr-x 1 root root       16 Aug  1  2024 gocryptfs.diriv
-rwxr-xr-x 1 root root 95998097 Aug  1  2024 OMvLJ8JjUeUimB7pp3c5Zg
-rwxr-xr-x 1 root root      275 Aug  1  2024 wikrN3RDYabCH-knjkLcNA

/mnt/USB-128/data:
total 64
drwxr-xr-x 2 root root 32768 Aug  1  2024 .
drwxr-xr-x 5 root root 32768 Dec 31  1969 ..
```

It appears that the system used to encrypt the file system is gocryptfs, and there are a few extra files to interact with it. There's the config and IV used to encrypt the files as well as a program to lock and unlock the files

lock:
```bash
#!/bin/bash
cd "$(dirname "$0")"
sleep 1
sync
fusermount -u ./data
sleep 1
```

unlock:
```bash
#!/bin/bash
cd "$(dirname "$0")"
sleep 1
exec ./.bin/gocryptfs "$@" -i 60s ./.data ./data
```

Moving on to the `files` folder, running `ls -alR` after unzipping it gives:
```bash
┌─[jamesj@parrot]─[~/Documents/codebreaker_2024/task5]
└──╼ $ls -alR files
files:
total 0
drwxr-xr-x 1 jamesj jamesj  44 Dec  4 09:33 .
drwxr-xr-x 1 jamesj jamesj 932 Jan 10 06:24 ..
drwxr-xr-x 1 jamesj jamesj 248 Aug  7 19:59 .keys
drwxr-xr-x 1 jamesj jamesj  64 Aug 15 10:53 .passwords
drwxr-xr-x 1 jamesj jamesj   8 Aug 12 18:22 .purple

files/.keys:
total 24
drwxr-xr-x 1 jamesj jamesj  248 Aug  7 19:59 .
drwxr-xr-x 1 jamesj jamesj   44 Dec  4 09:33 ..
-rw-r--r-- 1 jamesj jamesj  426 Aug  7 19:24 4C1D_public_key.pem
-rw-r--r-- 1 jamesj jamesj 1766 Aug  7 19:08 570RM_private_key.pem
-rw-r--r-- 1 jamesj jamesj  426 Aug  7 19:08 570RM_public_key.pem
-rw-r--r-- 1 jamesj jamesj  426 Aug  7 19:55 B055M4N_public_key.pem
-rw-r--r-- 1 jamesj jamesj  426 Aug  7 19:59 PL46U3_public_key.pem
-rw-r--r-- 1 jamesj jamesj  426 Aug  7 19:33 V3RM1N_public_key.pem

files/.passwords:
total 0
drwxr-xr-x 1 jamesj jamesj  64 Aug 15 10:53 .
drwxr-xr-x 1 jamesj jamesj  44 Dec  4 09:33 ..
drwxr-xr-x 1 jamesj jamesj 400 Aug 15 10:53 6def74e1ef65ed9bef8cc11bdf7fe9e9

files/.passwords/6def74e1ef65ed9bef8cc11bdf7fe9e9:
total 112
drwxr-xr-x 1 jamesj jamesj 400 Aug 15 10:53 .
drwxr-xr-x 1 jamesj jamesj  64 Aug 15 10:53 ..
-rw-r--r-- 1 jamesj jamesj  34 Aug 11 23:01 AmazonWebServices
-rw-r--r-- 1 jamesj jamesj  34 Aug 14 03:12 Apple
-rw-r--r-- 1 jamesj jamesj  34 Aug 14 23:27 Discord
-rw-r--r-- 1 jamesj jamesj  34 Aug 10 13:20 Facebook
-rw-r--r-- 1 jamesj jamesj  34 Aug 15 00:00 FileShare
-rw-r--r-- 1 jamesj jamesj  34 Aug 14 03:56 Github
-rw-r--r-- 1 jamesj jamesj  34 Aug 13 21:07 Google
-rw-r--r-- 1 jamesj jamesj  34 Aug  9 19:00 Hulu
-rw-r--r-- 1 jamesj jamesj  34 Aug 14 16:03 Instagram
-rw-r--r-- 1 jamesj jamesj  34 Aug 14 07:11 laptop
-rw-r--r-- 1 jamesj jamesj  34 Aug 14 14:23 LinkedIn
-rw-r--r-- 1 jamesj jamesj  34 Aug 11 06:02 Microsoft
-rw-r--r-- 1 jamesj jamesj  34 Aug 10 01:00 Netflix
-rw-r--r-- 1 jamesj jamesj  34 Aug 14 18:16 Peacock
-rw-r--r-- 1 jamesj jamesj  34 Aug  8 17:10 ProtonMail
-rw-r--r-- 1 jamesj jamesj  34 Aug  9 10:33 Reddit
-rw-r--r-- 1 jamesj jamesj  34 Aug  9 05:11 Samsung
-rw-r--r-- 1 jamesj jamesj  34 Aug 12 00:34 Skype
-rw-r--r-- 1 jamesj jamesj  34 Aug 15 08:48 Slack
-rw-r--r-- 1 jamesj jamesj  34 Aug 14 03:08 Snapchat
-rw-r--r-- 1 jamesj jamesj  34 Aug  8 18:14 Spotify
-rw-r--r-- 1 jamesj jamesj  34 Aug 12 13:23 Steam
-rw-r--r-- 1 jamesj jamesj  34 Aug 15 10:53 TikTok
-rw-r--r-- 1 jamesj jamesj  34 Aug 10 23:54 Twitter
-rw-r--r-- 1 jamesj jamesj  34 Aug 11 23:01 USB-128
-rw-r--r-- 1 jamesj jamesj  34 Aug 12 18:26 WhatsApp
-rw-r--r-- 1 jamesj jamesj  34 Aug  9 08:24 YouTube
-rw-r--r-- 1 jamesj jamesj  34 Aug 11 23:51 Zoom

files/.purple:
total 0
drwxr-xr-x 1 jamesj jamesj  8 Aug 12 18:22 .
drwxr-xr-x 1 jamesj jamesj 44 Dec  4 09:33 ..
drwxr-xr-x 1 jamesj jamesj 14 Aug 12 18:22 logs

files/.purple/logs:
total 0
drwxr-xr-x 1 jamesj jamesj 14 Aug 12 18:22 .
drwxr-xr-x 1 jamesj jamesj  8 Aug 12 18:22 ..
drwxr-xr-x 1 jamesj jamesj 10 Aug 12 18:22 bonjour

files/.purple/logs/bonjour:
total 0
drwxr-xr-x 1 jamesj jamesj 10 Aug 12 18:22 .
drwxr-xr-x 1 jamesj jamesj 14 Aug 12 18:22 ..
drwxr-xr-x 1 jamesj jamesj 46 Aug 12 18:22 570RM

files/.purple/logs/bonjour/570RM:
total 0
drwxr-xr-x 1 jamesj jamesj  46 Aug 12 18:22 .
drwxr-xr-x 1 jamesj jamesj  10 Aug 12 18:22 ..
drwxr-xr-x 1 jamesj jamesj  60 Aug 12 18:22 4C1D
drwxr-xr-x 1 jamesj jamesj  60 Aug 11 22:53 B055M4N
drwxr-xr-x 1 jamesj jamesj 120 Aug 12 18:22 PL46U3
drwxr-xr-x 1 jamesj jamesj  60 Aug 12 18:22 V3RM1N

files/.purple/logs/bonjour/570RM/4C1D:
total 4
drwxr-xr-x 1 jamesj jamesj   60 Aug 12 18:22 .
drwxr-xr-x 1 jamesj jamesj   46 Aug 12 18:22 ..
-rw-r--r-- 1 jamesj jamesj 1020 Aug 12 18:22 2024-08-12.182227-0400EDT.html

files/.purple/logs/bonjour/570RM/B055M4N:
total 4
drwxr-xr-x 1 jamesj jamesj   60 Aug 11 22:53 .
drwxr-xr-x 1 jamesj jamesj   46 Aug 12 18:22 ..
-rw-r--r-- 1 jamesj jamesj 2131 Aug 11 22:53 2024-08-11.225334-0400EDT.html

files/.purple/logs/bonjour/570RM/PL46U3:
total 8
drwxr-xr-x 1 jamesj jamesj  120 Aug 12 18:22 .
drwxr-xr-x 1 jamesj jamesj   46 Aug 12 18:22 ..
-rw-r--r-- 1 jamesj jamesj 2243 Aug 12 18:10 2024-08-12.181035-0400EDT.html
-rw-r--r-- 1 jamesj jamesj 1012 Aug 12 18:22 2024-08-12.182205-0400EDT.html

files/.purple/logs/bonjour/570RM/V3RM1N:
total 4
drwxr-xr-x 1 jamesj jamesj   60 Aug 12 18:22 .
drwxr-xr-x 1 jamesj jamesj   46 Aug 12 18:22 ..
-rw-r--r-- 1 jamesj jamesj 1057 Aug 12 18:22 2024-08-12.182256-0400EDT.html
```

There seem to be three major sections to this folder. First, there appears to be a list of encrypted passwords under `.passwords/6def74e1ef65ed9bef8cc11bdf7fe9e9/`. Secondly, there are a lot of logs in a path following the format: `.purple/logs/bonjour/username1/username2` where username1 is always 570RM. Lastly, there's the `.keys` directory with public keys for every aforementioned username, and a password-protected private key for 570RM.

Looking at the content of the logs, they appear to be chat logs about updating passwords for the USB and AWS:
```html
    <h2>Chat with 4C1D at 6:22 PM on August 12, 2024</h2>
    <div class="chatlog">
      <div class="chat" style="color: blue;">
        <b>570RM:</b> Xe7bFwXKYIyAh5Cd9d0cvHuqfPvX9180fQI8/q/hKe+y+zndg4yaP63Iq8xZtm8qucChx7AS1s7k8GqG9ZuyWVL/VPo9vRmJInmb/pEaEHlhFW4skWKPpNvLCPmZ6mfLiDaQpymqTLsAGeVgmbnR+WMWqaf9D6pO/vEQi3Mq6jQHLHaEsXEgf4hGtgilUWtw5wdqp9zxMMHnaOG8d5iJYzgC5FqmCpF7/ZW8Rp87OPnq2CF3AZdCGPKZM40bY+7SFVjs5PibV8NzKqWQJ4eFsE7Hwl838Dqy7nuVN0lLxMkgQ95FHzukDnC9Gy9Mh+wDdxg6ciFzZku05Svj+rCJQQ==
      </div>
      <div class="chat" style="color: purple;">
        <b>4C1D:</b> Thanks, I’ve got it.
      </div>
      <div class="chat" style="color: blue;">
        <b>570RM:</b> No problem. Let me know if you need anything else.
      </div>
      <div class="chat" style="color: purple;">
        <b>4C1D:</b> Will do. Appreciate it!
      </div>
    </div>
```

```html
<h2>Chat with B055M4N at 10:53 PM on August 11, 2024</h2>
    <div class="chatlog">
      <div class="chat" style="color: blue;">
        <b>570RM:</b> Got the USB drive you left for me. What’s on it?
      </div>
      <div class="chat" style="color: red;">
        <b>B055M4N:</b> The production build of the location service components that need to be deployed to the cloud. I’ll send you the password for it in a sec.
      </div>
      <div class="chat" style="color: blue;">
        <b>570RM:</b> Sounds good. I’ll get it set up once I have the password.
      </div>
      <div class="chat" style="color: red;">
        <b>B055M4N:</b> Cool. Also, heads up—the AWS password is about to expire. You should probably update it soon to avoid any issues.
      </div>
      <div class="chat" style="color: blue;">
        <b>570RM:</b> Thanks for the reminder. I’ll take care of that after I’m done with this.
      </div>
      <div class="chat" style="color: red;">
        <b>B055M4N:</b> No problem. I’m sending the USB drive password now.
      </div>
      <div class="chat" style="color: red;">
        <b>B055M4N:</b> dR6UPSE09Z9lRllcmBZWprmm0LFzjlIBmUq6MuLzIjOZWUmIaMuVHFs3BP9MwmLmbPWIpU7hlW6axPYu5SXt9x2fsYvWH8rz7fnJjea4XTruUC3Fp294daKONPF5g/8B9k6mQFQatQzXzMYvz2hd6pO05uDbKI7BUIMNDv+99sKwch09IINNPcwx14spGlBaU+9qPULm0Enqx559Ek7PmUNB20etckX/0yl2HXfEbcPbpw0HLcEzCqyZQ54ug3RSFfAbVbCsTCmmjh/cRV080CU4MZ2Q5YRsEMsljv3t3uKrMRJObqNgjJPD8twB/HMuQgLbg4kNkMJE8yRVgiHhXA==
      </div>
      <div class="chat" style="color: blue;">
        <b>570RM:</b> Got it. I’ll put the updated AWS password and the USB password into my password manager to keep everything secure.
      </div>
      <div class="chat" style="color: red;">
        <b>B055M4N:</b> Good call. Better to keep everything in one place.
      </div>
      <div class="chat" style="color: blue;">
        <b>570RM:</b> Exactly. Thanks for the heads up on everything.
      </div>
    </div>
```

```html
<h2>Chat with PL46U3 at 6:10 PM on August 12, 2024</h2>
    <div class="chatlog">
      <div class="chat" style="color: blue;">
        <b>570RM:</b> Have you had a chance to look at the latest changes to the obfuscation module?
      </div>
      <div class="chat" style="color: green;">
        <b>PL46U3:</b> Yeah, I think it should be effective for avoiding detection, but we might need to test it under different scenarios.
      </div>
      <div class="chat" style="color: blue;">
        <b>570RM:</b> Definitely. Also, we should make sure the payload is still clean after the new encryption layers are applied.
      </div>
      <div class="chat" style="color: green;">
        <b>PL46U3:</b> Good point. We can run it through the usual sandbox environments to be sure. Any updates on the server-side evasion techniques?
      </div>
      <div class="chat" style="color: blue;">
        <b>570RM:</b> I'm refining the code. We might need to add another layer of redirection to throw off any traffic analysis.
      </div>
      <div class="chat" style="color: green;">
        <b>PL46U3:</b> Makes sense. Better safe than sorry. By the way, I tried accessing the AWS account to spin up a new instance for our testing environment, but the old password doesn't work anymore.
      </div>
      <div class="chat" style="color: blue;">
        <b>570RM:</b> Oh, right. I recently updated the password. I'll send the new one to you in a moment.
      </div>
      <div class="chat" style="color: green;">
        <b>PL46U3:</b> Great, thanks. I need it ASAP to keep things moving.
      </div>
      <div class="chat" style="color: blue;">
        <b>570RM:</b> No problem. I'll send the password to you, V3RM1N, and 4C1D. But we'll use the custom encryption protocol we discussed earlier. Can't be too careful.
      </div>
      <div class="chat" style="color: green;">
        <b>PL46U3:</b> Understood. I’ll be ready on my end.
      </div>
      <div class="chat" style="color: blue;">
        <b>570RM:</b> Perfect. I'll have it to you shortly.
      </div>
    </div>
```

```html
<h2>Chat with PL46U3 at 6:22 PM on August 12, 2024</h2>
    <div class="chatlog">
      <div class="chat" style="color: blue;">
        <b>570RM:</b> VqpzVBxjmQ5/+trKindpobyE+Z1arWOMxSn8Njl5hBMX0OJ+5neh5yvN9MCE4kb/qEGzlYOjVuRX9oG/Mzv3xpp9lOk8kz8Ds8sAMWQ9Bs1qnUipT1LMBRd50uDhAXwysEtY+J3dP74uEeWnuKfgx1yUi378rheOCBwoTluN+ytRLrbi9Tzfb02gpuXQRTVB/SPRWbhZ7oLdZTxaoAhqBipvUnKcOwbkXlmQcac8kio2271MLlO9b+QeT8Tp7tLAj18sPt2N8Vs8VWkT1dLzE2MhUF7PON3wEH85qlj7b3cFNPm2rG1U0in8NoPdrRWbM7SucKiKHSeZ7Tum/JjE9g==
      </div>
      <div class="chat" style="color: green;">
        <b>PL46U3:</b> Got it!  Putting it into my password manager now.
      </div>
      <div class="chat" style="color: blue;">
        <b>570RM:</b> Have a good night!
      </div>
      <div class="chat" style="color: green;">
        <b>PL46U3:</b> Thanks, you too!
      </div>
    </div>
```

```html
<h2>Chat with V3RM1N at 6:22 PM on August 12, 2024</h2>
    <div class="chatlog">
      <div class="chat" style="color: blue;">
        <b>570RM:</b> TrdkBHkRLGxxNJLAOSeJiDq0Alyr9EoXc2FnxZjDpgJLfkPjCSU/Mu2ub6BerVRMISMDBMTG0d0PiA2ZSwwAHtWTetPfKl9+J21ZHrNMWt6Qjmtgna3Y0BpM2OxClWzwcejbiiOstmbMSuU1LbHUglRmCoMr33WOvjXDVK3mDHwIHiLCGCnStRDko4Id/QjdTn39JQ88aEGv1ttnOCGwjxU2pCQWSAhSuc9oGkgxuYQiKCrz2q082zoV8AUCb6x+i8niyuky6QlHMtzCS34y/SYJ11Eaa3o9aETO3cZb/+bTQTMbPI5NKSkAkaFJNT8tOcu64F3oTg2kAfvpubUZwQ==
      </div>
      <div class="chat" style="color: green;">
        <b>V3RM1N:</b> Excellent, got it. Appreciate it.
      </div>
      <div class="chat" style="color: blue;">
        <b>570RM:</b> No worries. Let me know if there’s anything else you need.
      </div>
      <div class="chat" style="color: green;">
        <b>V3RM1N:</b> Will do, thanks again for the help.
      </div>
    </div>
```

Sadly, the passwords seem to have been encrypted and encoded with base64. This does, however, give us 3 different encrypted versions of the password: gocryptfs, password store, and chat. This means that vulnerabilities in the cryptography that even give us partial information could help when putting them together.

The last folder is the `bins` folder. This folder just has two programs in it: `pm` and `pidgin_rsa_encryption`

### Reverse Engineering
Starting with `file` on the two programs:
```bash
┌─[jamesj@parrot]─[~/Documents/codebreaker_2024/task5]
└──╼ $file bins/pm 
bins/pm: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=0230dd29d2f5ba42b3274ff7981105c752577832, for GNU/Linux 2.6.32, stripped
┌─[jamesj@parrot]─[~/Documents/codebreaker_2024/task5]
└──╼ $file bins/pidgin_rsa_encryption 
bins/pidgin_rsa_encryption: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=0230dd29d2f5ba42b3274ff7981105c752577832, for GNU/Linux 2.6.32, stripped
```

Right off the bat, they look like normal linux executables, so I was hoping reverse engineering was going to be a little easier. Moving to running strings on the files, there didn't appear to be and strings related to how the program worked like errors or prompts. There were, however, a lot of library names, and quite a few referenced python. I took that to mean that these were either compiled python files or used python in some way.

Once I opened them up in Ghidra, I realized that I would need to do some searching. The function called by `_libc_start_main()` appeared to do a lot of python initialization things, and there wasn't a clear main function start place. Looking into it, I learned that the `pydata` section of the executable holds all of the python code, but it's all python bytecode handled by the python interpreter. After learning this, I looked more into how I could get source code back.

I found that I would need to extract the `pydata` section to get python bytecode. After that, it could be broken up with a tool called [pyinstxtractor](https://github.com/extremecoders-re/pyinstxtractor) to get the pyc files. Those pyc files can then disassembled and decompiled. I tried the recommended tools from pyinstxtractor, but uncompyle wasn't working for me, so I used [Decompile++](https://github.com/zrax/pycdc) instead. I will warn that this still didn't fully work (it didn't support some of the instructions for decompilation), but it gave enough in the code output that I could understand what was going on and fill in the gaps with the assembly file. I will also warn against using pylingual.io because it gave the wrong output even if it "supported" everything. This is in part because it would guess sometimes.

### Cryptographic Analysis
Now that we can see what everything is and what it does, it's time to identify the vulnerabilities. As the scenario states, "while the cryptography is usually solid, the implementation can often have flaws." This told me that I would need to look closely at how each of the programs worked as well as what flaws there can be in the protocols they use

#### pm
Starting with `pm`, this is the python output I got:
```py
# Source Generated with Decompyle++
# File: pm.pyc (Python 3.11)

import os
import sys
import base64
from getpass import getpass
import hashlib
import time
import string
import random
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC
from cryptography.hazmat.backends import default_backend
from cryptography.hazmat.primitives import hashes
SALT = b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'

def derive_key(password = None):
    kdf = PBKDF2HMAC(algorithm = hashes.SHA256(), length = 32, salt = SALT, iterations = 100000, backend = default_backend())
    return kdf.derive(password.encode())


def generate_password(length = None):
    character_list = string.ascii_letters + string.digits + string.punctuation
    password = []
# WARNING: Decompyle incomplete


def encrypt_password(spassword = None, password = None):
    key = derive_key(password)
    ts = str(int(time.time() / 60)).encode('utf-8')
    iv = hashlib.md5(ts).digest()
    cipher = Cipher(algorithms.AES(key), modes.CFB(iv), backend = default_backend())
    encryptor = cipher.encryptor()
    encrypted_password = encryptor.update(spassword.encode()) + encryptor.finalize()
    return iv + encrypted_password


def decrypt_password(encrypted_data = None, password = None):
    key = derive_key(password)
    iv = encrypted_data[:16]
    encrypted_password = encrypted_data[16:]
    cipher = Cipher(algorithms.AES(key), modes.CFB(iv), backend = default_backend())
    decryptor = cipher.decryptor()
    decrypted_password = decryptor.update(encrypted_password) + decryptor.finalize()
    return decrypted_password.decode()


def save_password(filename = None, password = None, spassword = None):
    encrypted_password = encrypt_password(spassword, password)
# WARNING: Decompyle incomplete


def load_password(filename = None, password = None):
    pass
# WARNING: Decompyle incomplete


def usage():
    print('Usage: pm.py <command>')
    print('Commands:')
    print('  init   - Create a new master password')
    print('  add    - Add a new password')
    print('  gen    - Generate a new password')
    print('  read   - Retrieve a password')
    print('  help   - Print this help file')


def main():
    if len(sys.argv) != 2:
        usage()
        sys.exit(1)
    command = sys.argv[1]
    if command == 'init':
        homedir = os.path.expanduser('~')
        passdir = homedir + '/.passwords'
        if not os.path.isdir(passdir):
            os.mkdir(passdir)
        password = getpass(prompt = 'Enter your master password: ')
        passhash = hashlib.md5(password.encode('utf-8')).hexdigest()
        dirname = passdir + '/' + passhash
        if not os.path.isdir(dirname):
            os.mkdir(dirname)
            return None
        None('directory already exists for that master password')
        return None
    if None == 'add':
        password = getpass(prompt = 'Enter your master password: ')
        passhash = hashlib.md5(password.encode('utf-8')).hexdigest()
        dirname = os.path.expanduser('~') + '/.passwords/' + passhash
        if not os.path.isdir(dirname):
            print('Unknown master password, please init first')
            return None
        service = None('Enter the service name:  ')
        filename = dirname + '/' + service
        if os.path.isfile(filename):
            print('A password was already stored for that service.')
            return None
        spassword = None(f'''Enter the password to store for {service}:  ''')
        save_password(filename, password, spassword)
        return None
    if None == 'read':
        password = getpass(prompt = 'Enter your master password: ')
        passhash = hashlib.md5(password.encode('utf-8')).hexdigest()
        dirname = os.path.expanduser('~') + '/.passwords/' + passhash
        if not os.path.isdir(dirname):
            print('Unknown master password')
            return None
        service = None('Enter the service name:  ')
        filename = dirname + '/' + service
        if not os.path.isfile(filename):
            print('No password stored for that service using that master password')
            return None
        spassword = None(filename, password)
        print(f'''Password for {service}: {spassword}''')
        return None
    if None == 'gen':
        password = getpass(prompt = 'Enter your master password: ')
        passhash = hashlib.md5(password.encode('utf-8')).hexdigest()
        dirname = os.path.expanduser('~') + '/.passwords/' + passhash
        if not os.path.isdir(dirname):
            print('Unknown master password, please init first')
            return None
        service = None('Enter the service name:  ')
        filename = dirname + '/' + service
        if os.path.isfile(filename):
            print('A password was already stored for that service.')
            return None
        if not input('Enter the password length (default 18):  '):
            pass_len = None('18')
            spassword = generate_password(pass_len)
            save_password(filename, password, spassword)
            return None
        if input('Enter the password length (default 18):  ') == 'help':
            usage()
            return None
        None('Unknown command')
        return None

if __name__ == '__main__':
    main()
    return None

```

Looking at this code, these are the main cryptographic flaws I could identify (from top to bottom):
| Flaw | Importance |
| --- | --- |
| 1. cryptography.hazmat library is being used | Leaves room for implementation flaws like the scenario talks about because the programmer controls every part of the cipher usage instead of relying on a secure default |
| 2. Consistent salt | Prevents randomness and allows an attacker to identify when the same information is encrypted or even repeat the result if it gets used somewhere |
| 3. PKDF low number of iterations | Makes the result more brute-forceable |
| 4. IV for encrypting the password uses minute resolution for the timestamp | Storing two passwords within the same minute (very possible) will have the same IV and not be as random |
| 5. CFB is used for AES cipher block mode | This has a lot of complex problems, so it will be analyzed after this table |
| 6. MD5 used to store the master password | MD5 is very brute-forceable (my laptop can check over 4 million per second). It is more likely to produce a hash colisions (two inputs that produce the same hash output) |

#### AES-CFB
*How it works*: Starting with the first block, the key is used at the key, and the IV is used at the input for AES. This produces one block of bytes that gets xored with the first block of plaintext to create the first block of ciphertext. That is then used as the next input for AES to produce another block of bytes. Those bytes are xored with the next block of plaintext to produce the second block of ciphertext. This process is repeated until there is no plaintext left.
![AES Block Mode Chart](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#/media/File:BlockCipherModesofOperation.svg "AES Block Mode Chart")

*Leaked information*: One unique feature of CFB is that it can stop at the exact length of the plaintext. For This case, each block is 16 bytes (128 bits). Normally, this means that the output size will always be a multiple of 16 (16, 32, 48, 64, ...) by the nature of block encryption. Looking at the password files, however, they are all 34 bytes. The first 16 bytes of that are for the IV which leaves 18 bytes of ciphertext. This is beacuse CFB can work in two ways. The first way is to throw out the extra bytes after the plaintext is done being xored (how it works here). It can also pad the plaintext to the size of a block with null bytes (better because it releases less information).

Of note, the number 18 should spark recognition. It's the default length for the password generation portion. Because all of the passwords here are 18 bytes, I would bet that all of these are randomly generated.

*Reused IV*: With this implementation, it is very important to have a different IV, key, or both. If the same IV and key are used, we run into the One-Time Pad (OTP) reuse issue. This stems from the idea that `A ^ B == C && A ^ C == B && B ^ C == A && C ^ B == A`, so we can get `Plaintext1 ^ Plaintext2` if we have two ciphertexts encrypted with the same pad.

#### pidgin_rsa_encryption
Moving to `pidgin_rsa_encryption`, Decompyle++ had an especially hard time with this one, so I had to reference the assembly to double check my assumptions. Luckily, all of the important information was still in the decompiled code:
```py
# Source Generated with Decompyle++
# File: pidgin_rsa_encryption.pyc (Python 3.11)

import sys
import math
import base64
import random
from Crypto.PublicKey import RSA
from rsa import core

def load_public_key(pub_key):
Unsupported opcode: BEFORE_WITH (108)
    pass
# WARNING: Decompyle incomplete


def load_private_key(password, priv_key):
Unsupported opcode: BEFORE_WITH (108)
    pass
# WARNING: Decompyle incomplete


def encrypt_chunk(chunk, public_key):
    k = math.ceil(public_key.n.bit_length() / 8)
    pad_len = k - len(chunk)
    random.seed(a = 'None')
    padding = (lambda .0: for i in .0:
pass[ random.randrange(1, 255) ])(range(pad_len - 3)())
    padding = b'\x00\x02' + padding + b'\x00'
    padded_chunk = padding + chunk.encode()
    input_nr = int.from_bytes(padded_chunk, byteorder = 'big')
    crypted_nr = core.encrypt_int(input_nr, public_key.e, public_key.n)
    encrypted_chunk = crypted_nr.to_bytes(k, byteorder = 'big')
    return base64.b64encode(encrypted_chunk).decode()


def decrypt_chunk(encrypted_chunk, private_key):
Segmentation fault
```

To get an idea of how the program is actually supposed to work, I also just ran it and checked the output
```bash
Usage: python pidgin_rsa_encryption.py <mode> [<recipient> <message> <public_key> | <encrypted_message> <password>]
Modes:
  send <recipient> <message> <public_key> - Send an encrypted message
  receive <encrypted_message> <password> <private_key> - Decrypt the given encrypted message
```

With this information, I can assume that this program made the encrypted messages from the chat logs. The main thing to note here is that a static pad is used. It's tricky to spot (and I wasn't too confident in the decompiler), but `random.seed(a='None')` is **very** different from `random.seed(a=None)`. The first one is a string while the second one is a keyword. The string sets the seed to the bytes for 'None' while the keyword, None, makes python pick a source of randomnes like `/dev/urandom` or the epoch time in miliseconds. This means that (just like in task 3) the same data will be generated when a random value is needed instead of unique data each run. In this case, that means the padding will be the same for every message.

#### Trying the Obvious
To start, I tried the quick and easy things just in case. I used hashcat with rockyou against both the MD5 of the master password and the hash for the private key with no results. I also looked into brute forcing gocryptfs with rockyou, but it took multiple seconds for each attempt (as intended), so it wasn't feasable. I then reimplented the padding scheme to run through rockyou using 570RM's public key and checking it against the message from B055M4N. While doing that, I noted that all of the RSA keys used an e of 3 which can lead to a variety of low exponent attacks because RSA is just `math.pow(plaintext, e) % n`.

### Exploiting Vulnerabilities
With all that in mind I started to put the following together:
1. RSA low exponent attack
2. AES-CFB OTP reuse
3. Available characters identified in `pm`

While I identified the low exponent attack quite early, I struggled to find an implementation. Eventually, I found a [writeup by xanhacks](https://xanhacks.gitlab.io/ctf-docs/crypto/rsa/08-hastad-broadcast-attack/) with an implementation for the broadcast attack that worked. This attack requires e or more of the same message to be sent using multiple ns with no common denominator (n is the result of multiplying two large primes, so two ns can happen to have one prime in common). We get this with the messages to 4C1D, PL4GU3, and V3RM1N letting them know what the new AWS key is. I assumed it would be the same copy/pasted message based on the timestamps, and each message would be sent with their respective public key. Running the script gives:
```bash
┌─[jamesj@parrot]─[~/Documents/codebreaker_2024/task5]
└──╼ $./venv/bin/python3 broadcast.py 
Unable to find the third root of : 11857946556585826766083506285492510038986961474833404528141798644049830639645553611088386021313223328092966097754795722079472921195356921443783835045787270682027603267169543891101265111212066475848740782926591100901037843550195750786539165017916073090657122626019472280107870357061300220302716954475256463168122627948952693964368034417845334665711499847523569028428058636440794765572250401518896120410736848523299061147394208748275531852109205515422178137746925796383314820215232255794807118408374468315830298333330170778430296905113030470748831005643938781648157128049007317178841563218569974244865937111288520935745
Unable to find the third root of : 10940546502377398034481726944118221927069493730353656213060346100197995842147252684230277484174880068519678914993022768318258766372207477071250161341194033364109575909794582391229144943003952181000505828796388018029295177261426284281023914160011884094321037945398840676671191954355528582862010578487079082502012766817190652587239929539186687067492135203043256058068181232899308773696834068890373318618905880278185214383496175109930236851382035941312795263986873331452134181693368201520825234780578083347657568540862694626101868329362059301862279607347441083868037254365793674860786663414111851420939752599072510887158
Unable to find the third root of : 9937021108690840654210730181575885240597977974288733461678120992746393564645333872289629007183688276347391985018985701005120605609317697356667509598313299565234216789841043650549564628414639645332341121836772039414359961913753681719743048931153161982054559197676363227805213635839153636588195109603435399830167869383633999949470396148129100571605443915284526740213996172263559662117485582559261671325208269538285153191665262888736638992669981242517143569118202217516256090942329702871646541667639340831389040489065488268537235699983835608633078973187826663520048113075624698485929329315101480607440637708751183616449
Cleartext : b'\x02U\x13\xa7\xc5\x0b\r)\xa62_\xbf\x05\x93jq\xfbO~\xe9\xdf\xb5\xf9L\xfe\xa1/\xc3\x1f\x8b\xfd*gQ\xdd\xe5\t\xdd\x06\xa6\x1e\xd3\x98\xc1K\x98\\_\nb\x87\xdfa\xd4\xcb\x187x\xbb\xc0\x8d\xbfm\r)\x81>\xc8\x814\x989p\x8d\xb3\\\x01{\xfa0\x9e\nDU\xd7\x89\x9c\xac\xcd\x99O\nH:U\xa3k\x8a\xaa\xee\xf33/\x00Hey!  I needed to update the AWS password since it expired.  The new password is F*9ce67"=?C~~uo%Q4.  Please add it to your password managers.  Thanks!'
```

Woo! Now we have the AWS password, but we need the USB password, so how does that help? Well, B055M4N told 570RM to generate a new AWS password when they gave 570RM the USB password, so I figured 570RM may have stored the USB password and generated the AWS password in the same minute. Looking at the password file contents, this appears to be the case:
```
┌─[jamesj@parrot]─[~/Documents/codebreaker_2024/task5]
└──╼ $xxd files/.passwords/6def74e1ef65ed9bef8cc11bdf7fe9e9/AmazonWebServices 
00000000: 7c30 037e ec00 b2e1 f409 ea92 272d 1e80  |0.~........'-..
00000010: 3326 e908 30e2 1ed1 26c3 8665 ba01 245a  3&..0...&..e..$Z
00000020: 1b1a                                     ..
┌─[jamesj@parrot]─[~/Documents/codebreaker_2024/task5]
└──╼ $xxd files/.passwords/6def74e1ef65ed9bef8cc11bdf7fe9e9/USB-128 
00000000: 7c30 037e ec00 b2e1 f409 ea92 272d 1e80  |0.~........'-..
00000010: 4e36 a133 17b1 71cc 74b1 bb34 8721 3018  N6.3..q.t..4.!0.
00000020: 7099
```

This means that we can take the ciphertexts of the passwords and xor them to get the first block of the xored plaintexts. With the xored plaintext, we take the one we know and xor it with that. This gives the first block (16 bytes) of the USB password:
```
AWS_pass ^ OTP = AWS_pass_encrypted
USB_PASS ^ OTP = USB_pass_encrypted
AWS_pass_encrypted ^ USB_pass_encrypted = AWS_pass ^ OTP ^ USB_PASS ^ OTP = AWS_pass ^ USB_PASS ^ OTP ^ OTP = AWS_pass ^ USB_PASS
USB_pass ^ AWS_pass ^ AWS_pass = USB_pass

AWS: 7c30037eec00b2e1f409ea92272d1e803326e90830e21ed126c38665ba01245a1b1a
	IV: 7c30037eec00b2e1f409ea92272d1e80
	Ciphertext: 3326e90830e21ed126c38665ba01245a1b1a
USB: 7c30037eec00b2e1f409ea92272d1e804e36a13317b171cc74b1bb34872130187099
	IV: 7c30037eec00b2e1f409ea92272d1e80
	Ciphertext: 4e36a13317b171cc74b1bb34872130187099

XOR: 7d 10 48 3b 27 53 6f 1d 52 72 3d 51 3d 20 14 42 | 6b 83
AWS: F*9ce67"=?C~~uo%Q4
USB-128[:16]: ;:qXBeX?oM~/CU{g
```

With the first 16 bytes of the USB password, it was easy enough to try and bruteforce the last 2 bytes using the following script:
```py
import string
import subprocess

characters = string.ascii_letters + string.digits + string.punctuation
password_base = ";:qXBeX?oM~/CU{g"

for char1 in characters:
    for char2 in characters:
        password = password_base + char1 + char2
        print("password: " + password)
        out = subprocess.run(["/mnt/USB-128/unlock"], input=password.encode(), capture_output=True)
        #print(out.stderr)
        if "Password incorrect." not in str(out.stderr):
            exit()
        print()
```

That revealed the password: `;:qXBeX?oM~/CU{gPf`