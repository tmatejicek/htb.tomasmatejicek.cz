---
layout: post
author: Tomáš Matějíček
title: "Laboratory"
date: 2020-12-10
tags: linux ssh exploit enumeration privesc hackthebox
---

## Úvod a kontext

Laboratory je stroj z Hack The Box. Níže ponechávám pouze technicky doložitelné kroky a místa, která nelze spolehlivě doložit, výslovně označuji k ověření.
## Počáteční průzkum

### Vyhledání otevřených portů

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.
```bash
ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP
```
```
PORT    STATE SERVICE  VERSION
```

### Detailní analýza služeb

V dalším kroku si zpřesňuji verze služeb a jejich charakteristiky, protože právě z těchto detailů obvykle vzniká rozhodnutí, zda pokračovat přes web, SSH nebo jinou vrstvu.
```text
22/tcp  open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
```
```
| ssh-hostkey:
|   3072 25:ba:64:8f:79:9d:5d:95:97:2c:1b:b2:5e:9b:55:0d (RSA)
|   256 28:00:89:05:55:f9:a2:ea:3c:7d:70:ea:4d:ea:60:0f (ECDSA)
|_  256 77:20:ff:e9:46:c0:68:92:1a:0b:21:29:d1:53:aa:87 (ED25519)
80/tcp  open  http     Apache httpd 2.4.41
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to https://laboratory.htb/
443/tcp open  ssl/http Apache httpd 2.4.41 ((Ubuntu))
| http-methods:
|_  Supported Methods: HEAD GET POST OPTIONS
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: The Laboratory
| ssl-cert: Subject: commonName=laboratory.htb
| Subject Alternative Name: DNS:git.laboratory.htb
| Issuer: commonName=laboratory.htb
| Public Key type: rsa
| Public Key bits: 4096
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2020-07-05T10:39:28
| Not valid after:  2024-03-03T10:39:28
| MD5:   2873 91a5 5022 f323 4b95 df98 b61a eb6c
|_SHA-1: 0875 3a7e eef6 8f50 0349 510d 9fbf abc3 c70a a1ca
| tls-alpn:
[... výstup zkrácen ...]
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-timesync:x:100:102:systemd Time Synchronization,,,:/run/systemd:/bin/false
systemd-network:x:101:103:systemd Network Management,,,:/run/systemd/netif:/bin/false
systemd-resolve:x:102:104:systemd Resolver,,,:/run/systemd/resolve:/bin/false
systemd-bus-proxy:x:103:105:systemd Bus Proxy,,,:/run/systemd:/bin/false
_apt:x:104:65534::/nonexistent:/bin/false
```

## Získání přístupu

### Získání user flagu

User flag zde slouží hlavně jako potvrzení, že už mám běžný uživatelský kontext a mohu pokračovat v lokální analýze systému.

[POZNÁMKA K OVĚŘENÍ: Konkrétní kroky pro získání uživatelského přístupu zde nejsou doložené s dostatečnou technickou přesností.]

## Eskalace oprávnění

### Získání root flagu

Tento krok ukazuje, jak se nalezená slabina nebo chyba v delegaci oprávnění mění v privilegovaný přístup.

[POZNÁMKA K OVĚŘENÍ: Konkrétní kroky pro eskalaci oprávnění a získání root přístupu zde nejsou doložené s dostatečnou technickou přesností.]

## Shrnutí klíčových poznatků

- Článek zachycuje jen část postupu, proto jsou místa bez dostatečné technické opory označena ověřovací poznámkou místo domněnek.
- Záměrně nedoplňuji neověřené detaily o exploitu, kredenciálech ani eskalaci; publikovatelná verze musí stát jen na dohledatelných krocích.
- Chybějící mezikroky mezi enumerací, potvrzením přístupu a finální eskalací zůstávají explicitně otevřené, protože je nelze doložit s dostatečnou technickou přesností.

## Co si odnést do praxe

- Pro publikovatelný HTB write-up je nutné uložit i mezikroky mezi enumerací, hypotézou a potvrzením přístupu; samotné placeholdery nestačí.
- Pokud chybí výstupy nebo přesná argumentace, je lepší explicitně přiznat nejistotu než doplňovat neověřené technické detaily.
- Stejné techniky mají smysl pouze v laboratorním nebo jinak autorizovaném testovacím prostředí.
