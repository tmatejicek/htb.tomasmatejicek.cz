---
layout: post
author: Tomáš Matějíček
title: "Fortress-Jet"
date: 2020-11-28
tags: linux sql-injection ssh php exploit enumeration
---

## Úvod a kontext

Fortress-Jet je stroj z Hack The Box. Článek sleduje cestu od prvotní enumerace k ověřenému přístupu a průběžně vysvětluje, proč měl každý další krok technický smysl.
## Počáteční průzkum

### Vyhledání otevřených portů

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.
```bash
nmap -p 1-65535 -T4 -A -sC -v $IP
```
```
PORT     STATE SERVICE  VERSION
```

### Detailní analýza služeb

V dalším kroku si zpřesňuji verze služeb a jejich charakteristiky, protože právě z těchto detailů obvykle vzniká rozhodnutí, zda pokračovat přes web, SSH nebo jinou vrstvu.
```text
22/tcp   open  ssh      OpenSSH 7.2p2 Ubuntu 4ubuntu2.4 (Ubuntu Linux; protocol 2.0)
```
```
| ssh-hostkey:
|   2048 62:f6:49:80:81:cf:f0:07:0e:5a:ad:e9:8e:1f:2b:7c (RSA)
|   256 54:e2:7e:5a:1c:aa:9a:ab:65:ca:fa:39:28:bc:0a:43 (ECDSA)
|_  256 93:bc:37:b7:e0:08:ce:2d:03:99:01:0a:a9:df:da:cd (ED25519)
53/tcp   open  domain   ISC BIND 9.10.3-P4 (Ubuntu Linux)
| dns-nsid:
|_  bind.version: 9.10.3-P4-Ubuntu
80/tcp   open  http     nginx 1.10.3 (Ubuntu)
| http-methods:
|_  Supported Methods: GET HEAD
|_http-server-header: nginx/1.10.3 (Ubuntu)
|_http-title: Welcome to nginx on Debian!
5555/tcp open  freeciv?
| fingerprint-strings:
|   DNSVersionBindReqTCP, GenericLines, GetRequest, adbConnect:
|     enter your name:
|     [31mMember manager!
|     edit
|     change name
|     gift
|     exit
7777/tcp open  cbt?
| fingerprint-strings:
|   Arucer, DNSStatusRequestTCP, DNSVersionBindReqTCP, GenericLines, GetRequest, HTTPOptions, RPCCheck, RTSPRequest, Socks5, X11Probe:
|     --==[[ Spiritual Memo ]]==--
|     Create a memo
|     Show memo
|     Delete memo
|     Can't you read mate?
9201/tcp open  http     BaseHTTPServer 0.3 (Python 2.7.12)
| http-methods:
|_  Supported Methods: GET
|_http-title: Site doesn't have a title (application/json).

nginx[1.10.3], HTTPServer[Ubuntu Linux][nginx/1.10.3 (Ubuntu)], HTML5
```

## Získání přístupu

### Získání user flagu

User flag zde slouží hlavně jako potvrzení, že už mám běžný uživatelský kontext a mohu pokračovat v lokální analýze systému.

V pracovních poznámkách se dochoval úsek, který zachycuje přechod k uživatelskému přístupu a jeho ověření přes `user.txt`.

```text
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
cat a_flag_is_here.txt
JET{pr3g_r3pl4c3_g3ts_y0u_pwn3d}
```

## Eskalace oprávnění

### Získání root flagu

Tento krok ukazuje, jak se nalezená slabina nebo chyba v delegaci oprávnění mění v privilegovaný přístup.

V poznámkách je zachycen i postup, kterým se potvrzuje privilegovaný přístup a načtení `root.txt`.

```text
cd /home/alex
cat flag.txt
JET{0v3rfL0w_f0r_73h_lulz}
```

## Analýza zjištění

### Přílohy

![fade.gif](/drafts/Fortress-Jet/fade.gif)

## Shrnutí klíčových poznatků

- Úvodní směr určovala webová enumerace: důležité nebylo jen něco najít, ale správně vyhodnotit, který artefakt skutečně otevírá další krok.
- K uživatelskému přístupu vedla práce s nalezenými přihlašovacími údaji, klíči nebo hashi a jejich ověření proti reálně dostupné službě.
- Finální část ukazuje, že po získání shellu je nutné systematicky hledat slabé delegace oprávnění, uložená tajemství a automatizované procesy.

## Co si odnést do praxe

- Ve webové vrstvě je důležité omezit úniky citlivých souborů, testovacích endpointů a vývojových artefaktů, protože často slouží jako odrazový můstek k dalším službám.
- Přístupové údaje je potřeba oddělovat mezi službami a minimalizovat jejich opětovné použití, jinak se z jedné slabiny rychle stane plnohodnotný vstup do systému.
- Inventura verzí a včasné záplatování snižují prostor pro přímé zneužití známých chyb i pro slepé spoléhání na zastaralé komponenty.
- Stejné techniky mají smysl pouze v laboratorním nebo jinak autorizovaném testovacím prostředí.
