---
layout: post
author: Tomáš Matějíček
title: "PlayerTwo"
date: 2020-12-29
tags: linux rce ssh php exploit enumeration
---

## Úvod a kontext

PlayerTwo je stroj z Hack The Box. Níže ponechávám pouze technicky doložitelné kroky a místa, která nelze spolehlivě doložit, výslovně označuji k ověření.
## Počáteční průzkum

### Vyhledání otevřených portů

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.
```bash
ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP
```
```
PORT     STATE SERVICE VERSION
```

### Detailní analýza služeb

V dalším kroku si zpřesňuji verze služeb a jejich charakteristiky, protože právě z těchto detailů obvykle vzniká rozhodnutí, zda pokračovat přes web, SSH nebo jinou vrstvu.
```text
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
```
```
| ssh-hostkey:
|   2048 0e:7b:11:2c:5e:61:04:6b:e8:1c:bb:47:b8:4d:fe:5a (RSA)
|   256 18:a0:87:56:64:06:17:56:4d:6a:8c:79:4b:61:56:90 (ECDSA)
|_  256 b6:4b:fc:e9:62:08:5a:60:e0:43:69:af:29:b3:27:14 (ED25519)
80/tcp   open  http    Apache httpd 2.4.29 ((Ubuntu))
| http-methods:
|_  Supported Methods: POST OPTIONS HEAD GET
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
8545/tcp open  http    twirp
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

=> twirp_invalid_route
=> https://github.com/twitchtv/twirp
=> https://developers.google.com/protocol-buffers/docs/proto3

=> http://player2.htb/proto/generated.proto
```

### Enumerace webu

Ve webové vrstvě hledám neveřejné cesty, vývojové artefakty a chybně vystavené soubory, protože právě ty často prozradí technologii aplikace, interní workflow nebo přímo přístupové údaje.
```bash
curl --location "http://product.player2.htb/api/totp.php" \
```
```
	 --header "Content-Type:application/json" \
     --cookie "PHPSESSID=van1nvsd6ugc4jfkgn12qb9ehn" \
     --data '{"action": 0}' \
     --verbose

=> http://product.player2.htb/protobs/

? Jak dostat exploit do firmware ?

wfuzz --hh 26 -w /usr/share/wordlists/dirb/big.txt -X "POST" -b 'PHPSESSID=dv0got03i77dqdbf4jpk8qd3tb' -d '{action=FUZZ}' http://product.player2.htb/api/totp

./dirsearch/dirsearch.py -u http://player2.htb/vendor/ -e php -x 403 -w /usr/share/wordlists/wfuzz/general/megabeast.txt

view-source:http://product.player2.htb/totp

./dirsearch/dirsearch.py -u http://product.player2.htb/ -e php -x 403

./dirsearch/dirsearch.py -u http://player2.htb/proto -e proto -x 403
```

### Enumerace webu (2)

Ve webové vrstvě hledám neveřejné cesty, vývojové artefakty a chybně vystavené soubory, protože právě ty často prozradí technologii aplikace, interní workflow nebo přímo přístupové údaje.
```bash
dirb http://product.player2.htb/
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
