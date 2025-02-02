---
layout: default
title: Task 3
parent: NSA Codebreakers 2024
nav_order: 4
---

# TASK 3
{: .no_toc}
- TOC
{:toc}

### Task Description
>  Great work finding those files! Barry shares the files you extracted with the blue team who share it back to Aaliyah and her team. As a first step, she ran strings across all the files found and noticed a reference to a known DIB, “Guardian Armaments” She begins connecting some dots and wonders if there is a connection between the software and the hardware tokens. But what is it used for and is there a viable threat to Guardian Armaments (GA)?
> 
> She knows the Malware Reverse Engineers are experts at taking software apart and figuring out what it's doing. Aaliyah reaches out to them and keeps you in the loop. Looking at the email, you realize your friend Ceylan is touring on that team! She is on her first tour of the Computer Network Operations Development Program
> 
> Barry opens up a group chat with three of you. He wants to see the outcome of the work you two have already contributed to. Ceylan shares her screen with you as she begins to reverse the software. You and Barry grab some coffee and knuckle down to help.
> 
> Figure out how the APT would use this software to their benefit
> 
> Prompt:
> - Enter a valid JSON that contains the (3 interesting) keys and specific values that would have been logged if you had successfully leveraged the running software. Do ALL your work in lower case.

### Files Given
- Executable from ZFS filesystem (server)
- Retrieved from the facility, could be important? (shredded.jpg)

### Static Analysis
Starting with `file` on the executable, it shows: `server: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, Go BuildID=FYmEDItHIsRHHIq98wlB/a0VGthTc36DMaPB42oqd/Z80VF_UGQE1u9089Ef4n/roe9PgJFthb0PSZ3SK_n, with debug_info, not stripped`. Of note from this, it's an x86-64 executable written in Go with symbols left in. This means it is time to open the program in Ghidra. 

Looking at the available main functions/classes, there are a few notable ones to me:
- `main.main`: Go's real main function and where the code starts from
- `main.NewSeedgenAuthClient`: Looks like some kind of initialization for an auth client
- `main.(*SeedgenAuthClient).auth`: Implementation of the SeedgenAuthClient class' auth function
- `main.(*seedGenerationServer).GetSeed`: A function to receive a seed from some kind of seed generation server
- `main.(*seedGenerationServer).Ping`: A function to test connection availability to the seed generation server

Looking at the main function
```c
...
  fmt.Fprintf(0x107,0,param_3,
              "Starting the Guardian Armaments OTP seed generation service!  Please ensure that this  software can reach the authentication service to register any generated seeds!  Other wise your token will not authenticate you to the network after you program it with thi s seed"
              ,0,0);
  if (os.Args.len == 0) {
    runtime.panicSliceB();
    runtime.deferreturn();
    return;
  }
  flag.(*FlagSet).Parse
            (os.Args.cap + -1,(uint)(-(os.Args.cap + -1) >> 0x3f) & 0x10,os.Args.array,
             os.Args.len + -1);
  if (main.authServerIP.len == 0) {
    local_f0._8_8_ = &PTR_s_Please_provide_the_IP_address_of_0095c9f0;
    local_f0._0_8_ = &DAT_008075e0;
    fmt.Fprintln(1,1,&DAT_008075e0,local_f0);
    os.Exit();
  }
  if (main.logLevel.len == 4) {
    if ((*(int *)main.logLevel.str == 0x6f666e69) || (*(int *)main.logLevel.str == 0x6e726177))
    goto LAB_007d0712;
  }
  else if ((main.logLevel.len == 5) &&
          (((*(int *)main.logLevel.str == 0x75626564 && (main.logLevel.str[4] == 0x67)) ||
           ((*(int *)main.logLevel.str == 0x6f727265 && (main.logLevel.str[4] == 0x72))))))
...
grpcClient = google.golang.org/grpc.NewClient(1,1);
...
  auth_client = main.NewSeedgenAuthClient();
  local_130 = uVar8;
  puVar3 = (undefined8 *)runtime.newobject();
  *puVar3 = auth_client;
  puVar3[2] = uVar6;
  puVar3[3] = uVar9;
  if (runtime.writeBarrier._0_4_ != 0) {
    puVar3 = (undefined8 *)runtime.gcWriteBarrier1();
    *puVar10 = local_130;
  }
  puVar3[1] = local_130;
  dVar11 = (double)otp/seedgen.RegisterSeedGenerationServiceServer(puVar3);
  log/slog.(*Logger).log(dVar11);
  uVar8 = local_118;
  lVar4 = google.golang.org/grpc.(*Server).Serve();
...
```

