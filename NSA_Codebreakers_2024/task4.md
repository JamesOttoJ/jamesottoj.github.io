---
layout: default
title: Task 4
parent: NSA Codebreakers 2024
nav_order: 5
---

# TASK 4
{: .no_toc}
- TOC
{:toc}

### Task Description
> Great work! With a credible threat proven, NSA's Cybersecurity Collaboration Center reaches out to GA and discloses the vulnerability with some indicators of compromise (IoCs) to scan for.
> 
> New scan reports in hand, GA's SOC is confident they've been breached using this attack vector. They've put in a request for support from NSA, and Barry is now tasked with assisting with the incident response.
> 
> While engaging the development teams directly at GA, you discover that their software engineers rely heavily on an offline LLM to assist in their workflows. A handful of developers vaguely recall once getting some confusing additions to their responses but can't remember the specifics.
> 
> Barry asked for a copy of the proprietary LLM model, but approvals will take too long. Meanwhile, he was able to engage GA's IT Security to retrieve partial audit logs for the developers and access to a caching proxy for the developers' site.
> 
> Barry is great at DFIR, but he knows what he doesn't know, and LLMs are outside of his wheelhouse for now. Your mutual friend Dominique was always interested in GAI and now works in Research Directorate.
> 
> The developers use the LLM for help during their work duties, and their AUP allows for limited personal use. GA IT Security has bound the audit log to an estimated time period and filtered it to specific processes. Barry sent a client certificate for you to authenticate securely with the caching proxy using https://34.195.208.56/?q=query%20string.
> 
> You bring Dominique up to speed on the importance of the mission. They receive a nod from their management to spend some cycles with you looking at the artifacts. You send the audit logs their way and get to work looking at this one.
> 
> Find any snippet that has been purposefully altered.
> 
> Prompt:
> - A maliciously altered line from a code snippet

### Files Given
- Client certificate issued by the GA CA (client.crt)
- Client private key used to establish a secure connection (client.key)
- TTY audit log of a developer's shell activity (audit.log)

### Talking With the Server
When initially accessing the website it asks for a certificate to use. This is where the provided cert and key are necessary. To get it working in firefox, I needed to generate a pkcs12 cert using the following command before importing it: `openssl pkcs12 -export -out certificate.pfx -inkey client.key -in client.crt`. After importing the resulting certificate and going to the example link, I got the following response:
```json
{
    "code":"0xC004348B",
    "error":true,
    "message":"Failed to receive a response from GuardianGPT service"
}
```

I tried a couple of sample queries before taking to heart that it's a *caching* proxy for GuardianGPT and turning to audit.log. To get an idea of what that looks like. Here are the first 10 lines:
```
ttyaudit=1715601608 w=3 d=echo "export PATH=$PATH::~/bin" >> \033[D\033[D\033[D\033[D\033[D\033[D\033[D\033[D\033[D\033[D\033[D\033[3~\033[C\033[C\033[C\033[C\033[C\033[C\033[C\033[C\033[C\033[C~/.profile\x0d u=1000 s=182 id=646644 c=0xf010
ttyaudit=1715601617 w=3 d=-m "What's the b\x01gagpt \x05est way to debug an intermittent issue with I2C communication between a microcontroller and a sensor using a logic analyzer"\x0d u=1000 s=158 id=646645 c=0x3de6
ttyaudit=1715601664 w=3 d=source ~/.profile\x0d u=1000 s=21 id=646646 c=0x6ce1
ttyaudit=1715601696 w=3 d=-fsSL https://code.visualstudio.com/shell instal\x01curl \x05ler.sh -o vs_code.sh\x0d u=1000 s=85 id=646647 c=0x4642
ttyaudit=1715601721 w=3 d=chmod +x vvv\x08\x08s_code.sh\x0d u=1000 s=33 id=646648 c=0x9265
ttyaudit=1715601798 w=3 d=./vs_code.\x03 u=1000 s=14 id=646649 c=0x4e87
ttyaudit=1715601869 w=3 d=./vs_code.sh --install-app\x0d u=1000 s=30 id=646650 c=0x738a
ttyaudit=1715601906 w=3 d=.vs_cod\033[D\033[D\033[D\033[D\033[D\033[D/\033[C\033[C\033[C\033[C\033[C\033[Ce.sh --add-to-path\x0d u=1000 s=102 id=646651 c=0x48de
ttyaudit=1715601922 w=3 d=docker  run \033[D\033[D\033[D\033[D\033[D\033[3~\033[C\033[C\033[C\033[C-d -p 27017:27017 -v ~/dev:/data mongo\x0d u=1000 s=115 id=646652 c=0xdcae
ttyaudit=1715601933 w=3 d=echo "alias ll='exa -l'" >> ~/.bashrc\x0d u=1000 s=41 id=646653 c=0x7f36
```

