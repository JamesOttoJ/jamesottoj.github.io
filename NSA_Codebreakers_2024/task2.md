---
layout: default
title: Task 2
parent: NSA Codebreakers 2024
nav_order: 3
---

# TASK 2
{: .no_toc}
- TOC
{:toc}

## Task Description
> Having contacted the NSA liaison at the FBI, you learn that a facility at this address is already on a FBI watchlist for suspected criminal activity.
> 
> With this tip, the FBI acquires a warrant and raids the location.
> 
> Inside they find the empty boxes of programmable OTP tokens, but the location appears to be abandoned. We're concerned about what this APT is up to! These hardware tokens are used to secure networks used by Defense Industrial Base companies that produce critical military hardware.
> 
> The FBI sends the NSA a cache of other equipment found at the site. It is quickly assigned to an NSA forensics team. Your friend Barry enrolled in the Intrusion Analyst Skill Development Program and is touring with that team, so you message him to get the scoop. Barry tells you that a bunch of hard drives came back with the equipment, but most appear to be securely wiped. He managed to find a drive containing what might be some backups that they forgot to destroy, though he doesn't immediately recognize the data. Eager to help, you ask him to send you a zip containing a copy of the supposed backup files so that you can take a look at it.
> 
> If we could recover files from the drives, it might tell us what the APT is up to. Provide a list of unique SHA256 hashes of all files you were able to find from the backups. Example (2 unique hashes):
> 
> 
>   471dce655395b5b971650ca2d9494a37468b1d4cb7b3569c200073d3b384c5a4
>   0122c70e2f7e9cbfca3b5a02682c96edb123a2c2ba780a385b54d0440f27a1f6
> 
> Prompt:
> - Provide your list of SHA256 hashes

## Files given
- disk backups (archive.tar.bz2)

## Looking at the Files
when presented with a .tar file, the first thing I always do is run `tar -xvf [file name]`. Doing this resulted in a lot of files with the format: `logseq[number]-i` along with one file without the "-i" at the end. Running `file` on each of the files gives: `snapshots/logseq13302615617174: ZFS snapshot (little-endian machine), version 17, type: ZFS, destination GUID: FFFFFFF2 75 FFFFFFA7 FFFFFF89 FFFFFFC9 FFFFFFB0 4C 03, name: 'mhnbpool/ixfs@logseq13302615617174'`. After seeing this I looked more into ZFS snapshots and how to analyze them.

## Setting up ZFS
Based on [a Medium article about setting up virtual disks for ZFS]( https://medium.com/@abaddonsd/zfs-usage-with-virtual-disks-62898064a29b), I ran:
- `dd if=/dev/zero of=~/Documents/codebreaker_2024/task2/zpool_disk.img bs=1M count=64` (absolute path omitted for anonymity)
- `sudo zpool create testpool ~/Documents/codebreaker_2024/task2/zpool_disk.img` (absolute path omitted for anonymity)
- `zfs list`
- `sudo zfs create testpool/testdisk`

This created a ZFS pool and disk to use for hosting the snapshots

## Importing and Finding the Backups
Because all the snapshot increments needs to be added in order, I wrote a script to brute force and go over every iteration possible:
```python
#!/usr/bin/python3

files = ["logseq14870984571-i","logseq188149928221-i","logseq275591500024897-i","logseq2811679046771-i","logseq284582537021286-i","logseq31049129429664-i","logseq3221662148509-i","logseq32882693717455-i","logseq33022166721967-i","logseq359152092707-i","logseq40582166913031-i","logseq5763223028829-i","logseq69301907728591-i","logseq9224279729674-i","logseq173613109912986-i","logseq223322278614086-i","logseq7781332013904-i","logseq164091379732168-i","logseq19893200728065-i","logseq359152092707-i"]

for i in range(len(files)):
    for j in range(len(files)):
        print("cat ~/Documents/codebreaker_2024/task2/snapshots/" + files[j] + " | sudo zfs recv -F testpool/testdisk")
```

To import them, run `cat ~/Documents/codebreaker_2024/task2/snapshots/logseq13302615617174 | sudo zfs recv -F testpool/testdisk` followed by the result of the above command.

## Getting All the Hashes
Looking more into ZFS backups, I found that the incremental files are stored in `.zfs/snapshot`. With this, I wrote a quick script to get all the sha256 hashes of the backups: `sha256sum /testpool/testdisk/.zfs/snapshot/*/planning/pages/* | awk '{print $1}' >> out.txt` `sha256sum /testpool/testdisk/planning/logseq/* | awk '{print $1}' >> out.txt` `sha256sum /testpool/testdisk/planning/pages/* | awk '{print $1}' >> out.txt`

After getting all the hashes into out.txt, I opened it in vim and used the command `:%!uniq`. This left only the unique hashes that I could then submit to the challenge.

**NOTE**: The pages in this challenge are actually a good read, and they provide great advice and insight into tools and techniques related to the challenges