The most notable parts I took from the main function were:
- Arguments
    - An IP address to use
    - Log level (info, debug, or error)
- Auth client is a gRPC client
- Auth server is a gRPC server

Moving onto `main.NewSeedgenAuthClient`, it calls 3 notable functions: `math/rand.Seed()`, `auth/auth_grpc.(*authServiceClient).Ping()`, and `math/rand.int63()`. Starting with seeing rand, it is called with `0x1d51d5abed24c` as an argument by putting it into RAX, but Ghidra doesn't seem to parse Go arguments properly. Afterward, it calls ping to check in with the authentication server. Finally, it generates a random number (this gets used later).

Lastly of this bunch, the `otp/seedgen.RegisterSeedGenerationServiceServer()` function. This essentially registers all the gRPC API endpoints for the seedGenerationServer class (Ping, StressTest, and GetSeed).

Focusing on the `GetSeed` function of those, it makes a call to `main.(*SeedgenAuthClient).auth` and uses the output to log and return some data. Diving into the auth function, it appeared that it would just take the data it was given and pass it to the auth server, but I noticed that it was doing extra processing and comparing the data to a random-looking hex value. The decompiled content itself wasn't much help, so I decided to look at the assembly.
```assembly
                             LAB_007cf602                                    XREF[2]:     007cf5d6(j), 007cf5e3(j)  
        007cf602 4c 8b a4        MOV        R12,qword ptr [RSP + username_spill.str]
                 24 b0 00 
                 00 00
        007cf60a 48 8b 8c        MOV        RCX,qword ptr [RSP + username_spill.len]
                 24 b8 00 
                 00 00
        007cf612 48 8b 54        MOV        RDX,qword ptr [RSP + local_58]
                 24 48
        007cf617 31 c0           XOR        EAX,EAX
        007cf619 eb 06           JMP        LAB_007cf621
                             LAB_007cf61b                                    XREF[5]:     007cf66d(j), 007cf68e(j), 
                                                                                          007cf6b1(j), 007cf6f6(j), 
                                                                                          007cf700(j)  
        007cf61b 44 31 fa        XOR        EDX,R15D
        007cf61e 4c 89 e8        MOV        RAX,R13
                             LAB_007cf621                                    XREF[1]:     007cf619(j)  
        007cf621 48 39 c1        CMP        RCX,RAX
        007cf624 0f 8e db        JLE        authenticating_log
                 00 00 00
        007cf62a 4c 8d 68 04     LEA        R13,[RAX + 0x4]
        007cf62e 4c 39 e9        CMP        RCX,R13
        007cf631 7c 3c           JL         LAB_007cf66f
        007cf633 48 39 c1        CMP        RCX,RAX
        007cf636 0f 86 e4        JBE        LAB_007cf920
                 02 00 00
        007cf63c 4c 8d 78 01     LEA        R15,[RAX + 0x1]
        007cf640 4c 39 f9        CMP        RCX,R15
        007cf643 0f 86 cc        JBE        LAB_007cf915
                 02 00 00
        007cf649 4c 8d 78 02     LEA        R15,[RAX + 0x2]
        007cf64d 4c 39 f9        CMP        RCX,R15
        007cf650 0f 86 b7        JBE        LAB_007cf90d
                 02 00 00
        007cf656 4c 8d 78 03     LEA        R15,[RAX + 0x3]
        007cf65a 66 0f 1f        NOP        word ptr [RAX + RAX*0x1]
                 44 00 00
        007cf660 4c 39 f9        CMP        RCX,R15
        007cf663 0f 86 9c        JBE        LAB_007cf905
                 02 00 00
        007cf669 45 8b 3c 04     MOV        R15D,dword ptr [R12 + RAX*0x1]
        007cf66d eb ac           JMP        LAB_007cf61b
```

While assembly is intimidating at first, this one does something quite simple. It starts by loading the username data address into R12, the username length address into rcx, and a mystery variable's address into RDX. After that, it clears the data in RAX and jumps into a loop. This loop starts by comparing the username length to RAX and jumping to the next section if the username length is less than or equal to RAX. After that, it checks if the username length is less than 4 more than RAX and jumps away. It continues to do this with each offset (RAX+1, RAX+2, RAX+3, RAX+4) before moving the next 4-byte chunk into R15 and XORing it with the mystery variable. To make it easier to understand, here it is in Python pseudocode:

