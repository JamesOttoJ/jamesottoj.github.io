---
layout: default
title: NSA Codebreakers 2023
nav_order: 3
has_children: true
---

### Overview

Codebreakers is a cybersecurity challenge put on by the NSA that includes a series of challenges progressing in diffuculty from one to another. With each challenge requiring knowledge from the previous ones, it feels like a real investigation where analysts slowly unravel a problem starting from initial compromise on their system to full access on the attacker system. 

### My Role

Codebreakers is an individual challenge, so I did all of the challenge solving by myself. Even with this in mind, I did discuss my solutions and thought process with a fellow student after the competition ended. This provided an interesting insight into his thought process when looking at the same challenges as me.

### Skills

These challenges taught me a lot about how embeded systems worked as well as how they could be used for malicious purposes. Starting with task 1, I knew a good amount about SQL already, so remmebering how to add precision to float checking was the main hurdle. Moving on to task 2, this was another interesting challenge where I learned how to read chip IDs and how easy it is to grab datasheets for the chips online. Task 3 taught me about the boot process and how some chips can add complicaitons to it. It also taught me about uboot tooling and the basics of qemu. Task 4 was a lot easier after taking COM S 252 because I could read sed and awk. This allowed me to quickly identify the password format for decrpyting the USB drive. Task 5 added a lot to my reverse engineering toolbox. Between extracting files and analyzing go, I learned that there was a lot more to reverse engineering than just the compiled C programs and custom assembly I saw before.

### Tools

These challenges used a wide variety of tools. From tools for embeded systems like uboot and qemu to linux tools like sed, awk, and nc, each challenge had a new and unique toolset to learn. To learn more about the individual tools used, please look at the writeups for each task below

### Writups

- [Task 1](./task1.md)
- [Task 2](./task2.md)
- [Task 3](./task3.md)
- [Task 4](./task4.md)
- [Task 5](./task5.md)