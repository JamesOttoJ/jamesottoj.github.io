---
layout: default
title: Bio Bundle
parent: Hack the Box University CTF 2023
nav_order: 3
---

# Biobundle (rev)
{: .no_toc}
- TOC
{:toc}

### Inital Analysis
I always run `file`, `strings`, and `binwalk` on any reversing challenge. File shows that this is a typical x86_64 ELF with LSB. This means that it's a 64-bit linux executable where the bytes are stores with the least significant values first. Strings, however was quite interesting. It had all of the standard linux libraries, but it also had a lot of 7s and "/proc/self/fd/%d". The 7s will be important later, but the "/proc/self/fd/%d" shows that this program is doing something with files it opens in memory

### Static Analysis
main:
```c
undefined8 main(void)

{
  int iVar1;
  undefined8 uVar2;
  size_t sVar3;
  char local_98 [128];
  code *local_18;
  undefined8 local_10;
  
  local_10 = get_handle();
  local_18 = (code *)dlsym(local_10,&DAT_00102019); // DAT_00102019 -> "_"
  if (local_18 == (code *)0x0) {
    uVar2 = 0xffffffff;
  }
  else {
    fgets(local_98,0x7f,stdin);
    sVar3 = strcspn(local_98,"\n");
    local_98[sVar3] = '\0';
    iVar1 = (*local_18)(local_98);
    if (iVar1 == 0) {
      puts("[x] Critical Failure");
    }
    else {
      puts("[*] Untangled the bundle");
    }
    uVar2 = 0;
  }
  return uVar2;
}
```

Opening the file in ghidra and looking at the main function, it's pretty straight forward (assuming you know the terminology). get_handle() is custom function that we'll dive more into later, but it essentially opens a dynamic library and returns a "handle" for it (handles are like file descriptors for libraries). dlsym() gets the address of the _() function in the library that was opened by get_handle. After that, the code gets user input and passes it into the _() function from the library.

From here we know that we need to find out what the _() function is and what it checks for in the user data. To do that, we need to look at get_handle(), but it's a bit longer, so I'll go chunk by chunk

```c
  local_14 = memfd_create(&DAT_00102004,0); // DAT_00102004 -> ":^)"
  if (local_14 == 0xffffffff) {
    exit(-1);
  }
```

The above chunk starts off the function by creating a file in memory named, ":^)" and exits if it fails to open

```c
  for (local_10 = 0; local_10 < 0x3e08; local_10 = local_10 + 1) {
    local_1029 = __[local_10] ^ 0x37;
    write(local_14,&local_1029,1);
  }
```

The next chunk goes byte by byte through a data segment labeled "__" and xors it with 0x37 which is ascii for 7 (this is where the 7s come in). As it xors each byte, it appends it to the previously opened file. It does this for 15880 bytes of the data segment. Of note, the first four bytes of the data, xored with 7, is: 7f 45 4c 46. These 4 bytes match the ELF file signature, so we know it's an executable.

```
        00101246 48 8d 95        LEA        RDX=>local_1018,[RBP + -0x1010]
                 f0 ef ff ff
        0010124d b8 00 00        MOV        EAX,0x0
                 00 00
        00101252 b9 fe 01        MOV        ECX,0x1fe
                 00 00
        00101257 48 89 d7        MOV        RDI,RDX
        0010125a f3 48 ab        STOSQ.REP  RDI
```

The next chunk of code is in assembly because the c code doesn't look quite right. The address of RBP-0x1010 is loaded into the RDX register, and then preperation for the STOSQ,REP operation happens. STOSQ coppies the string at EAX into the location pointed to by RDI (RDX is moved into RDI, so it's the aforementioned value). REP then looks at RCX to determine how many times to run STOSQ (incrementing each time). This is likely used to clear out leftover data from the stack to make analysis harder.

```c
  sprintf((char *)&local_1028,"/proc/self/fd/%d",(ulong)local_14);
  local_20 = dlopen(&local_1028,1);
  if (local_20 == 0) {
    exit(-1);
  }
  return local_20;
```

The final chunk is pretty simple, it takes the file we created in memory, and opens it as a dynamic library. If it fails to open, it exits. If it succeeds, the handle is passed back to main.

At this point, there are two paths: the easy path, and the fun path. The easy path is to copy the bytes from the given data segment, throw them into cyberchef, and xor them. This will get you the loaded binary where you will be able to find the _ function (given below)

```c
bool _(char *param_1)

{
  int iVar1;
  undefined8 local_48;
  undefined8 local_40;
  undefined8 local_38;
  undefined8 local_30;
  undefined8 local_28;
  undefined8 local_20;
  undefined8 local_18;
  undefined8 local_10;
  
  local_48 = 0x743474737b425448; // "t4ts{BTH"
  local_40 = 0x5f3562316c5f6331; // "_5b1l_c1"
  local_38 = 0x6c3030635f747562; // "l00c_tub"
  local_30 = 0x7d7233; // "}r3"
  local_28 = 0;
  local_20 = 0;
  local_18 = 0;
  local_10 = 0;
  iVar1 = strcmp(param_1,(char *)&local_48);
  return iVar1 == 0;
}
```

If you know that the flag format is HTB{flag_here}, then it's easy to tell that it's a character array of the flag that is little endian and gets checked against the user input. Putting this into CyberChef with from hex and reverse (and making sure to put the bottom one first), we get the flag: HTB{st4t1c_l1b5_but_c00l3r}

### Dynamic analysis
If you want to do this the fun way or see what the assembly portion was trying to clear, welcome to dynamic analysys. We start by running gdb with biorev as the file to debug. After that, I ran `disas get_handle` and compared that to what is in ghidra. After comparing the two, I found that +123 was the offset right after the file got encrypted, so I set a breakpoint there with `b *(get_handle+123)`. Then, I ran the file and waited for it to hit the brakpoint. In a new tab, we have to find the PID of the process with `ps x | grep biobundle`. You can then do `ls -l /proc/[pid]/fd` to find the ":^)" file (It may say deleted, but that's okay). With the fd number, run `dd if=/proc/[pid]/fd/[number] of=proc_data bs=1` to take the data of the file and put it into the file "proc_data". The file will then show the same function as above and allow yout to solve.

What was up with clearing that data though? If you type "n" for "next" you can watch what each instruction does (I recomend the [peda](https://github.com/longld/peda) extension to help). It looks to me like it's cleaning up all of the local variables used by the program as well as data set by the memfd_create() function so that an analyst wouldn't know that a temp file was created or where it might be.