```python
i = 0
result = mysterious_var
for i < username.len:
    temp = ""
    if username.len < i + 4:
        # Extrapolated from area it jumps to
        temp = username.str[i:username.len]
    elif username.len == i + 1:
        Panic!
    elif username.len == i + 2:
        Panic!
    elif username.len == i + 3:
        Panic!
    else:
        temp = username.str[i:i+4]
    result ^= temp

# authenticating_log (What the assembly does after the JLE at 0x007cf624)
if result == 0x8b16bd5d:
    print("test user authenticated, but has no privileges in network so no need to authenticate with Auth Service!")
    return username, password, new_seed
else:
    print("Authenticating with auth service")
    if auth_grpc.AuthRequest() == True:
        return username, password, new_seed
    else:
        print("Failed to authenticate client with service")
        return NULL
```

Because of Go's runtime, static analysis can only go so far. There will be instructions like `NOP dword ptr [RAX + RAX*0x1]`, and it will often be hard to track where the data is going. To make it easier to analyze at this point, I found places where I wanted to know what the state of the program was and set a breakpoint in GDB.

#### Interacting with the program
When first running a program, it's always worth trying it with the `-h` or `--help` arg. Running `server --help` gave this messsage:
```bash
┌─[jamesj@parrot]─[~/Documents/codebreaker_2024/task3]
└──╼ $./server --help
Starting the Guardian Armaments OTP seed generation service!  Please ensure that this software can reach the authentication service to register any generated seeds!  Otherwise your token will not authenticate you to the network after you program it with this seedUsage of ./server:
  -auth-ip string
    	Set the IP address of the auth server (default "127.0.0.1")
  -loglevel string
    	Set the logging level (debug, info, warn, error) (default "info")
```

Looking at these arguments, I figured keeping the IP to localhost would be best because that means I can act as the auth server. For the log level, I always like to set it to debug when possible. This will often leak extra information during startup, and it will help with finding the errors in your payload.

Just running the program with debug logs gives:
```bash
┌─[jamesj@parrot]─[~/Documents/codebreaker_2024/task3]
└──╼ $./server -loglevel debug
Starting the Guardian Armaments OTP seed generation service!  Please ensure that this software can reach the authentication service to register any generated seeds!  Otherwise your token will not authenticate you to the network after you program it with this seed{"time":"2025-01-25T08:46:09.235210649-06:00","level":"INFO","msg":"Connected to auth server"}
{"time":"2025-01-25T08:46:09.235712108-06:00","level":"ERROR","msg":"Failed to ping the auth service","ping_response":null,"err":"rpc error: code = Unavailable desc = connection error: desc = \"transport: Error while dialing: dial tcp 127.0.0.1:50052: connect: connection refused\""}
```

Looks like it's trying to ping a service on port 50052. In case it was just checking if there was anything up at all, I opened a netcat listener with `nc -lnvp 50052`. This changed the message to:
```bash
Starting the Guardian Armaments OTP seed generation service!  Please ensure that this software can reach the authentication service to register any generated seeds!  Otherwise your token will not authenticate you to the network after you program it with this seed{"time":"2025-01-25T08:48:35.334606264-06:00","level":"INFO","msg":"Connected to auth server"}
{"time":"2025-01-25T08:48:55.35009975-06:00","level":"ERROR","msg":"Failed to ping the auth service","ping_response":null,"err":"rpc error: code = Unavailable desc = connection error: desc = \"error reading server preface: read tcp 127.0.0.1:57528->127.0.0.1:50052: use of closed network connection\""}
```
The netcat listener received the following from the application:
```bash
listening on [any] 50052 ...
connect to [127.0.0.1] from (UNKNOWN) [127.0.0.1] 57528
PRI * HTTP/2.0

SM

```

