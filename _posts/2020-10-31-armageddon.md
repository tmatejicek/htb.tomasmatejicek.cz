---
layout: post
author: Tomáš Matějíček
title: "Armageddon"
date: 2020-10-31
tags: sql-injection ssh sudo php exploit enumeration
---

## Úvod a kontext

Armageddon je stroj z Hack The Box. Článek sleduje cestu od prvotní enumerace k ověřenému přístupu a průběžně vysvětluje, proč měl každý další krok technický smysl.
## Počáteční průzkum

### Vyhledání otevřených portů

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.
```bash
ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP
```
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey:
|   2048 82:c6:bb:c7:02:6a:93:bb:7c:cb:dd:9c:30:93:79:34 (RSA)
|   256 3a:ca:95:30:f3:12:d7:ca:45:05:bc:c7:f1:16:bb:fc (ECDSA)
|_  256 7a:d4:b3:68:79:cf:62:8a:7d:5a:61:e7:06:0f:5f:33 (ED25519)
80/tcp open  http    Apache httpd 2.4.6 ((CentOS) PHP/5.4.16)
|_http-favicon: Unknown favicon MD5: __CENSORED__
|_http-generator: Drupal 7 (http://drupal.org)
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
| http-robots.txt: 36 disallowed entries (15 shown)
| /includes/ /misc/ /modules/ /profiles/ /scripts/
| /themes/ /CHANGELOG.txt /cron.php /INSTALL.mysql.txt
| /INSTALL.pgsql.txt /INSTALL.sqlite.txt /install.php /INSTALL.txt
|_/LICENSE.txt /MAINTAINERS.txt
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.4.16
|_http-title: Welcome to  Armageddon |  Armageddon
```

## Analýza zjištění

### Identifikace a hledání exploitu

Zjišťuji technologii a ověřuji známé zranitelnosti.
```bash
searchsploit Drupal 7.56
```
```
---------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                              |  Path
---------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Drupal < 7.58 - 'Drupalgeddon3' (Authenticated) Remote Code (Metasploit)                                                    | php/webapps/44557.rb
Drupal < 7.58 - 'Drupalgeddon3' (Authenticated) Remote Code Execution (PoC)                                                 | php/webapps/44542.txt
Drupal < 7.58 / < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution                                         | php/webapps/44449.rb
Drupal < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution (Metasploit)                                     | php/remote/44482.rb
Drupal < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution (PoC)                                            | php/webapps/44448.py
Drupal < 8.5.11 / < 8.6.10 - RESTful Web Services unserialize() Remote Command Execution (Metasploit)                       | php/remote/46510.rb
Drupal < 8.6.10 / < 8.5.11 - REST Module Remote Code Execution                                                              | php/webapps/46452.txt
Drupal < 8.6.9 - REST Module Remote Code Execution                                                                          | php/webapps/46459.py
---------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

### Identifikace a hledání exploitu (2)

Zjišťuji technologii a ověřuji známé zranitelnosti.
```bash
searchsploit -p 44449
```
```
  Exploit: Drupal < 7.58 / < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution
      URL: https://www.exploit-db.com/exploits/44449
     Path: /usr/share/exploitdb/exploits/php/webapps/44449.rb
File Type: Ruby script, ASCII text, with CRLF line terminators
```

## Získání přístupu

### Přihlášení na cíl

Jakmile mám pověření nebo jednorázový shell, snažím se přejít na stabilní a reprodukovatelný přístup, aby bylo možné bezpečně pokračovat v interní enumeraci.
```bash
ssh brucetherealadmin@$IP
```
```
## cat user.txt
__CENSORED__

## sudo -l
=>     (root) NOPASSWD: __CENSORED__ install *
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

## Získání přístupu

### Získání user flagu

User flag zde slouží hlavně jako potvrzení, že už mám běžný uživatelský kontext a mohu pokračovat v lokální analýze systému.

Následující úsek zachycuje přechod k uživatelskému přístupu a jeho ověření přes `user.txt`.

```text
ssh brucetherealadmin@$IP
## cat user.txt
f13a151e923b81d9f5e318b555d09ce5

## sudo -l
=>     (root) NOPASSWD: /usr/bin/snap install *
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
