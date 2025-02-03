---
layout: default
title: Task 6
parent: NSA Codebreakers 2024
nav_order: 7
mermaid: true
---

<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs';
  mermaid.initialize({ startOnLoad: true });
</script>

# TASK 6
{: .no_toc}
- TOC
{:toc}

### Task Description
>  The recovered data indicates the APT is using a DNS server as a part of their operation. The triage team easily got the server running but it seems to reply to every request with errors.
> 
> You decide to review past SIGINT reporting on the APT. Why might the APT be targeting the Guardian Armaments JCTV firmware developers? Reporting suggests the APT has a history of procuring information including the location and movement of military personnel.
> 
> Just then, your boss forwards you the latest status update from Barry at GA. They found code modifications which suggest additional DNS packets are being sent via the satellite modem. Those packets probably have location data encoded in them and would be sent to the APT.
> 
> This has serious implications for national security! GA is already working on a patch for the firmware, but the infected version has been deployed for months on many vehicles.
> 
> The Director of the NSA (DIRNSA) will have to brief the President on an issue this important. DIRNSA will want options for how we can mitigate the damage.
> 
> If you can figure out how the DNS server really works maybe we will have a chance of disrupting the operation.
> 
> Find an example of a domain name (ie. foo.example.com.) that the DNS server will handle and respond with NOERROR and at least 1 answer. 
> 
> Prompt:
> - Enter a domain name which results in a NOERROR response. It should end with a '.' (period)

### New Files From the Decrypted USB
- coredns
- Corefile
- microservice (Not used in this task. For task 7)

### Corefiles
The `coredns` and `microservice` files are executables while the `Corefile` file is text, so I started there. The contents are:
```
.:1053 {
  acl {
    allow type A
    filter
  }
  view firewall {
    expr type() == 'A' && name() matches '^x[^.]{62}\\.x[^.]{62}\\.x[^.]{62}\\.net-x7yfcbnc\\.example\\.com\\.$'
  }
  log
  cache 3600
  errors
  frontend
}
```