This told me that I needed some kind of http 2.0 server listening on that port. Thinking back to the disassembled code, I started looking into the gRPC protocol to see if it would help. After looking into the protocol, I realized that all gPRC calls use HTTP 2.0 and protobufs. These protobufs provide a format for what requests/responses are available and what to send/receive in a request/response. After more digging, I found a program to extract protobufs from compiled go programs called [protodump](https://github.com/arkadiyt/protodump).

After extraction, there were two notable protobufs: auth.proto and seed_generation.proto.
**auth.proto**:
```go
syntax = "proto3";

package auth_service;

option go_package = "./auth_service";

service AuthService {
  rpc Authenticate (.auth_service.AuthRequest) returns (.auth_service.AuthResponse) {}
  rpc RegisterOTPSeed (.auth_service.RegisterOTPSeedRequest) returns (.auth_service.RegisterOTPSeedResponse) {}
  rpc VerifyOTP (.auth_service.VerifyOTPRequest) returns (.auth_service.VerifyOTPResponse) {}
  rpc RefreshToken (.auth_service.RefreshTokenRequest) returns (.auth_service.RefreshTokenResponse) {}
  rpc Logout (.auth_service.LogoutRequest) returns (.auth_service.LogoutResponse) {}
  rpc Ping (.auth_service.PingRequest) returns (.auth_service.PingResponse) {}
}

message AuthRequest {
  string username = 1;
  string password = 2;
}

message AuthResponse {
  bool success = 1;
}

message RegisterOTPSeedRequest {
  string username = 1;
  int64 seed = 2;
}

message RegisterOTPSeedResponse {
  bool success = 1;
}

message VerifyOTPRequest {
  string username = 1;
  int64 otp = 2;
}

message VerifyOTPResponse {
  bool success = 1;
  string token = 2;
}

message RefreshTokenRequest {
  string token = 1;
}

message RefreshTokenResponse {
  string token = 1;
}

message LogoutRequest {
  string token = 1;
}

message LogoutResponse {
  bool success = 1;
}

message PingRequest {
  int64 ping = 1;
}

message PingResponse {
  int64 response = 1;
}
```

**seed_generation.proto**
```go
syntax = "proto3";

package seed_generation;

option go_package = "./seedgen";

service SeedGenerationService {
  rpc GetSeed (.seed_generation.GetSeedRequest) returns (.seed_generation.GetSeedResponse) {}
  rpc Ping (.seed_generation.PingRequest) returns (.seed_generation.PingResponse) {}
  rpc StressTest (.seed_generation.StressTestRequest) returns (.seed_generation.StressTestResponse) {}
  rpc testEmbeddedByValue (.seed_generation.Empty) returns (.seed_generation.Empty) {}
}

message GetSeedRequest {
  string username = 1;
  string password = 2;
}

message GetSeedResponse {
  int64 seed = 1;
  int64 count = 2;
}

message StressTestRequest {
  int64 count = 1;
}

message StressTestResponse {
  int64 response = 1;
}

message PingRequest {
  int64 ping = 1;
}

message PingResponse {
  int64 response = 1;
}

message Empty {
}
```

Looking at the above protobufs, auth.proto looks to be related to the SeedgenAuthClient class, and seed_generation.proto appears to be related to the seedGenerationServer class. This means that we can now try and interact with them.

To start, I noticed that the server tries reaching out first. The client usually reaches out in a client/server relationship, so I figured that auth.proto would be most useful here. Next, I noticed that the error message from earlier said that it, "Failed to ping the auth service". This means that the gRPC server will need to receive and handle the request using the Ping service in auth.proto. I first did this in python (see my codebreakers_2024 repo for more), but I decided to switch to go for efficiency and coherence sake. For help in implementing this, I would recommend their [examples](https://github.com/grpc/grpc-go/tree/master/examples) and [quick start guide](https://grpc.io/docs/languages/go/quickstart/). My finished auth server looked like this:
```go
package main

import (
	"context"
    "flag"
    "net"
    "fmt"
    "log"

	"google.golang.org/grpc"
//	"google.golang.org/grpc/codes"
//	"google.golang.org/grpc/status"

//	grpc_auth "github.com/grpc-ecosystem/go-grpc-middleware/auth"
//	grpc_ctxtags "github.com/grpc-ecosystem/go-grpc-middleware/tags"
    pb "codebreakers/proto/auth_service"
)

var (
	port = flag.Int("port", 50052, "The server port")
)

type server struct {
	pb.UnimplementedAuthServiceServer
}

func (s *server) Ping(_ context.Context, in *pb.PingRequest) (*pb.PingResponse, error) {
//	log.Printf("Received: Ping")
    return &pb.PingResponse{Response: 8765}, nil
}

func main() {
	flag.Parse()
	lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	s := grpc.NewServer()
	pb.RegisterAuthServiceServer(s, &server{})
	log.Printf("server listening at %v", lis.Addr())
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

After putting this in place and running the server, the error from the original program output turned into:
```bash
...
{"time":"2025-01-25T16:41:08.7000501-06:00","level":"DEBUG","msg":"Auth Service Pong ","pong":8765}
{"time":"2025-01-25T16:41:08.700068748-06:00","level":"INFO","msg":"Seedgen Server running on port 50051"}
```

Now that the server is listening for connections, I turned to the seed_generation protobuf. Of those, the GetSeed service seemed the most interesting because it appeared to be the main function of the program. To do this, I wrote a gRPC client as follows:
```go
package main

import (
	"context"
	"flag"
	"log"
	"time"
    "runtime"

    "github.com/zenthangplus/goccm"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	pb "codebreakers/proto/seed_generation"
)

var (
	addr = flag.String("addr", "localhost:50051", "the address to connect to")
)


func main() {
	flag.Parse()
	// Set up a connection to the server.
	conn, err := grpc.NewClient(*addr, grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()
	c := pb.NewSeedGenerationServiceClient(conn)

    ctx, cancel := context.WithTimeout(context.Background(), time.Minute)
	defer cancel()
    r, err := c.GetSeed(ctx, &pb.GetSeedRequest{Username: "test", Password: ""})
	if err != nil {
		log.Fatalf("could not greet: %v", err)
	}
    log.Printf("Seed: %x | Count: %x", r.GetSeed(), r.GetCount())
}
```

This gave the result: `2025/01/25 16:52:28 Seed: 2188fa513e4f1a02 | Count: 1`
The server reported: 
```bash
{"time":"2025-01-25T16:52:28.513744712-06:00","level":"DEBUG","msg":"got a GetSeed","username":"test","password":""}
{"time":"2025-01-25T16:52:28.513769785-06:00","level":"INFO","msg":"Authenticating","username":"test","password":""}
{"time":"2025-01-25T16:52:28.513775931-06:00","level":"DEBUG","msg":"test user authenticated, but has no privileges in network so no need to authenticate with Auth Service!"}
{"time":"2025-01-25T16:52:28.513782426-06:00","level":"INFO","msg":"Registered OTP seed with authentication service","username":"test","seed":2416456426928937474,"count":1}
```
Trying it again, however, gave:
```bash
{"time":"2025-01-25T16:55:59.872874488-06:00","level":"INFO","msg":"Authenticating","username":"test","password":""}
{"time":"2025-01-25T16:55:59.872880564-06:00","level":"DEBUG","msg":"Authenticating with auth service"}
{"time":"2025-01-25T16:55:59.874033363-06:00","level":"ERROR","msg":"Failed to authenticate client with service"}
{"time":"2025-01-25T16:55:59.874050404-06:00","level":"WARN","msg":"failed to authenticate user","username":"test","password":""}
```

With that, I decided to add the auth service to my server followed by the RegisterOTPSeed service after another error. This gave the following program that successfully handled all GetSeed calls no matter the credentials:
```go
package main

import (
	"context"
    "flag"
    "net"
    "fmt"
    "log"

	"google.golang.org/grpc"
//	"google.golang.org/grpc/codes"
//	"google.golang.org/grpc/status"

//	grpc_auth "github.com/grpc-ecosystem/go-grpc-middleware/auth"
//	grpc_ctxtags "github.com/grpc-ecosystem/go-grpc-middleware/tags"
    pb "codebreakers/proto/auth_service"
)

var (
	port = flag.Int("port", 50052, "The server port")
)

type server struct {
	pb.UnimplementedAuthServiceServer
}

func (s *server) Authenticate(_ context.Context, in *pb.AuthRequest) (*pb.AuthResponse, error) {
//	log.Printf("Received: Authenticate")
    return &pb.AuthResponse{Success: true}, nil
}

func (s *server) RegisterOTPSeed(_ context.Context, in *pb.RegisterOTPSeedRequest) (*pb.RegisterOTPSeedResponse, error) {
//	log.Printf("Received: Register Seed")
//    if (in.GetCount() % 1000000 == 0) { log.Printf("Rcv seed count: %d\n", in.GetCount()) }
    return &pb.RegisterOTPSeedResponse{Success: true}, nil
}

func (s *server) Ping(_ context.Context, in *pb.PingRequest) (*pb.PingResponse, error) {
//	log.Printf("Received: Ping")
    return &pb.PingResponse{Response: 8765}, nil
}

func main() {
	flag.Parse()
	lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	s := grpc.NewServer()
	pb.RegisterAuthServiceServer(s, &server{})
	log.Printf("server listening at %v", lis.Addr())
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

Now that I had it working, I wondered why that first request worked, so I started dynamic analysis

### Dynamic Analysis
Looking back at the code, I found the "test user authenticated, but has no privileges in network so no need to authenticate with Auth Service!" string being used in the auth function, and I traced it back to the assembly snippet that was analyzed earlier. I set a breakpoint at the beginning and waited for it to be hit after the GetSeed request. Once the breakpoint was hit, I followed the code and saw that the mysterious variable was some random sequence of 4 bytes that got XORed with "test" to get the 0x8b16bd5d value it looks for.

After seeing why "test" passed, I tried it again to see why it didn't pass. I noticed that the mysterious variable was both different and recognizable. It was the least significant four bytes of the first seed: 0x3e4f1a02. Thinking back to the code again, I remembered that the random was seeded, so I made a quick, test Go program to see if the random values lined up:
```go
package main

import (
  "math/rand"
  "log"
)

func main() {
  rand.Seed(515797029933644) // 0x1d51d5abed24c
  log.Printf("%x\n", rand.Int63())
  log.Printf("%x\n", rand.Int63())
  log.Printf("%x\n", rand.Int63())
}
```
Which resulted in:
```
2025/01/25 17:15:15 6a73fad3ff65d829
2025/01/25 17:15:15 2188fa513e4f1a02
2025/01/25 17:15:15 5e2653083382a521
```

### Finding the vulnerability
The seed means that using the username "test" in the first request will work every time. It also means that there is a series of bytes in each request that will make the server skip authentication. To determine what this means, it's time to look at the other file. shredded.jpg contains a shredded piece of paper with "jasper_0" on it. Looking back to task 1, the record out of place had a user with the name "Jasper Wright" and an email of "jasper_05038@guard.ar". Emails are made up of a username and domain in the format "username@domain", so I figured that it was worth "jasper_05038" as the username in multiple GetSeed requests until it bypassed the authentication.

As my cryptography professor always said, 4 bytes is easy to brute force. Because random numbers naturally spread out evenly over time, a random choice is just as brute force-able as going through the options methodically. This will also produce three important values: a username, seed, and count like the prompt wants.

### Exploiting
When I first tried exploiting it, I send a request for the sever for each attempt. This was only giving me around 10,000 requests every minute and no results after 12 hours, so I switched to emulating the process. Once I did this in go using the following program, it only took around 5 minutes:
```go
package main

import (
  "math/rand"
  "log"
)

func main() {
  rand.Seed(515797029933644) // 0x1d51d5abed24c
  oldSeed := rand.Int63() & 0xffffffff
  count := 0
  for true {
    //    temp := 0x74736574 ^ oldSeed // "temp" little end
    temp := 0x7073616a ^ oldSeed // "jasp" little end
    temp = 0x305f7265 ^ temp // "er_0" little end
    temp = 0x38333035 ^ temp // "5038" little end
    count = count + 1
    seed := rand.Int63()
//    log.Printf("%x | %x | %d | %d\n", temp, oldSeed, seed, count)
//    break
    if (temp == 0x8b16bd5d) { log.Printf("Success! {\"username\":\"jasper_05038\",\"seed\":%d,\"count\":%d}\n", seed, count); break }
    if (count % 1000000 == 0) { log.Printf("Count: %d\n", count) }
    oldSeed = seed & 0xffffffff
  }
}
```
With this program, there are two important things to note:
1. The username gets broken into sequential 4-byte chunks, but each chunk is stored in little-endian. This means that "jasper_05038" gets turned into three chunks that look like: "psaj", "0_re", and "8305" in that order for XORing
2. An initial seed is generated in main.NewSeedgenAuthClient() and used for the first auth request. After that, a new number is generated (the seed) and returned to the user. That seed is then used for the next request in the XOR process. This means that the value which successfully produces 0x8b16bd5d after getting XORed with the username is the the seed from the request *before* the successful request.

The successful result was: `{"username":"jasper_05038","seed":2797860527612852619,"count":3456434966}`