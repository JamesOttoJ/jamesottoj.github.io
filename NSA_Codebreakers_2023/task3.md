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

