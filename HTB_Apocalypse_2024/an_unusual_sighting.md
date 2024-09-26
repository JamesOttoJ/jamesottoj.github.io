---
layout: default
title: An Unusual Sighting
parent: HTB Apocalypse 2024
nav_order: 2
---

To start, this challenge provides two files, bash_history.log and sshd.log. The bash history file should give a list of command run on the system where a couple are suspicious. The next one is a log of ssh attemts that will show who connected to the system when. Based on the oporating hours in the description, it would be good to look for times from 00:00-09:00 and 19:00-23:59. Starting with bash history, two of the most common commands run by novice attackers are `whoami` and `cat /etc/passwd`. If we search for those commands in the file using `grep "whoami" ./bash_history.txt` and `grep "cat /etc/passwd ./bash_history.txt` we see that both get a hit at \[2024-02-19 04:00:18\] and \[2024-02-19 04:00:40\] respecively. Of note, both of these are outside operating hours. While this can sometimes be because the system didn't have its clock set properly, looking at other commands in the file, this is unlikely.

Moving to the sshd.log file, we can check for logins on that day using `grep "2024-02-19" ./sshd.log`. This provides the following lines:
```
[2024-02-19 04:00:14] Connection from 2.67.182.119 port 60071 on 100.107.36.130 port 2221 rdomain ""
[2024-02-19 04:00:14] Failed publickey for root from 2.67.182.119 port 60071 ssh2: ECDSA SHA256:OPkBSs6okUKraq8pYo4XwwBg55QSo210F09FCe1-yj4
[2024-02-19 04:00:14] Accepted password for root from 2.67.182.119 port 60071 ssh2
[2024-02-19 04:00:14] Starting session: shell on pts/2 for root from 2.67.182.119 port 60071 id 0
[2024-02-19 04:38:17] syslogin_perform_logout: logout() returned an error
[2024-02-19 04:38:17] Received disconnect from 2.67.182.119 port 60071:11: disconnected by user
[2024-02-19 04:38:17] Disconnected from user root 2.67.182.119 port 60071
```

Looking at what happens, someone using the IP `2.67.182.119` connects to the IP `100.107.36.130` and port `2221` using the fingerprint `OPkBSs6okUKraq8pYo4XwwBg55QSo210F09FCe1-yj4`. This connection is done with the root user and its password

With that information. Connecting the the server and answering its questions will give the flag. After connecting to the server with `nc IP:PORT`, the following questions pop up where the answers found above can be inserted. The only answer not found yet is the first successful connection, but that can simply be found with `grep "Starting session" ./sshd.log` and looking at the timestamp for the first line. Of note, the command (added at the bottom) also shows all the IPs that connected to the server where all but one have an initial octet in the 100s. This can also indicate an abnormal connection for internal servers because it means that attacker is likely not on a known network.

```
└──╼ $nc 83.136.254.11 51119

+---------------------+---------------------------------------------------------------------------------------------------------------------+
|        Title        |                                                     Description                                                     |
+---------------------+---------------------------------------------------------------------------------------------------------------------+
| An unusual sighting |                        As the preparations come to an end, and The Fray draws near each day,                        |
|                     |             our newly established team has started work on refactoring the new CMS application for the competition. |
|                     |                  However, after some time we noticed that a lot of our work mysteriously has been disappearing!     |
|                     |                     We managed to extract the SSH Logs and the Bash History from our dev server in question.        |
|                     |               The faction that manages to uncover the perpetrator will have a massive bonus come the competition!   |
|                     |                                                                                                                     |
|                     |                                            Note: Operating Hours of Korp: 0900 - 1900                               |
+---------------------+---------------------------------------------------------------------------------------------------------------------+


Note 2: All timestamps are in the format they appear in the logs

What is the IP Address and Port of the SSH Server (IP:PORT)
> 100.107.36.130:2221
[+] Correct!

What time is the first successful Login
> 2024-02-13 11:29:50
[+] Correct!

What is the time of the unusual Login
> 2024-02-19 04:00:14
[+] Correct!

What is the Fingerprint of the attacker's public key
> OPkBSs6okUKraq8pYo4XwwBg55QSo210F09FCe1-yj4
[+] Correct!

What is the first command the attacker executed after logging in
> whoami
[+] Correct!

What is the final command the attacker executed before logging out
> ./setup
[+] Correct!

[+] Here is the flag: HTB{4n_unusual_s1ght1ng_1n_SSH_l0gs!}
```

grep "Starting session" ./sshd.log
```
[2024-02-13 11:29:50] Starting session: shell on pts/2 for root from 100.81.51.199 port 63172 id 0
[2024-02-15 10:40:51] Starting session: shell on pts/2 for softdev from 101.111.18.92 port 44711 id 0
[2024-02-15 18:51:51] Starting session: shell on pts/2 for softdev from 101.111.18.92 port 44711 id 0
[2024-02-16 10:26:51] Starting session: shell on pts/2 for softdev from 100.86.71.224 port 58713 id 0
[2024-02-19 04:00:14] Starting session: shell on pts/2 for root from 2.67.182.119 port 60071 id 0
[2024-02-20 11:10:14] Starting session: shell on pts/2 for softdev from 100.87.190.253 port 63371 id 0
[2024-02-21 10:49:50] Starting session: shell on pts/2 for softdev from 102.11.76.9 port 48875 id 0
[2024-02-21 18:17:50] Starting session: shell on pts/2 for softdev from 100.7.98.129 port 47765 id 0
[2024-02-22 12:07:14] Starting session: shell on pts/2 for softdev from 100.11.239.78 port 49811 id 0
[2024-02-23 10:49:50] Starting session: shell on pts/2 for softdev from 102.11.76.9 port 48875 id 0
[2024-02-23 18:17:50] Starting session: shell on pts/2 for softdev from 100.7.98.129 port 47765 id 0
[2024-02-24 11:15:08] Starting session: shell on pts/2 for softdev from 102.11.76.9 port 48875 id 0
[2024-02-24 14:07:18] Starting session: shell on pts/2 for softdev from 100.7.98.129 port 47765 id 0
[2024-02-26 09:57:01] Starting session: shell on pts/2 for softdev from 102.11.76.9 port 48875 id 0
[2024-02-26 15:07:18] Starting session: shell on pts/2 for softdev from 100.7.98.129 port 47765 id 0
[2024-02-27 13:41:51] Starting session: shell on pts/2 for softdev from 100.85.206.20 port 54976 id 0
[2024-02-28 17:19:50] Starting session: shell on pts/2 for softdev from 100.7.98.129 port 47765 id 0
[2024-02-29 09:57:01] Starting session: shell on pts/2 for softdev from 102.11.76.9 port 48875 id 0
[2024-02-29 18:01:29] Starting session: shell on pts/2 for softdev from 100.7.98.129 port 47765 id 0
```