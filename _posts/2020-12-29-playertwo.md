---
layout: post
author: Tomáš Matějíček
title: "PlayerTwo"
date: 2020-12-29
tags: linux rce ssh php exploit enumeration
---
[PlayerTwo](https://www.hackthebox.eu/home/machines/profile/221) je stroj z Hack The Box. Klíčová část útoku je webová enumerace a praktické zneužití nalezené slabiny.

### Vyhledání otevřených portů
Nejdřív mapuji služby, které jsou dostupné zvenku.
`ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP`
```
PORT     STATE SERVICE VERSION
```

### Přihlášení na cíl
Po získání přihlašovacích údajů přecházím na stabilní shell na cílovém stroji.
`22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)`
```
| ssh-hostkey:
|   2048 0e:7b:11:2c:5e:61:04:6b:e8:1c:bb:47:b8:4d:fe:5a (RSA)
|   256 18:a0:87:56:64:06:17:56:4d:6a:8c:79:4b:61:56:90 (ECDSA)
|_  256 b6:4b:fc:e9:62:08:5a:60:e0:43:69:af:29:b3:27:14 (ED25519)
80/tcp   open  http    Apache httpd 2.4.29 ((Ubuntu))
| http-methods:
|_  Supported Methods: POST OPTIONS HEAD GET
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
8545/tcp open  http    twirp
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

=> twirp_invalid_route
=> https://github.com/twitchtv/twirp
=> https://developers.google.com/protocol-buffers/docs/proto3

=> http://player2.htb/proto/generated.proto
```

### Enumerace webu
Procházím web a hledám skryté cesty, které nejsou dostupné z hlavní stránky.
`curl --location "http://product.player2.htb/api/totp.php" \`
```
	 --header "Content-Type:application/json" \
     --cookie "PHPSESSID=van1nvsd6ugc4jfkgn12qb9ehn" \
     --data '{"action": 0}' \
     --verbose

=> http://product.player2.htb/protobs/

? Jak dostat exploit do firmware ?

wfuzz --hh 26 -w /usr/share/wordlists/dirb/big.txt -X "POST" -b 'PHPSESSID=dv0got03i77dqdbf4jpk8qd3tb' -d '{action=FUZZ}' http://product.player2.htb/api/totp

./dirsearch/dirsearch.py -u http://player2.htb/vendor/ -e php -x 403 -w /usr/share/wordlists/wfuzz/general/megabeast.txt

view-source:http://product.player2.htb/totp

./dirsearch/dirsearch.py -u http://product.player2.htb/ -e php -x 403

./dirsearch/dirsearch.py -u http://player2.htb/proto -e proto -x 403
```

### Enumerace webu (2)
Procházím web a hledám skryté cesty, které nejsou dostupné z hlavní stránky.
`dirb http://product.player2.htb/`

### Získání user flagu
`TODO`

### Získání root flagu
`TODO`
