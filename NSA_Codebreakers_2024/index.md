---
layout: default
title: NSA Codebreakers 2024
nav_order: 10
has_children: true
---

### Overview

Codebreakers is a cybersecurity challenge put on by the NSA that includes a series of challenges progressing in diffuculty from one to another. With each challenge requiring knowledge from the previous ones, it feels like a real investigation where analysts slowly unravel a problem starting from initial compromise on their system to full access on the attacker system. 

### My Role

Codebreakers is an individual challenge, so I did all of the challenge solving by myself. Even with this in mind, I did discuss my solutions and thought process with a fellow student after the competition ended. This provided an interesting insight into his thought process when looking at the same challenges as me.

### Skills

The challenges for this year focused on finding an attacker's hideout and revealing how they used tools to compromise military vehicle systems. To do this, I drew upon my background in reverse engineering, cryptography, and forensics. The per-task skill breakdown is as follows:

- Task 1:
    - File format knowledge
    - Parsing a large amount of reccords to find annomalies
- Task 2:
    - Using ZFS backups
- Task 3:
    - Go reverse engineering
    - gRPC interaction
    - Protobufs
    - Exploitation
- Task 4:
    - ANSI/ASCII characters
    - Scripting
    - Data parsing
    - mTLS
    - AI response analysis
- Task 5:
    - PYc reverse engineering
    - Host forensic analysis
    - AES block mode flaws
    - RSA implementation flaws (coppersmith/broadcast attack)
    - Cryptographic analysis
    - Gocryptfs
- Task 6:
    - Go reverse engineering
    - Noise cryptograpic protocol
    - Diffie-Hellman key exchanges
    - Symmetric vs asymmetric cryptography
    - RegEx parsing
    - DNS
- Task 7:
    - Compiled NodeJS reverse engineering (deno)
    - Express app analysis
    - msgpack
    - 

### Tool

These challenges used a wide variety of tools and libraries specific to the programs and scenarios. Across all of them, a lot of python scripts were reqired for automation and interaction. Other than that, the task-specific tools used were:

- Task 1:
    - Libre office
    - Pivot tables
- Task 2:
    - ZFS/zpool
    - sha256sum
- Task 3:
    - Go
    - gRPC
    - Ghidra
- Task 4:
    - mTLS
    - Caching servers
- Task 5:
    - Ghidra
    - objcopy
    - pyinstxtractor
    - gocryptfs
    - CyberChef
- Task 6:
    - Ghidra
    - nslookup
    - CyberChef
- Task 7:
    - strings