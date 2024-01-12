# Collection of Raw Notes

- [Task 1](#task-1)
- [Task 2](#task-2)
- [Task 3](#task-3)
- [Task 4](#task-4)
- [Task 5](#task-5)
- [Task 6](#task-6)

### TASK 1

To get tables:
`.tables`

To get column names:
`.schema tableName`

Found the database, need matching reccords:
sqlite> `SELECT * FROM location WHERE ABS(latitude-28.52) < 0.01;`
874|28.5293|-91.63676|0 m
sqlite> `SELECT * FROM location WHERE ABS(longitude+91.63) < 0.01;`
874|28.5293|-91.63676|0 m

location ID: 874

sqlite> SELECT \* FROM event WHERE location_id=874;
id|location_id|name|audio_object_id|timestamp_id
874|874|a48fea3f980821c656598ec0cadf9c6aed2fcdaeb4297d656d9cdd4de1a3a703|874|874

sqlite> SELECT \* FROM timestamp WHERE recDate='02/08/2023';
id|recTime|recDate
746|00:27:17|02/08/2023
874|04:16:31|02/08/2023
969|04:18:53|02/08/2023

sqlite> SELECT \* FROM location WHERE ABS(latitude-28.52) < 0.03;
874|28.5293|-91.63676|0 m
969|28.53213|-91.64326|0 m
sqlite> SELECT \* FROM location WHERE ABS(longitude+91.63) < 0.03;
874|28.5293|-91.63676|0 m
969|28.53213|-91.64326|0 m

sqlite> SELECT \* FROM timestamp WHERE id=969 OR id=874;
874|04:16:31|02/08/2023
969|04:18:53|02/08/2023


### TASK 2

Method:

1. start by looking up text on the datasheet
2. See that Ras Pi 3 b has a BCM2837
   1. Look at how Ras Pi is dealing with GPIO pinouts
   2. What's the difference between 3.3v and 5v? 3.3 is more controlled power
   3. Ground is ground, any will work
3. Look up "BCM2837 datasheet", you'll find a PDF
4. In that PDF, look for information about GPIO pins
5. You'll find a chart with what each GPIO pin is for
6. Which column to use? (This is where the boot log comes in)
7. You can see that the header has default and ALT1-5, the boot shows ALT5
8. In the legend underneath, we can look for which ones send and receive over UART (usually Tx for transmit and Rx for receive)
9. Look for the pins set to Tx and Rx in the datasheet (this is the number on the pins labeled as GPIO in the pinout)
10. Count which physical pin for the 3.3v, ground, Tx, and Rx. submit it as P#
    https://usermanual.wiki/Datasheet/BCM2837ARMPeripheralsBroadcom.1054296467

### TASK 3

env variables of interest:
ivaddr=467a0010
kernel_addr_r=40400000
keyaddr=467a0000

use md to see what's the address. Keep in mind that each _block_ is little endian
so, a result of 00112233 44556677 from `md 0` is:
0x0: 33
0x1: 22
0x2: 11
0x3: 00
0x4: 77
0x5: 66
0x6: 55
0x7: 44
to get it in the right order, use `md.b address` to look at bytes
`md.b 0` would show up as: "33 22 11 00 77 66 55 44" from before
key at key addr: f0eced4c9ef21abec6403b00c44adfb3

### TASK 4

Want to find: encryption password

Pop into basic linux shell as root

four "devices" of interest
/dev/mmcblk0p1 : Mounted to the (/) root directory and is on the SD card
/dev/mmcblk0p2 : Mounted to the /boot directory and is on the SD card
/dev/sda1 :
On the USB
mounted to /opt
seems to hold the encrypted data we want
/dev/sda2 :
On the usb
mounted to /private
seems to hold the private keys for encryption

(found via `fdisk -l` and `mount`)

private:
total 40
drwxr-xr-x 3 root 0 4096 May 15 2022 .
drwxr-xr-x 21 root 0 4096 Jan 1 00:01 ..
-rw------- 1 root 0 96 May 15 2022 ecc_p256_private.bin
-rw------- 1 root 0 64 May 15 2022 ecc_p256_pub.bin
-rw------- 1 root 0 36 May 15 2022 id.txt
-rw------- 1 root 0 387 May 15 2022 id_ed25519
drw------- 2 root 0 16384 May 15 2022 lost+found (empty)

opt:
total 28740
drwxr-xr-x 4 root 0 4096 May 15 2022 .
drwxr-xr-x 21 root 0 4096 Jan 1 00:01 ..
drwx------ 2 root 0 4096 May 15 2022 .ssh (empty)
-rw-r--r-- 1 root 0 11 May 15 2022 hostname
drwx------ 2 root 0 16384 May 15 2022 lost+found (empty)
-rwxrwx--- 1 root 0 443 May 15 2022 mount_part
-rw-r--r-- 1 root 0 29360128 May 15 2022 part.enc

mount_part

```
#!/bin/sh

SEC_DRIVE=$1
SEC_MOUNT=$2
ENC_PARTITION=$3
ENC_MOUNT=$4

[ ! -e $ENC_PARTITION ] && { echo "encrypted partition not found"; exit 1; }

mkdir -p $SEC_MOUNT
mount $SEC_DRIVE $SEC_MOUNT
NAME=`hostname`
ID=`cat /private/id.txt`

DATA="${NAME}${ID:0:3}"
echo "cryptsetup: opening $ENC_PARTITION"
echo -n $DATA | openssl sha1 | awk '{print $NF}' | cryptsetup open $ENC_PARTITION part
mkdir -p $ENC_MOUNT
mount /dev/mapper/part $ENC_MOUNT
```

SEC_DRIVE=/dev/sda2
SEC_MOUNT=/private
ENC_PARTITION=/opt/part.enc
ENC_MOUNT=/agent

`cryptsetup open /opt/part.enc part` asks for key input

hashcat --stdout -a 6 -1?l?u?d ./hostname.txt ?1?1?1 > base_passwords.txt

```encrypt.sh
while read LINE; do
  echo -n "$LINE" | openssl sha1 | awk '{print $NF}' >> passwords.txt
done <base_passwords.txt
```

hashcat -m 14600 -a 0 -w 3 part.enc passwords.txt -o luks_password.txt

password: 08cd7ef68a46dba71da0ce56c9488110a638387f
pre-sha1: crazyfence282

### TASK 5

Want to find: IP address

Encrypted files in /agent

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

start

```sh
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

config

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

to find IPs in bin
strings \[binName\] | grep -E '\[0-9\]{1,3}\.\[0-9\]{1,3}\.\[0-9\]{1,3}\.\[0-9\]{1,3}'

- agent: none
- config: 0.0.0.0
- diagclient:
  - A couple false positives
  - 127.0.0.1 (localhost)
- dropper
  - Same false positives as before
  - localhost again
  - Calls to this: 169.254.170.2 or 169.254.169.254
    - http://169.254.170.2/AWS_SECRET_ACCESS_KEYECDSAWithP256AndSHA256ECDSAWithP384AndSHA384ECDSAWithP521AndSHA512error
    - http://169.254.169.254/latest/api/tokenoutput
    - http://169.254.169.254/latest/meta-data/iam/security-credentials/tls

ssh hostname? nonroot_user

```
      FUN_00435700();
      if (*(char *)puVar1 == '\\0') {
        *puVar1 = 0x5f746f6f726e6f6e; -- translates to \_toornon
        *(undefined8 *)(param_2 + 0x85) = 0x726573755f746f;
      }
      FUN_00411f00("SSH_SERVER_ADDRESS",param_2 + 0xc0,1);
      FUN_00411f00("SSH_SERVER_PORT",param_2 + 0x100,1);
      FUN_00411f00("PRIVATE_KEY_PATH",*(undefined8 *)(param_1 + 0x28),1);
      FUN_00411f00("SSH_USERNAME",puVar1,1);
      FUN_00411f00("EXPECTED_HOST_KEY",param_2 + 0x140,1);
      FUN_00411f00("BALLOON_ID",param_1,1);
      FUN_004356a0(CONCAT17(uStack_11,uStack_18),&uStack_18);
```

0x00480f08: start of usernames and/or passwords?

getting mem: dd if=/proc/177/mem of=/tmp/agent_full skip=$((0x00400000)) bs=1 count=983040

dd if=/proc/253/mem of=/tmp/agent_stack skip=$((0x00770000)) bs=1 count=6266
88

answer: 100.105.36.5
0x64692405
got from running dropper with a fake config. Output:

```
1970/01/01 02:48:26 Connecting to server...
1970/01/01 02:48:26 ...Connected!
1970/01/01 02:48:26 Loading/tmp/upload
1970/01/01 02:50:26 Processing/tmp/upload/boot_log_1970-01-01T00:11:53+00:00boot_log_1970-01-01T00:11:53+00:00
1970/01/01 02:50:56 Failed to insert into MongoDB:server selection error: server selection timeout, current topology: { Type: Unknown, Servers: [{ Addr: 100.105.36.5:27017, Type: Unknown, Last error: dial tcp 100.105.36.5:27017: connect: network is unreachable }, ] }
```

binwalk -Me dropper:
database:
collection: files
database: snapshot-44180ec0a37b
url: mongodb://maintenance:36599080632635@100.105.36.5:27017/?authSource=snapshot-44180ec0a37b
server:
directory: /tmp/upload

### TASK 6

(Not finished, but here was some poking around I got to)
Connecting:
Make sure ssh config matches:

```bash
Host jumpbox external-support.bluehorizonmobile.com *
    User user
    HostName external-support.bluehorizonmobile.com
    IdentityFile /home/jjotto753/Documents/codebreaker_2023/task6/jumpbox.key
    IdentitiesOnly yes
    LocalForward 127.0.0.1:27017 100.105.36.5:27017
```

run `ssh jumpbox` in a different tab
run commands with this template

```python
import pymongo
import pprint
import bson

JUMPBOX_HOST = "external-support.bluehorizonmobile.com"
MONGO_HOST = "100.105.36.5"
MONGO_DB = "snapshot-44180ec0a37b"
MONGO_USERNAME = "maintenance"
MONGO_PASSWORD = "36599080632635"
MONGO_COLLECTION = "files"

client = pymongo.MongoClient("mongodb://"+MONGO_USERNAME+":"+MONGO_PASSWORD+"@127.0.0.1:27017/?authSource="+MONGO_DB+"&authMechanism=SCRAM-SHA-1")
db = client[MONGO_DB]
collection = db[MONGO_COLLECTION]

# Commands here

client.close()
```

Database(MongoClient(host=['127.0.0.1:27017'], document_class=dict, tz_aware=False, connect=True, authsource='snapshot-44180ec0a37b', authmechanism='SCRAM-SHA-1'), 'topology_description')

databases: {'name': 'snapshot-44180ec0a37b', 'sizeOnDisk': 8192, 'empty': False}

server info:

```json
{
  "version": "6.0.10",
  "gitVersion": "8e4b5670df9b9fe814e57cb5f3f8ee9407237b5a",
  "modules": [],
  "allocator": "tcmalloc",
  "javascriptEngine": "mozjs",
  "sysInfo": "deprecated",
  "versionArray": [6, 0, 10, 0],
  "openssl": {
    "running": "OpenSSL 3.0.2 15 Mar 2022",
    "compiled": "OpenSSL 3.0.2 15 Mar 2022"
  },
  "buildEnvironment": {
    "distmod": "ubuntu2204",
    "distarch": "x86_64",
    "cc": "/opt/mongodbtoolchain/v3/bin/gcc: gcc (GCC) 8.5.0",
    "ccflags": "-Werror -include mongo/platform/basic.h -ffp-contract=off -fasynchronous-unwind-tables -ggdb -Wall -Wsign-compare -Wno-unknown-pragmas -Winvalid-pch -fno-omit-frame-pointer -fno-strict-aliasing -O2 -march=sandybridge -mtune=generic -mprefer-vector-width=128 -Wno-unused-local-typedefs -Wno-unused-function -Wno-deprecated-declarations -Wno-unused-const-variable -Wno-unused-but-set-variable -Wno-missing-braces -fstack-protector-strong -fdebug-types-section -Wa,--nocompress-debug-sections -fno-builtin-memcmp",
    "cxx": "/opt/mongodbtoolchain/v3/bin/g++: g++ (GCC) 8.5.0",
    "cxxflags": "-Woverloaded-virtual -Wno-maybe-uninitialized -fsized-deallocation -std=c++17",
    "linkflags": "-Wl,--fatal-warnings -pthread -Wl,-z,now -fuse-ld=gold -fstack-protector-strong -fdebug-types-section -Wl,--no-threads -Wl,--build-id -Wl,--hash-style=gnu -Wl,-z,noexecstack -Wl,--warn-execstack -Wl,-z,relro -Wl,--compress-debug-sections=none -Wl,-z,origin -Wl,--enable-new-dtags",
    "target_arch": "x86_64",
    "target_os": "linux",
    "cppdefines": "SAFEINT_USE_INTRINSICS 0 PCRE_STATIC NDEBUG _XOPEN_SOURCE 700 _GNU_SOURCE _FORTIFY_SOURCE 2 BOOST_THREAD_VERSION 5 BOOST_THREAD_USES_DATETIME BOOST_SYSTEM_NO_DEPRECATED BOOST_MATH_NO_LONG_DOUBLE_MATH_FUNCTIONS BOOST_ENABLE_ASSERT_DEBUG_HANDLER BOOST_LOG_NO_SHORTHAND_NAMES BOOST_LOG_USE_NATIVE_SYSLOG BOOST_LOG_WITHOUT_THREAD_ATTR ABSL_FORCE_ALIGNED_ACCESS"
  },
  "bits": 64,
  "debug": "False",
  "maxBsonObjectSize": 16777216,
  "storageEngines": ["devnull", "ephemeralForTest", "wiredTiger"],
  "ok": 1.0
}
```

```json
{
  "users": [
    {
      "_id": "snapshot-44180ec0a37b.maintenance",
      "userId": "44fcc222-0676-472d-901f-45ef9f881192",
      "user": "maintenance",
      "db": "snapshot-44180ec0a37b",
      "roles": [
        { "role": "readWrite", "db": "snapshot-44180ec0a37b" },
        { "role": "userAdmin", "db": "snapshot-44180ec0a37b" }
      ],
      "mechanisms": ["SCRAM-SHA-1", "SCRAM-SHA-256"]
    }
  ],
  "ok": 1.0
}
```
