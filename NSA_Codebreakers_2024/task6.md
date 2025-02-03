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

## Task Description
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

## New Files From the Decrypted USB
- coredns
- Corefile
- microservice (Not used in this task. For task 7)

## Corefiles
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
blocks any requests that aren't A name requests and match the regex: `'^x[^.]{62}\\.x[^.]{62}\\.x[^.]{62}\\.net-x7yfcbnc\\.example\\.com\\.$'`. Using [regex101](https://regex101.com/), I found that this regex checks for a string matching exactly: `x[exactly 62 non-period characters].x[exactly 62 non-period characters].x[exactly 62 non-period characters].net-x7yfcbnc.example.com.`. The carat at the beginning means it has to be the start, and the dollar sign at the end means the line has to end there, so no additional characters can be added to the beginning or end. An example string that would work is: "xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.net-x7yfcbnc.example.com."

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

## Reverse Engineering
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

### Unstripping
Looking at this, I wondered if I had missed something, so I checked the coredns github just in case, and I didn't see any mention of frontend still. Interestingly, those strings at the top kind of looked like functions signatures, so I went to look at where in the code they were and what references they had in Ghidra. Ghidra just showed this in the `.rodata` segment, though:
```
        01dc1090 12              ??         12h
        01dc1091 2a 66 72        ds         "*frontend.Frontend"
                 6f 6e 74 
                 65 6e 64 

```
I found it strange that this string would be in here without a reference, so I looked for the address value in hex throughout the program just in case. When doing this, I had to keep in mind that go stores strings as length, data pairs, so the 0x12 byte before it was likely part of the string object still. Searching for  0x01dc1090 got a hit at 0x0321f946 (I tried 0x01dc1091 just in case, but no hits). The address it took me to was in a segment called `.gopclntab`. I had never heard of it before, so I decided to look it up. Aparently, it's what go uses to link function names and addresses even when the program is supposedly stripped. This was good news because I could likely get real function names back. (Looking back at task 2 while doing writeups, this was actually hinted in one of the files `golang.md`: "**Go**: Go binaries often contain symbol information, even when stripped. This can be beneficial for reverse engineering but can also make the binary larger and potentially more complex.")

