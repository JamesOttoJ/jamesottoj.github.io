---
layout: default
title: Follow The path
parent: HTB Apocalypse 2024
nav_order: 1
---

# Follow The Path
{: .no_toc}
- TOC
{:toc}

### Where to start
With all binaries, I start with a few programs to get an idea of what I'm working with:
```
┌─[jamesj@parrot]─[~/Documents/apocalypse_2024/rev_followthepath]
└──╼ $file chall.exe 
chall.exe: PE32+ executable (console) x86-64, for MS Windows, 10 sections
┌─[jamesj@parrot]─[~/Documents/apocalypse_2024/rev_followthepath]
└──╼ $binwalk -Me chall.exe 

Scan Time:     2024-07-16 16:23:56
Target File:   /home/jamesj/Documents/apocalypse_2024/rev_followthepath/chall.exe
MD5 Checksum:  0be00d4ba92e5906137b81bf22e61120
Signatures:    411

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             Microsoft executable, portable (PE)
122464        0x1DE60         XML document, version: "1.0"

┌─[jamesj@parrot]─[~/Documents/apocalypse_2024/rev_followthepath]
└──╼ $strings chall.exe 
!This program cannot be run in DOS mode.$
.text
.rdata
```
I like to work with malware, so file helps to ensure that the program I'm running is what I expect it to be (which this seems to be). Binwalk helps to find hidden programs that may be extracted later or hidden file information it accesses (which this doesn't). Strings can show what the program does, but this program mostly had metadata and random snippets of strings at a glance. To check for more relevant data, I often filter for more relevant terms like the HTB (the flag format), flag, input, password, or terms from the challenge description. On this file `strings chall.exe | grep flag` turned up "Please enter the flag".

Knowing that the program wants me to submit the flag, I know that it has to check the input against something in the code to validate each character. I also know I can find what part of the code uses that string to determine where that checking might start.

### Getting the context

Now that I know where to start my analysis, I disassembled the code in Ghidra and looked for the "Please enter the flag" string. This brought me to FUN_140001960
```
void FUN_140001960(void)

{
  _iobuf *p_Var1;
  undefined auStack_d8 [32];
  char *local_b8;
  undefined *local_b0;
  undefined *local_a8;
  code *local_a0;
  char local_98 [128];
  ulonglong local_18;
  
  local_18 = DAT_14001c008 ^ (ulonglong)auStack_d8;
  FUN_1400046d0((longlong)"Please enter the flag"); // Ask for input
  p_Var1 = (_iobuf *)FUN_140003064(0);
  local_b8 = local_98;
  thunk_FUN_1400045f8(local_b8,0x7f,p_Var1); // Get input (See the end for a more full analysis of this function)
  // Switched to asm
  1400019a4 LEA        RAX,[FUN_140001000]
  1400019ab MOV        qword ptr [RSP + local_a0],RAX=>FUN_140001000
  1400019b0 LEA        RAX,[LAB_140001a00]
  1400019b7 MOV        qword ptr [RSP + local_a8],RAX=>LAB_140001a00
  1400019bc LEA        RAX,[LAB_140001a20]
  1400019c3 MOV        qword ptr [RSP + local_b0],RAX=>LAB_140001a20
  1400019c8 MOV        qword ptr [RSP + local_b8],RSI
  1400019cd MOV        R10,qword ptr [RSP + local_a8]
  1400019d2 MOV        R11,qword ptr [RSP + local_b0]
  1400019d7 MOV        R12,qword ptr [RSP + local_b8]
  1400019dc XOR        RCX,RCX
  1400019df JMP        qword ptr [RSP + 0x38]=>FUN_140001000

}
```
This function gets pretty cleanly disassembled by ghidra, but the general idea is that it asks for input, gets the input, sets parameters for another function, and calls that function. The parameters are passed in a somewhat non-standard way, so I felt it would be easiest to switch to assembly there. The registers passing data to the `FUN_140001000` function are R10, R11, R12, and RCX. The get set to the values below:
- R10 = (void *) 0x140001a00 // A code block that prints "nope!"
- R11 = (void *) 0x140001a20 // A code block that prints "correct"
- R12 = RSI // A pointer to the user's input
- RCX = 0

