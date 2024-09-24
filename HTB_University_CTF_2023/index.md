---
layout: default
title: Hack the Box University CTF 2023
nav_order: 2
has_children: true
---

### Overview

The university CTF from Hack The Box was a challenge given to university students to solve puzzles near the end of the fall semester. It covered a lot of the traditional CTF categories like PWN, Rev, and Forensics.

### My Role

I had the chance to work with other members of the Hacking and Cybersecurity Club, and we each fell into different categories. My role focused on reverse engineering while we had other students doing categories like hardware and web. In the end, I was able to solve 2 challenges while another teammate solved a third one. While this wasn't the performance we expected, I still learned a lot about process management and general reverse engineering.

### Skills

The two challenges I solved for this competition both dealt with reverse engineering. The easier of the two, windows of oportunity, had to do with analyzing the logic flow of a compiled function to extract data. The next challenge, bio bundle, xors a segment of the program, stores the resultant code in a file, and runs that code. This helped me explore anti-reversing techniques along with how to solve problems using both dynamic and static reverse engineering.

### Tools

To solve these challenges, I mostly used Ghidra for examining the code and flow of information. On top of Ghidra, I also used a variety of helpers like dd, strings, and python to analyze and extract data.

### Writups

- [Windows of Oportunity](./windows_of_opportunity.md)
- [Bio Bundle](./bio_bundle.md)