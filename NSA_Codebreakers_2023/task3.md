# TASK 3

### Task Description
>Leveraging that datasheet enabled you to provide the correct pins and values to properly communicate with the device over UART. Because of this we were able to communicate with the device console and initiate a filesystem dump.
>
>To begin analysis, we loaded the firmware in an analysis tool. The kernel looks to be encrypted, but we found a second-stage bootloader that loads it. The decryption must be happening in this bootloader. There also appears to be a second UART, but we don't see any data coming from it.
>
>Can you find the secret key it uses to decrypt the kernel?
>
>Tips:
>
> - You can emulate the loader using the provided QEMU docker container. One download provides the source to build your own. The other is a pre-built docker image. See the README.md from the source download for steps on running it.
> - Device tree files can be compiled and decompiled with `dtc`.

### Files Given
u-boot.bin
device_tree.dtb
cbc_qemu_aarch64-source.tar.bz2

### What's up with these files?
First off, they're large, so I won't be linking most of them for this task. I will just provide relevant snippets. u-boot.bin is the bootloader that is extracted from the device. The device tree is then used by the bootloader to know what devices are available for input and output. The archive is just a common emulator that is used so that arm code can run while working on the challenge.

### Devices
Seeing as the challenge description gives a simple way to analyze the device tree, I started with that. There isn't anything too interesting in here that sticks out, but we can verify that UART pins are available for use.
```
pl011@9040000 {
		secure-status = "okay";
		status = "disabled";
		clock-names = "uartclk\0apb_pclk";
		clocks = <0x8000 0x8000>;
		interrupts = <0x00 0x08 0x04>;
		reg = <0x00 0x9040000 0x00 0x1000>;
		compatible = "arm,pl011\0arm,primecell";
	};
...
pl011@9000000 {
		clock-names = "uartclk\0apb_pclk";
		clocks = <0x8000 0x8000>;
		interrupts = <0x00 0x01 0x04>;
		reg = <0x00 0x9000000 0x00 0x1000>;
		compatible = "arm,pl011\0arm,primecell";
	};
```

### Diverging Paths
From here, there are two ways to go: dynamic analysis and static analysis. Dynamic analysis entails starting up Qemu, starting the bootloader, and poking around. Static analysis would mean putting the u-boot file into Ghidra and checking out how it works. I'm a bigger fan of dynamic analysis because I feel it provides better context, so I'm going to start there

### Dynamic Analysis
After opening the Qemu source archive, there are a couple files: a [Dockerfile](./static/Dockerfile), [qemu-ifup](./static/qemu-ifup), [qemu-ifdown](./static/qemu-ifdown), myfiles, and [README.md](./static/README.md). As always, I start with the README. It covers how to build Docker, run Docker, run Qemu, and connect. From there, I added u-boot.bin and device_tree.dtb to the "myfiles" folder as stated in the README. After that, I checked out the Docker file as well as the ifup and ifdown file. ifup and ifdown were basic networking files. The Dockerfile made a basic Ubuntu box and installed QEMU with aarch64 specified. It also puts all of the content from the "myfiles" folder into the Docker container. After following these steps and getting the propper netcat listeners up, a shell for u-boot opens up.

### Who Boot? U-boot!
Because I was new to working in U-boot, I started by running commands and doing cursory research. I found this [really helpful guide](https://www.digi.com/resources/documentation/digidocs/PDFs/90000852.pdf) that led me to the `printenv` command. I did try a couple others first, but that's the only thing that gave me something interesting. Of interest was `ivaddr=467a0010`, `kernel_addr_r=40400000`, and `keyaddr=467a0000`. Based on the format, it looks like these are memory locations. From the U-boot guide, there is one command called md that looks at memory address data. This stumped me for too long because I just used regular md which displays four byte chunks in lsb order with each chunk at increasing addresses. Regular md would turn the key: 1122334455667788 into 44332211 88776655. This tripped me up because I tried it in both the most significant bit (4433221188776655) and least significant bit (5566778811223344) format. Both of these are incorrect. To display it in a way that is easier to digest, use md.b which displays the bytes in ascending order. The key 1122334455667788 displays as 11 22 33 44 55 66 77 88 with md.b.

Running md.b 467a0000 gives the key f0eced4c9ef21abec6403b00c44adfb3

### What About Static Analysis?