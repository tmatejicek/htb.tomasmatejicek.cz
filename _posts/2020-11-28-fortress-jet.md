---
layout: post
author: Tomáš Matějíček
title: "Fortress-Jet"
date: 2020-11-28
tags: linux sql-injection ssh php exploit enumeration
---
Fortress-Jet je stroj z Hack The Box. Klíčová část útoku je webová enumerace a praktické zneužití nalezené slabiny.

### Vyhledání otevřených portů
Nejdřív mapuji služby, které jsou dostupné zvenku.
`nmap -p 1-65535 -T4 -A -sC -v $IP`
```
PORT     STATE SERVICE  VERSION
```

### Přihlášení na cíl
Po získání přihlašovacích údajů přecházím na stabilní shell na cílovém stroji.
`22/tcp   open  ssh      OpenSSH 7.2p2 Ubuntu 4ubuntu2.4 (Ubuntu Linux; protocol 2.0)`
```
| ssh-hostkey:
|   2048 62:f6:49:80:81:cf:f0:07:0e:5a:ad:e9:8e:1f:2b:7c (RSA)
|   256 54:e2:7e:5a:1c:aa:9a:ab:65:ca:fa:39:28:bc:0a:43 (ECDSA)
|_  256 93:bc:37:b7:e0:08:ce:2d:03:99:01:0a:a9:df:da:cd (ED25519)
53/tcp   open  domain   ISC BIND 9.10.3-P4 (Ubuntu Linux)
| dns-nsid:
|_  bind.version: 9.10.3-P4-Ubuntu
80/tcp   open  http     nginx 1.10.3 (Ubuntu)
| http-methods:
|_  Supported Methods: GET HEAD
|_http-server-header: nginx/1.10.3 (Ubuntu)
|_http-title: Welcome to nginx on Debian!
5555/tcp open  freeciv?
| fingerprint-strings:
|   DNSVersionBindReqTCP, GenericLines, GetRequest, adbConnect:
|     enter your name:
|     [31mMember manager!
|     edit
|     change name
|     gift
|     exit
7777/tcp open  cbt?
| fingerprint-strings:
|   Arucer, DNSStatusRequestTCP, DNSVersionBindReqTCP, GenericLines, GetRequest, HTTPOptions, RPCCheck, RTSPRequest, Socks5, X11Probe:
|     --==[[ Spiritual Memo ]]==--
|     Create a memo
|     Show memo
|     Delete memo
|     Can't you read mate?
9201/tcp open  http     BaseHTTPServer 0.3 (Python 2.7.12)
| http-methods:
|_  Supported Methods: GET
|_http-title: Site doesn't have a title (application/json).

nginx[1.10.3], HTTPServer[Ubuntu Linux][nginx/1.10.3 (Ubuntu)], HTML5
```

### Získání user flagu
`TODO`

### Získání root flagu
`TODO`

### Přílohy
![fade.gif](/drafts/Fortress-Jet/fade.gif)