Looking for tools to restore these function names, I found [GoReSym from Mandiant](https://github.com/mandiant/GoReSym). This allowed me to put all the function names back and made it a lot easier to continue.

### What Does it Do?
Looking for "frontend" in the function pane showed two main functions:
- github.com/coredns/example.Frontend.Name
- github.com/coredns/example.Frontend.ServeDNS

The Name function appears to just give its name (likely just necessary for any and all coredns plugin). ServeDNS is a lot more interesting. Like I do with many go programs, I start by getting a gist from just looking for strings and function calls. This can get an idea for how the program flow goes. Tracing through it, this is a basic function call graph I crafted (I know it's a bit small):
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

Looking at the flow, I split it up into a few key steps:
1. Name decode
2. Crypto functions
  a. Initialize
  b. Decrypt data
3. Forward data elsewhere

### Parsing the Query Data
`example.name2buffer`:
```c
param_7 = in_RAX;
param_8 = unaff_RBX;
while (&stack0x00000000 <= *(undefined **)(unaff_R14 + 0x10)) {
  runtime.morestack_noctxt();
  param_3 = extraout_RDX_03;
}
                  /* Split Domain name by . and return the first 3 */
auVar6 = strings.genSplit(1,0,param_3,".",4);
lVar1 = auVar6._0_8_;
if (3 < param_8) {
  runtime.concatstring3
            (*(undefined8 *)(lVar1 + 0x10),*(undefined8 *)(lVar1 + 0x18),auVar6._8_8_,
            *(undefined8 *)(lVar1 + 8),*(undefined8 *)(lVar1 + 0x20),
            *(undefined8 *)(lVar1 + 0x28));
                  /* Remove all instances of xn-- */
  strings.Replace(4,0,extraout_RDX,"xn--",0,0xffffffffffffffff);
                  /* Remove all instances of x */
  strings.Replace(1,0,extraout_RDX_00,"x",0,0xffffffffffffffff);
                  /* Remove all instances of y */
  strings.Replace(1,0,extraout_RDX_01,"y",0,0xffffffffffffffff);
  lVar4 = 1;
                  /* Replace z with = */
  strings.Replace();
  strings.ToUpper();
  encoding/base32.(*Encoding).DecodeString();
  ...
}
```
These parts are the most important ones from this function. It starts off with [genSplit](https://github.com/golang/go/blob/master/src/strings/strings.go#L291) (an internal version of Split not available to programmers). Looking at the source code, though, it appears to split the string by a given character and return the first n-1 elements. For the example DNS strings above it would do the following: 
`genSplit("xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.net-x7yfcbnc.example.com.", ".", 0, 4) -> xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaxaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaxaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa`
After that, it takes the result and removes all instances of "xn--", "x", and "y". This turns that string into "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa" and leaves only the characters we choose. Next, it replaces any "z"s with "=" and turns all the letters to uppercase. Lastly, it decodes it from base32. **Of note**, base32 decoders (like the one in CyberChef) will typically use the character space "A-Z0-7=". Go's base32 library mentions a separate character space: "0-9A-V=" called [HexEncoding](https://pkg.go.dev/encoding/base32#pkg-variables) that gets used for DNS contexts, and it will be used here because x, y, and z all get altered in the code.

### Encryption and Noise
Looking at the next category, the main function is called `example.NoiseRecv`. While I learned how the functions worked by looking over the code and walking through important parts in dynamic analysis (you can see me tracing function args in my [notepad](../notepad)), I feel that it's a lot easier to just look up the function name and look at the source code/docs. I started with [flynn's noise package for go](https://github.com/flynn/noise). That package does a good job showing off how the protocol works, but I was reading write-ups after the competition, and I read [a write-up by jp0x1](https://jp0x1.github.io/blog/nsa-cbc/#noise-protocol) that appears to have found the [actual source code](blob:https://noiseexplorer.com/602a6433-137a-43ff-a78b-905c4b02670a) from a website that covers the attack used later. Using the second code source also makes it easier to interact with the protocol itself.

Moving back to analyzing the protocol, I was first tipped off that it may be a known overarching cryptographic protocol when I saw the cryptographic protocol string: "Noise_K_25519_ChaChaPoly_BLAKE2s". This gave the overarching protocol as well which type and which protocols it uses. The noise protocol string in particular breaks down to:
- Protocol: Noise
- Handshake type: K
- DH protocol: Curve25519
- Cipher: ChaChaPoly
- Hash: BLAKE2s

Of these, the focus will be on the handshake mode because the other protocols don't have relevant flaws here.

## Protocol Misuse
To understand the attack, it's important to understand how the protocol works. In short, this is what the protocol does in the code:

**Variables**
- s, e: The local party's static and ephemeral key pairs (which may be empty).
- rs, re: The remote party's static and ephemeral public keys (which may be empty).
- h: A handshake hash value that hashes all the handshake data that's been sent and received.
- ck: A chaining key that hashes all previous DH outputs. Once the handshake completes, the chaining key will be used to derive the encryption keys for transport messages.
- k, n: An encryption key k (which may be empty) and a counter-based nonce n. 

**Functions**
- mixHash(value): h = HASH(h \|\| value)
- mixKey(ck, key_material): ck, k = HKDF(ck, key_material, 2)

**Initialize** (initializeSymmetric)
Both:
  h = HASH("Noise_K_25519_ChaChaPoly_BLAKE2s")
  ck = h
  mixHash("") // Normally the prologue, but not used here
  mixHash(our_public_key) 
  mixHash(their_public_key)

**Send Message** (writeMessageA)
Initiator:
  Generate new key pair: e
  mixHash(e.public) // e
  mixKey(ck, dh(e.private, their_public_key)) // es
  mixKey(ck, dh(our_private_key, their_public_key)) // ss
  ciphertext = encryptAndHash(plaintext)
  Send e + ciphertext

**Receive Message** (readMessageA)
Receiver:
  MixHash(e.public) // e
  mixKey(ck, dh(their_private_key, e.public)) // es
  mixKey(ck, dh(their_private_key, our_public_key)) // ss
  decryptAndHash(h, ciphertext)

With the K handshake method, it's also important to note that it's expected that each person shares their public key with the other beforehand. This means that the server must know three important things: its private key, its public key, and our public key. Knowing where these are in the protocol, I can line it up to the functions and look at what is being passed into each function during dynamic analysis:
```
Server Public (0x03b83900): `884c809374464472ca6b937ce3620750caf3569f069f09eeeff4c89e4161400e`
Server Private (0x03b83920): `c00148283ae459fc94519a4d749bd17529769ce014575a6fa55f8127376a8429`
Our Public (0x03b83940): `e451a5067a33e891a8d9cde65da8f2fb74405c7debf89e3157f5c6e3458ec34e`
```

After looking at the above protocol, I realized the one thing that allows us to communicate with only those three things: Diffie-Hellman (DH) is made to give symmetric results with asymmetric information. This gives the following truths:
- dh(e.private, their_public_key) == dh(their_private_key, e.public)
- dh(our_private_key, their_public_key) == dh(their_private_key, our_public_key)

This means that we just need to change the sending part of the protocol to:
  Generate new key pair: e
  mixHash(e.public) // e
  mixKey(ck, dh(e.private, their_public_key)) // es
  mixKey(ck, dh(their_private_key, our_public_key)) // ss
  ciphertext = encryptAndHash(plaintext)
  Send e + ciphertext

## Crafting a Payload
At this point, it was the final day, and I had an event, so I didn't score any points for this task. The next day, I looked at the write-up above and use the code they referenced to solve the challenge. To help read the code, the protocol would normally identify `s` as our keypair and `rs` as their keypair, but these are reversed in this case (reference the keys above if confused):
```go
//...
func writeMessageA(hs *handshakestate, payload []byte) ([32]byte, messagebuffer, cipherstate, cipherstate, error) {
	var err error
	var messageBuffer messagebuffer
	ne, ns, ciphertext := emptyKey, []byte{}, []byte{}
	hs.e = generateKeypair()
	ne = hs.e.public_key
	mixHash(&hs.ss, ne[:])
	/* No PSK, so skipping mixKey */
	mixKey(&hs.ss, dh(hs.e.private_key, hs.s.public_key)) // e private and their public
	mixKey(&hs.ss, dh(hs.s.private_key, hs.rs)) // their private and our public
	_, ciphertext, err = encryptAndHash(&hs.ss, payload)
	if err != nil {
		cs1, cs2 := split(&hs.ss)
		return hs.ss.h, messageBuffer, cs1, cs2, err
	}
	messageBuffer = messagebuffer{ne, ns, ciphertext}
	cs1, cs2 := split(&hs.ss)
	return hs.ss.h, messageBuffer, cs1, cs2, err
}
//...
func main() {
  type location_event struct {
    v string
    t int
    m int
    d int
  }

  log.Printf("Starting...")

  privateKey := [32]byte{0xc0, 0x01, 0x48, 0x28, 0x3a, 0xe4, 0x59, 0xfc, 0x94, 0x51, 0x9a, 0x4d, 0x74, 0x9b, 0xd1, 0x75, 0x29, 0x76, 0x9c, 0xe0, 0x14, 0x57, 0x5a, 0x6f, 0xa5, 0x5f, 0x81, 0x27, 0x37, 0x6a, 0x84, 0x29}
  publicKey := [32]byte{0x88, 0x4c, 0x80, 0x93, 0x74, 0x46, 0x44, 0x72, 0xca, 0x6b, 0x93, 0x7c, 0xe3, 0x62, 0x07, 0x50, 0xca, 0xf3, 0x56, 0x9f, 0x06, 0x9f, 0x09, 0xee, 0xef, 0xf4, 0xc8, 0x9e, 0x41, 0x61, 0x40, 0x0e}
  staticRS := keypair{
    private_key: privateKey,
    public_key:  publicKey,
  }

  PubKey := [32]byte{0xe4, 0x51, 0xa5, 0x06, 0x7a, 0x33, 0xe8, 0x91, 0xa8, 0xd9, 0xcd, 0xe6, 0x5d, 0xa8, 0xf2, 0xfb, 0x74, 0x40, 0x5c, 0x7d, 0xeb, 0xf8, 0x9e, 0x31, 0x57, 0xf5, 0xc6, 0xe3, 0x45, 0x8e, 0xc3, 0x4e}
  
  prologue := []byte{}
  buf := new(bytes.Buffer)
  // Reset sessions at the start of each iteration
  initiatorSession := InitSession(false, prologue, staticRS, PubKey)
  responderSession := InitSession(false, prologue, staticRS, PubKey)
  
  payload := []byte{0x84, 0xa1, 0x76, 0xa8, 0x4e, 0x2d, 0x30, 0x30, 0x2d, 0x30, 0x30, 0x31, 0xa1, 0x74, 0xcb, 0x42, 0x79, 0x4b, 0x95, 0xae, 0x3f, 0x00, 0x00, 0xa1, 0x64, 0xcd, 0x97, 0xd6, 0xa1, 0x76, 0xa9, 0x7b, 0x22, 0x24, 0x6e, 0x65, 0x22, 0x3a, 0x30, 0x7d, 0xa1, 0x6d, 0xce, 0x01, 0xc0, 0x02, 0x25}
  // Send message and get updated session
  sessionPtr, messageBuffer1, err := SendMessage(&initiatorSession, payload)
  if err != nil {
    fmt.Println("Error in SendMessage:", err)
    return
  }
  initiatorSession = *sessionPtr

  buf.Write(messageBuffer1.ne[:])
  buf.Write(messageBuffer1.ns)
  buf.Write(messageBuffer1.ciphertext)

  // Print result and domain format
  fmt.Printf("%x\n", buf.Bytes())
  fillerX := "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
  
  encoded := base32.HexEncoding.EncodeToString(buf.Bytes())
  fmt.Printf("%s\n", encoded)
  if (len(encoded) < 63){
    fmt.Printf("x%s.x%s.x%s.net-x7yfcbnc.example.com.\n", strings.Replace(encoded[:len(encoded)], "=", "z", -1) + fillerX[:62-len(encoded)], fillerX, fillerX)
  } else if (len(encoded) < 125) {
    fmt.Printf("x%s.x%s.x%s.net-x7yfcbnc.example.com.\n", encoded[:62], strings.Replace(encoded[62:len(encoded)], "=", "z", -1) + fillerX[:124-len(encoded)], fillerX)
  } else if (len(encoded) < 187) {
    fmt.Printf("x%s.x%s.x%s.net-x7yfcbnc.example.com.\n", encoded[:62], encoded[62:124], strings.Replace(encoded[124:len(encoded)], "=", "z", -1) + fillerX[:186-len(encoded)])
  } else {
    fmt.Printf("Too long of a result")
  }
  buf.Reset()
  // Receive message and get updated session
  sessionPtr, plaintext, valid1, err := RecvMessage(&responderSession, &messageBuffer1)
  if err != nil || !valid1 {
    fmt.Printf("Handshake failed at responder: %s\n", err)
  }
  responderSession = *sessionPtr
  
  fmt.Printf("%x\n", plaintext)
 }
```

The resulting DNS query can then be sent with the command: `nslookup -port=1053 -type=A -retry=0 [domain_name] 127.0.0.1`