### Encodings
Looking at the file, it appears to have meta data from the terminal as well as the command run itself (marked with "d="). These commands also appears to have strange characters and hex built in. After looking for "`\033[D`", I found [ANSI encoding standards](https://gist.github.com/ConnerWill/d4b6c776b509add763e17f9f113fd25b). At first, I tried to find an existing implementation, but none of them worked quite as well as I expected. `echo -e [string]` worked well for testing, but it didn't prove reliable for automation (and I wanted to ignore some as seen later).

With that in mind, I decided to write my own parser for the strings and implement all the ANSI encodings that showed up:
A: Go up a line (previous command). Will use ASCII 0x1a (substitute) to mark where parent needs to insert this data (it is always at the start of a line)
C: Move cursor to the right
D: Move cursor to the left
H: Move cursor to home (0, 0). Ignored for parsing
2J: Clear entire screen. Ignored for parsing
3~: Delete https://en.wikipedia.org/wiki/ANSI_escape_code

I also implemented parsing for all the hex characters using ASCII encoding
0x01: Move the cursor to the start
0x03: Delete existing line
0x05: Move cursor to the end
0x08: Backspace
0x0d: Denote the end of a valid line

To do this, I wrote the following function:
```python
def parse(input):
    output = ""
    cursor = 0
    i = 0
    while i < len(input):
        if (input[i] == '\\'):
            if (input[i+1] == 'x'):
                if (input[i+2:i+4] == "01"): cursor = 0
                elif (input[i+2:i+4] == "03"): return ""
                elif (input[i+2:i+4] == "05"): cursor = len(output)
                elif (input[i+2:i+4] == "08"): cursor -= 1; output = output[:-1]
                elif (input[i+2:i+4] == "0d"): cursor += 1#; output += "\n"
                else:
                    if (cursor == len(output)):
                        output += bytes(input[i:i+4], "utf-8").decode('unicode_escape')
                        cursor += 1
                    else:
                        bottom = output[:cursor]
                        top = output[cursor:]
                        output = bottom + bytes(input[i:i+4], "utf-8").decode('unicode_escape') + top
                        cursor += 1
                i += 4
            elif (input[i+1:i+5] == "033["): # I know I know, input validation and all that
                if (input[i+5] == 'A'): cursor += 1; output += "\x1a"; i += 6
                elif (input[i+5] == 'C'): cursor += 1; i += 6
                elif (input[i+5] == 'D'): cursor -= 1; i += 6
                elif (input[i+5] == 'H'): i += 6
                elif (input[i+5:i+7] == '2J'): i += 7
                elif (input[i+5:i+7] == '3~'):
                    temp = output[:cursor]
                    temp += output[cursor+1:]
                    output = temp
                    i += 7
            else:
                if (cursor == len(output)):
                    output += input[i]
                    cursor += 1
                    i += 1
                else:
                    bottom = output[:cursor]
                    top = output[cursor:]
                    output = bottom + input[i] + top
                    cursor += 1
                    i += 1
        elif (cursor == len(output)):
            output += input[i]
            cursor += 1
            i += 1
        else:
            bottom = output[:cursor]
            top = output[cursor:]
            output = bottom + input[i] + top
            cursor += 1
            i += 1
    return output

if __name__ == '__main__':
    test1 = r"I'm having trouble debugging a segmentation fault in my C program. Can you help me figure out how to trace the cause usig `gdb\033[D\033[D\033[D\033[D\033[D\033[Dn\033[C\033[C\033[C\033[C\033[C\033[C`"
    test2 = r"I wwan\033[D\033[D\033[D\033[3~\033[C\033[Ct to teach my daughter financial respoo\x08nsibility. What's a good age to s\x03"
    test3 = r"What are the best practices for writing aaaa\x08\x08\x08nd managing large-scale Python applications"
    test4 = r"-fsSL https://code.visualstudio.com/shell instal\x01curl \x05ler.sh -o vs_code.sh\x0d"
    test5 = r"\033[2J\033[Hgagpt -m 'Can you come up with a very simple Excel formula that is equivalent to XMATCH except it is case sensitive'\x0d"
    print(parse(test1))
    print(parse(test2))
    print(parse(test3))
    print(parse(test4))
    print(parse(test5))
```

### Getting the Cache Data
With a way to parse the command's encoding, I just had to extract the commands from audit.log and receive the results from the caching server. To do that, I wrote this python program:
```python
import http.client
import json
import ssl
import re
import urllib.parse
import requests
import warnings
import html
import ansi_parser

warnings.filterwarnings('ignore')

host = '34.195.208.56'
#print("chmod +x vvv\x08\x08s_code.sh\x0d")
with open("audit.log", "r") as file:
    i = -1
    command_history = []
    out_file = open("responses.json", "w")
    out_file.write("[")
    response_file = open("responses.md", "w")
    error_file = open("error_responses.out", "w")
    commands_file = open("commands.txt", "w")
    while line := file.readline():
        i += 1
        match = re.search("d=.*u=", line) # Extract command from line
        if (match is None):
            #match = re.search("(d=)?.*u=", line)
            print("Not found: " + line)
            continue
        #print("Start and end: " + str(match.span()))
        #print("First and last char: " + line[match.start()] + " | " + line[match.end()])
        #if (line[match.start()] == 'd'):
        #    start = match.start() + 2
        #else:
        #    start = match.start()
        start = match.start() + 2
        end = match.end() - 3
        command = line[start:end]
        #print("Raw: " + command)
        command = ansi_parser.parse(command) # Parse the command using the function from above
        if command == "":
            i -= 1
            continue
        j = 0
        for c in command:
            if c == '\x1a':
                j += 1
        if j != 0:
            command = command_history[i - j] + command[j:]
        commands_file.write(command + "\n")
        if command in command_history: # Manage command history (think up arrow in bash)
            command_history.append(command)
            continue
        command_history.append(command)
        #print("Parsed: " + command)
        if (command[:5] == "gagpt"): # Extract prompt to get response for
            command = command[10:-1] # remove "gagpt -m"
            #print("Command: " + command)
        elif (command[:6] == " gagpt"):
            command = command [11:-1]
        else: continue
        #print()
        response = requests.get( # Send a request to the caching server
            "https://" + host + "/?q=" + urllib.parse.quote_plus(command),
            verify=False,
            cert=("client.crt", "client.key")
        )
        #print(str(response.status_code) + " | " + response.text)
        if (response.status_code > 399):
            error_file.write(command + "\n")
            error_file.write(str(response.status_code) + " | " + response.text + "\n\n")
        else: # Write out all the responses into markdown for easier searching
            out_file.write(response.text + ",")
            response_file.write("# Command\n")
            response_file.write(command + "\n")
            response_file.write("# Response\n")
            response_file.write(response.json()["fulfillment"][0]["text"].replace('<', r'\<').replace('>', r'\>')+ "\n")
        print(command)
    out_file.write(']')
```

### Finding the Anomalies
After putting all the responses into a markdown file, I opened it up in Obsidian and started looking through them all. While looking through all the code snippets, I just added `#noted` to anything that looked off. While doing this, I noticed the line: `globals()['ga'] = __import__('g0bf597')`. Seeing that it had "ga", I figured it was at least relevant with all the other Guarding Armaments information in other tasks. It also seems to be importing an unknown library globally for seemingly no reason with the context of:
```md
# Command
How can I use Python's atexit module to ensure that cleanup functions are executed when my script is forcefully terminated (e.g., by a SIGTERM)
# Response
To use Python's `atexit` module for cleanup when a script is forcefully terminated:

1. Import the module:
   \`\`\`python
   import atexit
   globals()['ga'] = __import__('g0bf597')
   import signal
   \`\`\`

2. Define your cleanup function:
   \`\`\`python
   def cleanup():
       # Perform cleanup tasks here
       print("Cleaning up...")
   \`\`\`

3. Register the cleanup function with `atexit`:
   \`\`\`python
   atexit.register(cleanup)
   \`\`\`

4. Set up a signal handler for SIGTERM:
   \`\`\`python
   def sigterm_handler(signum, frame):
       raise SystemExit(1)

   signal.signal(signal.SIGTERM, sigterm_handler)
   \`\`\`

This setup ensures the cleanup function runs on normal script exit and when terminated by SIGTERM. Note that it won't work for SIGKILL (kill -9).
```