Once those are set, `FUN_140001000` is called

# The head of the snake

```
void FUN_140001000(longlong param_1)

{
  longlong lVar1;
  undefined4 unaff_EBX;
  code *UNRECOVERED_JUMPTABLE;
  longlong unaff_R12;
  
  if (*(char *)(unaff_R12 + param_1) != 'H') {
                    /* WARNING: Could not recover jumptable at 0x00014000101b. Too many branches */
                    /* WARNING: Treating indirect jump as call */
    (*UNRECOVERED_JUMPTABLE)();
    return;
  }
  lVar1 = 0;
  do {
    *(byte *)(lVar1 + 0x140001039) = *(byte *)(lVar1 + 0x140001039) ^ 0xde;
    lVar1 = lVar1 + 1;
  } while (lVar1 != 0x39);
  out(0x39,unaff_EBX);
                    /* WARNING: Bad instruction - Truncating control flow here */
  halt_baddata();
}
```

Above is what the function looks like decompiled. There are a couple things to note. First, there seems to be a bit of odd code and a couple warnings. This is usually a signal that looking at the assembly would be best. Second, it appears to be checking a value against the character H. In normal malware, this would just look like a strange artifact, but this is a Hack The Box challenge. That means that the string "HTB{" should always be top of mind because it often indicates that it has something to do with the flag. The "H" here then tells me that it might be looking for a value to hold the flag and exit if it doesn't.

The chunk after that if statement does have something more notable it takes the variable `lvar1`, sets it to 0, and uses it to access an offest of memory. This part of memory should be notable because our function starts at 0x140001000, and it is accessing memory that is just 39 bytes past the start of the function. Using C pointer knoledge, it looks like it takes the value of the byte at that address, XORs it with 0xde, and put the result in the same location. After it changes that byte, it increments `lvar1`, checks that it is less than 0x39, and repeats. This is where looking at the assembly can give a better perspective.

```
140001000 4d 31 c0        XOR        R8,R8                             // Clear R8
140001003 45 8a 04 0c     MOV        R8B,byte ptr [R12 + param_1*0x1]  // R8 = user_in[0]
140001007 49 81 f0        XOR        R8,0xc4                           // R8 = R8 ^ 0xc4
          c4 00 00 00
14000100e 49 81 f8        CMP        R8,0x8c                           // if (R8 == 0x8c) { // Can also be translated to R8 ^ 0x8c == 0
          8c 00 00 00
140001015 0f 84 03        JZ         LAB_14000101e                     //   goto 0x14000101e
          00 00 00                                                     // }
14000101b 41 ff e2        JMP        R10                               // print "nope!"
                      LAB_14000101e  
14000101e 48 ff c1        INC        param_1                           // i++
140001021 4c 8d 05        LEA        R8,[LAB_140001039]                // R8 = 0x140001039
          11 00 00 00
140001028 48 31 d2        XOR        RDX,RDX                           // j = 0
                      LAB_14000102b  
14000102b 41 80 34        XOR        byte ptr [R8 + RDX*offset LAB_140001039],0xde // *(byte *) (0x140001039 + j) ^= 0xde
          10 de
140001030 48 ff c2        INC        RDX                               // j++
140001033 48 83 fa 39     CMP        RDX,0x39                          // if (j != 0x39) {
140001037 75 f2           JNZ        LAB_14000102b                     //   goto 0x14000102b
                      LAB_140001039                                    // }
140001039 93              XCHG       EAX,EBX                           // ???
                      LAB_14000103a
14000103a ef              OUT        DX,EAX                            // ???
14000103b 1e              ??         1Eh
14000103c 9b              ??         9Bh
14000103d 54              ??         54h    T
```

