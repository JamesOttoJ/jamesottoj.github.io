---
layout: default
title: Task 5
parent: NSA Codebreakers 2023
nav_order: 6
---

# TASK 5
{: .no_toc}
- TOC
{:toc}

### Task Description:
>Based on the recovered hardware the device seems to to have an LTE modem and SIM card implying it connects to a GSM telecommunications network. It probably exfiltrates data using this network. Now that you can analyze the entire firmware image, determine where the device is sending data.
> 
>Analyze the firmware files and determine the IP address of the system where the device is sending data.

No files given, so only previous information is needed to solve this

### Ooh! New files
Decrypting the volume from the last task gives a couple of interesting files:
```
total 19944
drwxr-xr-x    3 root     0             4096 May 15  2022 .
drwxr-xr-x   21 root     0             4096 Jan  1 02:11 ..
-rwxr-xr-x    1 root     0           891224 May 15  2022 agent
-rw-r--r--    1 root     0                0 May 15  2022 agent_restart
-rw-r-----    1 root     0              567 May 15  2022 config
-rwx--x--x    1 root     0          7975035 May 15  2022 diagclient
-rwxr-xr-x    1 root     0         11483488 May 15  2022 dropper
drwx------    2 root     0            16384 May 15  2022 lost+found
-rwxrwx---    1 root     0              396 May 15  2022 start
```
To start, I looked at the file labled as such

Start:
``` sh
#!/bin/sh

DIR=/agent
PROC=agent
RESTART_FILE=agent_restart

# start the navigation service
/bin/nav

mkdir -p /tmp/upload
dmesg > /tmp/upload/boot_log_`date -Iseconds`

# start agent and restart if it exists
while [ 1 ]; do
    if [ ! -e $DIR/$RESTART_FILE ]; then
        break
    fi
    if [ -z "`ps -o comm | egrep ^${PROC}$`" ]; then
        $DIR/$PROC $DIR/config
    fi
    sleep 10
done
```

Breaking this down, it does a couple things:
1. It starts by setting a couple variabled for where programs are
2. The navigation service is started # TODO: find out more about this
3. A new directory called "/tmp/upload" is created if it doesn't exist, and a log file is put in
4. A continuous while loop is started
5. `[ ! -e $DIR/$RESTART_FILE ]` checks if the "agent_restart" file is in the above directory. If it isn't it breaks
6. ```[ -z "`ps -o comm | egrep ^${PROC}$`" ]``` does a lot, so these are each part
    - `-z` returns true if the folling string is empty
    - ``` `` ``` runs the script inside and will put the result in the string
    - `ps -o comm` outputs each of the commands being run for the current session
    - `|` takes the output of the previous command and sends it to the next command
    - `egrep ^${PROC}$` takes the list of commands and filters them with regex to check if a line only contains "agent"
    - Putting it together says that the next line will be run if there isn't a proccess that was just started with the command "agent"
7. If the above is true, the agent program is run using the config file, and it gets re-run every 10 seconds if it has stopped

Onto the config file, it defines a couple useful parameters for the program:

Config:
```shell
logfile = "/tmp/log.txt"

# levels 0 (trace) - 5 (fatal)
loglevel = 1

daemonize = true

id_file = "/private/id.txt"
ssh_priv_key = "/private/id_ed25519"
priv_key = "/private/ecc_p256_private.bin"

cmd_host = "0.0.0.0"
cmd_port = 9000

collectors_usb = [ "/dev/ttyUSB0", "/dev/ttyUSB1" ]
collectors_ipc = [ "/tmp/c1.unix", "/tmp/c2.unix" ]
collect_enabled = true
dropper_exe = "/agent/dropper"
dropper_config = "/tmp/dropper.yaml"
dropper_dir = "/tmp/upload"

navigate_ipc = "/tmp/nav_service.unix"

key_file = "/agent/hmac_key"
restart_flag = "/agent/agent_restart"
```
At this point, I was a little stuck, so I ran strings on the binaries to see if any of them had IPs hard coded. All of the IPs I found were worthless like 0.0.0.0 (gateway), 127.0.0.1 (localhost), or 169.254.169.254 (cloud metadata endpoint). After that, I looked more into the file names and found out that dropper is a common name in malware for a program that puts malicious code on the target system. Because we want to find the IP, I decided that the dropper program would be the best way to go.

### Dynamic Analysis

Looking into how to get data from the dropper program, I figured that it might be useful to let it run and look at `netstat` to find the IP it was reaching out to. This became a problem when running it because it needed a configuration file that agent didn't exist on the file system and wasn't created by agent. To see what would happen, I ran it and passed in a random file, but that led to an error about the file formatting. To get arround this, I just tried a non-existant file, and it happened to spit out the answer on the command line?
```
1970/01/01 02:48:26 Connecting to server...
1970/01/01 02:48:26 ...Connected!
1970/01/01 02:48:26 Loading/tmp/upload
1970/01/01 02:50:26 Processing/tmp/upload/boot_log_1970-01-01T00:11:53+00:00boot_log_1970-01-01T00:11:53+00:00
1970/01/01 02:50:56 Failed to insert into MongoDB:server selection error: server selection timeout, current topology: { Type: Unknown, Servers: [{ Addr: 100.105.36.5:27017, Type: Unknown, Last error: dial tcp 100.105.36.5:27017: connect: network is unreachable }, ] }
```
(While this is technically the answer, it is not enough information to move on)

### Static Analysis

After realizing that the configuration came from the file itself, I looked into how I could extract the data from the program. I realized that there was a program I have heard of, but hadn't tried yet: `binwalk`. `binwalk` is a program that is specifically made to extract different types of files inside of the main file. To do this, run `binwalk -Me /agent/dropper`. This extracts the zipped configuration file and unzips it to reveal:
```
database:
  collection: files
  database: snapshot-44180ec0a37b
  url: mongodb://maintenance:36599080632635@100.105.36.5:27017/?authSource=snapshot-44180ec0a37b
server:
  directory: /tmp/upload
```
This includes the information needed for task 6 to connect to the database