---
layout: post
author: Tomáš Matějíček
title: "Intelligence"
date: 2020-12-05
tags: windows smb kerberos ldap active-directory
---

## Úvod a kontext

Intelligence je stroj z Hack The Box. Článek sleduje cestu od prvotní enumerace k ověřenému přístupu a průběžně vysvětluje, proč měl každý další krok technický smysl.

## Počáteční průzkum

### Vyhledání otevřených portů

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.
```bash
ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP
```
```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
|_http-favicon: Unknown favicon MD5: __CENSORED__
| http-methods:
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Intelligence
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2021-10-29 22:49:14Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: intelligence.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=dc.intelligence.htb
| Subject Alternative Name: othername:<unsupported>, DNS:dc.intelligence.htb
| Issuer: commonName=intelligence-DC-CA
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2021-04-19T00:43:16
| Not valid after:  2022-04-19T00:43:16
| MD5:   7767 9533 67fb d65d 6065 dff7 7ad8 3e88
|_SHA-1: 1555 29d9 fef8 1aec 41b7 dab2 84d7 0f9d 30c7 bde7
|_ssl-date: 2021-10-29T22:50:46+00:00; +7h03m36s from scanner time.
445/tcp   open  microsoft-ds?
[... výstup zkrácen ...]
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 7h03m35s, deviation: 0s, median: 7h03m35s
| smb2-security-mode:
|   2.02:
|_    Message signing enabled and required
| smb2-time:
|   date: 2021-10-29T22:50:05
|_  start_date: N/A
```

### Enumerace SMB

U SMB sdílení ověřuji, jaká data jsou dostupná bez dalších oprávnění a zda z nich lze získat účty, dokumenty nebo konfigurační tajemství.
```bash
smbmap -H intelligence.htb -u Tiffany.Molina -p NewIntelligenceCorpUser9876 -R --depth 1
```
```
[+] IP: intelligence.htb:445    Name: unknown
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        IPC$                                                    READ ONLY       Remote IPC
        .\IPC$\*
        fr--r--r--                3 Mon Jan  1 00:57:44 1601    InitShutdown
        fr--r--r--                5 Mon Jan  1 00:57:44 1601    lsass
        fr--r--r--                3 Mon Jan  1 00:57:44 1601    ntsvcs
        fr--r--r--                3 Mon Jan  1 00:57:44 1601    scerpc
        fr--r--r--                1 Mon Jan  1 00:57:44 1601    Winsock2\CatalogChangeListener-390-0
        fr--r--r--                3 Mon Jan  1 00:57:44 1601    epmapper
        fr--r--r--                1 Mon Jan  1 00:57:44 1601    Winsock2\CatalogChangeListener-1c4-0
        fr--r--r--                3 Mon Jan  1 00:57:44 1601    LSM_API_service
        fr--r--r--                3 Mon Jan  1 00:57:44 1601    eventlog
        fr--r--r--                1 Mon Jan  1 00:57:44 1601    Winsock2\CatalogChangeListener-280-0
        fr--r--r--                3 Mon Jan  1 00:57:44 1601    atsvc
        fr--r--r--                4 Mon Jan  1 00:57:44 1601    wkssvc
        fr--r--r--                1 Mon Jan  1 00:57:44 1601    Winsock2\CatalogChangeListener-4ec-0
        fr--r--r--                1 Mon Jan  1 00:57:44 1601    Winsock2\CatalogChangeListener-24c-0
        fr--r--r--                1 Mon Jan  1 00:57:44 1601    Winsock2\CatalogChangeListener-24c-1
        fr--r--r--                3 Mon Jan  1 00:57:44 1601    RpcProxy\49691
        fr--r--r--                3 Mon Jan  1 00:57:44 1601    6f2901363fb76a9e
        fr--r--r--                3 Mon Jan  1 00:57:44 1601    RpcProxy\593
[... výstup zkrácen ...]
        fr--r--r--           524288 Mon Apr 19 02:51:46 2021    NTUSER.DAT{6392777f-a0b5-11eb-ae6e-000c2908ad93}.TMContainer00000000000000000001.regtrans-ms
        fr--r--r--           524288 Mon Apr 19 02:51:46 2021    NTUSER.DAT{6392777f-a0b5-11eb-ae6e-000c2908ad93}.TMContainer00000000000000000002.regtrans-ms
        fr--r--r--               20 Mon Apr 19 02:51:46 2021    ntuser.ini
        dw--w--w--                0 Mon Apr 19 02:51:46 2021    Pictures
        dr--r--r--                0 Mon Apr 19 02:51:46 2021    Recent
        dr--r--r--                0 Mon Apr 19 02:51:46 2021    Saved Games
        dr--r--r--                0 Mon Apr 19 02:51:46 2021    SendTo
        dr--r--r--                0 Mon Apr 19 02:51:46 2021    Start Menu
        dr--r--r--                0 Mon Apr 19 02:51:46 2021    Templates
        dw--w--w--                0 Mon Apr 19 02:51:46 2021    Videos
```

### Enumerace SMB (2)

U SMB sdílení ověřuji, jaká data jsou dostupná bez dalších oprávnění a zda z nich lze získat účty, dokumenty nebo konfigurační tajemství.
```bash
impacket-smbclient Tiffany.Molina:NewIntelligenceCorpUser9876@intelligence.htb
```

## Analýza zjištění

### Lámání hesel nebo hashů

Hash nebo zašifrovaný artefakt má smysl lámat jen tehdy, pokud může otevřít další službu, účet nebo vrstvu prostředí; právě to zde ověřuji.
```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```
```
=> Mr.Teddy         (Ted.Graves)
```

## Získání přístupu

### Získání user flagu

User flag zde slouží hlavně jako potvrzení, že už mám běžný uživatelský kontext a mohu pokračovat v lokální analýze systému.
```bash
cat user.txt
```
```
__CENSORED__
```

## Eskalace oprávnění

### Získání root flagu

Tento krok ukazuje, jak se nalezená slabina nebo chyba v delegaci oprávnění mění v privilegovaný přístup.
```bash
cat root.txt
```
```
__CENSORED__
```

## Shrnutí klíčových poznatků

- Rozhodující byla síťová a doménová enumerace, protože právě z dostupných služeb a sdílení vzešly další identity nebo tajné údaje.
- K uživatelskému přístupu vedla práce s nalezenými přihlašovacími údaji, klíči nebo hashi a jejich ověření proti reálně dostupné službě.
- Eskalace oprávnění stála na příliš širokém `sudo` pravidle nebo na možnosti ovlivnit vstup či prostředí privilegovaného procesu.

## Co si odnést do praxe

- V prostředí Active Directory je klíčové hlídat oprávnění ke sdílením, servisním účtům a delegacím; i malý únik informací se snadno řetězí do dalších kroků.
- Pravidla `sudo` mají být co nejmenší a bez zbytečných možností typu `SETENV`, volného zápisu nebo vyhodnocování neověřeného vstupu.
- Přístupové údaje je potřeba oddělovat mezi službami a minimalizovat jejich opětovné použití, jinak se z jedné slabiny rychle stane plnohodnotný vstup do systému.
- Stejné techniky mají smysl pouze v laboratorním nebo jinak autorizovaném testovacím prostředí.