Above is the assembly for the previous cuntion (Note that it starts at 0x140001000). Keep in mind the arguments put into R10, R11, R12, and RCX (`param_1`) before. With that in mind, the first interesting thing to me was that there is only JMP, Jcc (conditional jump), CALL, or RET that goes outside of this function. That is at 0x14000101b where is jumps the the function that prints "Nope!". That immediately told me that there was more to this function than what got dissassembled. 

Starting with what we already know, up to the first label (`LAB_14000101e`), the code checks if the user input is equal to 'H'. This is found by using the associative property of xor (`(user_in[i] ^ 0xc4) ^ 0x8c = user_in[i] ^ (0xc4 ^ 0x8c)`). After that, it prints "nope!" if it's incorrect or continues to `LAB_14000101e`. `LAB_14000101e` will increment i, store the address `0x140001039`, and set another variable to 0. After that, it goes into a loop where it xors the 57 (0x39) bytes after `0x140001039`. The code following that loop then looks a bit... off. Checking the address, we can see that it was the location of the bytes that get xored earlier. This is where the crux of the problem lies.

Now that we know how many bytes are acccessed along with what they get XORed with, we can copy the 57 bytes starting at `0x140001039`. After ralizing this, I copied the bytes and put them into cyberchef. This allowed me to get the real bytes for the assembly code. After this, I found an [online disassembler](https://defuse.ca/online-x86-assembler.htm) to get the following next segment.

```
0:  4d 31 c0                xor    r8,r8  
3:  45 8a 04 0c             mov    r8b,BYTE PTR [r12+rcx*1]  
7:  49 81 f0 55 00 00 00    xor    r8,0x55  
e:  49 81 f8 01 00 00 00    cmp    r8,0x1  
15: 0f 84 03 00 00 00       je     0x1e  # Go if second byte is T
1b: 41 ff e2                jmp    r10  # Jump to nope
1e: 48 ff c1                inc    rcx  
21: 4c 8d 05 11 00 00 00    lea    r8,[rip+0x11]        # 0x39  
28: 48 31 d2                xor    rdx,rdx  
2b: 41 80 34 10 eb          xor    BYTE PTR [r8+rdx*1],0xeb  
30: 48 ff c2                inc    rdx  
33: 48 83 fa 39             cmp    rdx,0x39  
37: 75 f2                   jne    0x2b  
```

A lot of this assembly should look similar. It isn't notated or broken up by labels, but it is essentially the same. Now that RCX was incremented in the previous segment, the next character in the user's input will be checked. Now that we know that we can look at only a key few points. The values at line 7 and e tell us what values to XOR to find the character we want in the user string. The value at line 2b will tell us what to xor the next segment with to comtinue along.

During the competition itself, I did all of this by hand. I coppied each segment of bytes, XORed it, and disassembled it before storing it in my notes. This took around 2 hours and was a bad idea. A month or two after, I was looking at what was possible with python's pwn library, and I realized that I could automate the process. With that in mind, I developped the folling script to make this challenge less labor intensive.

Because this isn't an ELF, I couldn't nicely input and parse things, so I started by finding the real byte offsets of the functions I wanted with `LANG=C grep -obUaP "\x4d\x31\xc0" /bin/grep` which was 1024. Looking at the machine code, we have 3 offsets we're interested in. The two bytes used to get the flag are at offsets 0xA and 0x11. The byte value used to XOR the next 57 bytes is at 0x2F. This is all we need to take in the stream of bytes from the file to get the flag.

solve.py
```
#!/usr/bin/python3

with open("chall.exe", "rb") as f:
    offset = f.read(1024)
    key = 0
    flag = b''
    while (segment := f.read(57)):
        decrypted_segment = b''
        for i in range(len(segment)):
            decrypted_segment += (segment[i] ^ int(key)).to_bytes()
        if (decrypted_segment[0:3] != b'\x4d\x31\xc0'):
            print(flag)
            break;
        flag += (segment[10] ^ segment[17]).to_bytes()
        key = decrypted_segment[47]
```