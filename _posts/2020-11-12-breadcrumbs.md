---
layout: post
author: Tomáš Matějíček
title: "Breadcrumbs"
date: 2020-11-12
tags: windows sql-injection smb ssh php
---

## Úvod a kontext

Breadcrumbs je stroj z Hack The Box. Článek sleduje cestu od prvotní enumerace k ověřenému přístupu a průběžně vysvětluje, proč měl každý další krok technický smysl.

## Počáteční průzkum

### Vyhledání otevřených portů

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.
```bash
ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP
```
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

## Analýza zjištění

### Lámání hesel nebo hashů

Hash nebo zašifrovaný artefakt má smysl lámat jen tehdy, pokud může otevřít další službu, účet nebo vrstvu prostředí; právě to zde ověřuji.
```text
john	__CENSORED__
```
```
emma	__CENSORED__
william	__CENSORED__
lucas	__CENSORED__
sirine	__CENSORED__
juliette	__CENSORED__
support	__CENSORED__
```

### Lámání hesel nebo hashů (2)

Hash nebo zašifrovaný artefakt má smysl lámat jen tehdy, pokud může otevřít další službu, účet nebo vrstvu prostředí; právě to zde ověřuji.
```bash
hashcat -m100 -a 0 hashes /usr/share/wordlists/rockyou.txt
```

## Získání přístupu

### Přihlášení na cíl

Jakmile mám pověření nebo jednorázový shell, snažím se přejít na stabilní a reprodukovatelný přístup, aby bylo možné bezpečně pokračovat v interní enumeraci.
```bash
ssh www-data@$IP
```
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

Jakmile mám pověření nebo jednorázový shell, snažím se přejít na stabilní a reprodukovatelný přístup, aby bylo možné bezpečně pokračovat v interní enumeraci.
```bash
ssh juliette@$IP
```
```
gc C:\Users\juliette\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\plum.sqlite* .
```

### Získání user flagu

User flag zde slouží hlavně jako potvrzení, že už mám běžný uživatelský kontext a mohu pokračovat v lokální analýze systému.

Následující úsek zachycuje přechod k prvnímu stabilnímu uživatelskému přístupu nebo shellu.

```text
ssh development@$IP
fN3)sN5Ee@g
http://passmanager.htb:1234/index.phpmethod=select&username=administrator&table=passwords
```

## Eskalace oprávnění

### Získání root flagu

Tento krok ukazuje, jak se nalezená slabina nebo chyba v delegaci oprávnění mění v privilegovaný přístup.
```bash
more root.txt
```
```
__CENSORED__
```

## Shrnutí klíčových poznatků

- Rozhodující byla síťová a doménová enumerace, protože právě z dostupných služeb a sdílení vzešly další identity nebo tajné údaje.
- K uživatelskému přístupu vedla práce s nalezenými přihlašovacími údaji, klíči nebo hashi a jejich ověření proti reálně dostupné službě.
- Finální část ukazuje, že po získání shellu je nutné systematicky hledat slabé delegace oprávnění, uložená tajemství a automatizované procesy.

## Co si odnést do praxe

- Ve webové vrstvě je důležité omezit úniky citlivých souborů, testovacích endpointů a vývojových artefaktů, protože často slouží jako odrazový můstek k dalším službám.
- I zdánlivě dílčí úniky konfigurace, lokálních tajemství nebo interních rozhraní je potřeba brát vážně, protože právě jejich řetězení často rozhodne o kompromitaci hostu.
- Přístupové údaje je potřeba oddělovat mezi službami a minimalizovat jejich opětovné použití, jinak se z jedné slabiny rychle stane plnohodnotný vstup do systému.
- Inventura verzí a včasné záplatování snižují prostor pro přímé zneužití známých chyb i pro slepé spoléhání na zastaralé komponenty.