I knew nothing about this, so I knew I would need to look it up. Looking into [Corefiles](https://coredns.io/2017/07/23/corefile-explained/), it appears that it is used by the `coredns` program to manage how it handles queries coming in. Breaking down what it does:

`.:1053` means that these rules will apply to requests for any domain or IP as long as it is received on port 1053. 

```
  acl {
    allow type A
    filter
  }
```
acts as an access control policy. It will send a NOERROR if it matches the filter (any A name request).

```
  view firewall {
    expr type() == 'A' && name() matches '^x[^.]{62}\\.x[^.]{62}\\.x[^.]{62}\\.net-x7yfcbnc\\.example\\.com\\.$'
  }
```
blocks any requests that aren't A name requests and match the regex: `'^x[^.]{62}\\.x[^.]{62}\\.x[^.]{62}\\.net-x7yfcbnc\\.example\\.com\\.$'`. Using [regex101](https://regex101.com/), I found that this regex checks for a string matching exactly: `x[exactly 62 non-period characters].x[exactly 62 non-period characters].x[exactly 62 non-period characters].net-x7yfcbnc.example.com.`. The carat at the beginning means it has to be the start, and the dollar sign at the end means the line has to end there, so no additional characters can be added to the beginning or end. An example string that would work is: "xAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA.xAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA.xAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA.net-x7yfcbnc.example.com."

`log` writes a log of the query to stdout

`cache 3600` caches all responses for 3600 seconds

`errors` prints any errors that have been encountered thusfar

With all my searching, I couldn't find any reference to a `frontend` plugin in coredns. This was my first clue that the coredns executable wasn't normal, so I ran `file` on it, downloaded a fresh version, and checked the hashes
```
┌─[jamesj@parrot]─[~/Documents/codebreaker_2024/task6]
└──╼ $file files/coredns 
files/coredns: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, stripped
```
```
Source: `7a2d00cc8a9a8751d7c5c00a4e83e224e517ab4522fa59a4be4a2f594287d0c3  ./coredns`
On USB: `629fe157c525b885439d8d48ac38b0bdc1a81ab06e22197d0e896d972879dd0d  coredns`
```
This means that `frontend` is likely a new plugin put in by the challenge authors

### Reverse Engineering
Realizing the program had been altered, I popped it into Ghidra and sighed deeply. It's a stripped go binary... Luckily, I found an interesting trick while looking for relevant code segments in the assembly. Looking for the string, "frontend" in the program showed the following results:
```bash
┌─[jamesj@parrot]─[~/Documents/codebreaker_2024/task6]
└──╼ $strings files/coredns | grep -i frontend
*frontend.keypair
*frontend.Frontend
*frontend.cipherstate
*frontend.noisesession
*frontend.symmetricstate
*frontend.handshakestate
/logout[%s] %v%s: %d matches,:;%+-^%s(%#v)DECRYPTENCRYPTSUCCESSFAILUREspan %s"Unset""Error"PackageDefaultMessageImportsMethods!!merge.membered25519mappingopenapiserversexplodeuntypedenabledmarshal%.0f %s%.1f %s]?)(.*)(/.*)?$Corefiledns.porttransferdb\.(.*)autopathbradbeamekleinergreenpaunchrisdkpmoroneyrtreffersnebel29yongtangclouddns[DEBUG] %s%s%06x.privateresponseendpointdisabled{common}metadata/metricsrevision[ERROR] continuetemplateparseInttimeoutsservednsfrontendbad typedurationGoString01234567beEfFgGvsignal: truncatereadlinkscavengepollDesctraceBufdeadlockraceFinipanicnilcgocheckrunnable procid  is not  pointer packed=BAD RANK status unknown(trigger= npages= nalloc= nfreed=) errno=[signal GODEBUG= newval= mcount= bytes, , errno=
=>	../../frontend	(devel)	
github.com/coredns/example.Frontend.ServeDNS
github.com/coredns/example.Frontend.Name
github.com/coredns/example.(*Frontend).Name
github.com/coredns/example.(*Frontend).ServeDNS
github.com/coredns/example@v0.0.0-20200925060636-a998e071a3a3/frontend.go
=>	../../frontend	(devel)
```

#### Unstripping
Looking at this, I wondered if I had missed something, so I checked the coredns github just in case, and I didn't see any mention of frontend still. Interestingly, those strings at the top kind of looked like functions signatures, so I went to look at where in the code they were and what references they had in Ghidra. Ghidra just showed this in the `.rodata` segment, though:
```
        01dc1090 12              ??         12h
        01dc1091 2a 66 72        ds         "*frontend.Frontend"
                 6f 6e 74 
                 65 6e 64 

```
I found it strange that this string would be in here without a reference, so I looked for the address value in hex throughout the program just in case. When doing this, I had to keep in mind that go stores strings as length, data pairs, so the 0x12 byte before it was likely part of the string object still. Searching for  0x01dc1090 got a hit at 0x0321f946 (I tried 0x01dc1091 just in case, but no hits). The address it took me to was in a segment called `.gopclntab`. I had never heard of it before, so I decided to look it up. Aparently, it's what go uses to link function names and addresses even when the program is supposedly stripped. This was good news because I could likely get real function names back. (Looking back at task 2 while doing writeups, this was actually hinted in one of the files `golang.md`: "**Go**: Go binaries often contain symbol information, even when stripped. This can be beneficial for reverse engineering but can also make the binary larger and potentially more complex.")

Looking for tools to restore these function names, I found [GoReSym from Mandiant](https://github.com/mandiant/GoReSym). This allowed me to put all the function names back and made it a lot easier to continue.

#### What Does it Do?
Looking for "frontend" in the function pane showed two main functions:
- github.com/coredns/example.Frontend.Name
- github.com/coredns/example.Frontend.ServeDNS

The Name function appears to just give its name (likely just necessary for any and all coredns plugin). ServeDNS is a lot more interesting. Like I do with many go programs, I start by getting a gist from just looking for strings and function calls. This can get an idea for how the program flow goes. Tracing through it, this is a basic function call graph I crafted:
<div class="mermaid">
%%{
  init: {
    'theme': 'base',
    'themeVariables': {
      'primaryColor': '#204',
      'primaryTextColor': '#fff',
      'primaryBorderColor': '#ddd',
      'lineColor': '#fff',
      'secondaryColor': '#006100',
      'tertiaryColor': '#fff'
    }
  }
}%%
flowchart TD;
	A([example.Frontend.ServeDNS]) --> B([example.name2buffer]);
	A --> C([example.NoiseRecv]);
	A --> D([example.doForwardData]);
	C --> E([example.InitSession]);
	C --> F([example.RecvMessage]);
	D --> G([example.doForwardData.func1]);
	E --> H([example.initializeInitiator]);
	E --> I([example.initializeResponder]);
	F --> J([example.readMessageA]);
	F --> K([example.readMessageRegular]);
	H --> L([example.initializeSymmetric]);
	H --> M([example.mixHash]);
	I --> L;
	I --> M;
	J --> N([example.validatePublicKey]);
	J --> M;
	J --> O([example.mixKey]);
	J --> P([example.decryptAndHash]);
	J --> Q([example.split]);
	K --> R([example.decryptWithAd]);
	L --> S([example.hashProtocolName]);
	O --> T([example.getHkdf]);
	P --> R;
	P --> M;
	Q --> T;
	R --> U([example.decrypt]);
</div>

### Protocol Misuse

### Crafting a Payload