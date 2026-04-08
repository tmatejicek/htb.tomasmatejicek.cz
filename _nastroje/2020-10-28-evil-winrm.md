---
layout: post
author: Tomáš Matějíček
title: "Evil-WinRM"
date: 2020-10-28
tags: nastroje windows active-directory winrm powershell
---
## Úvod a kontext

`Evil-WinRM` není exploit. Je to praktický klient pro okamžik, kdy už existuje použitelný windowsový účet, hash nebo jiný autentizační materiál a je potřeba z něj udělat stabilní vzdálený shell. Právě proto se v rozborech opakuje tak často: jakmile řetězec skončí u WinRM služby na `5985` nebo `5986`, bývá to nejpohodlnější způsob další práce.

Na [Resolute](/resolute) otevře shell po nalezení výchozího hesla, na [Forestu](/forest) a [Sauně](/sauna) po Kerberos nebo AS-REP footholdu, na [Remote](/remote) po reuse tajemství z TeamVieweru a na [Anubis](/anubis) až po certifikátové kompromitaci identity. To je důležité: `Evil-WinRM` sám nic neláme. Jen převádí už získaný přístup do použitelného PowerShell kontextu.

## Co tenhle nástroj v praxi řeší

Když je na Windows hostu otevřený WinRM a existují validní přihlašovací údaje, bývá několik možností:

- pracovat dál přes SMB a jednorázové příkazy,
- zůstat u webshellu nebo jiného křehkého footholdu,
- nebo přejít na interaktivní vzdálený shell.

Právě třetí varianta bývá nejpraktičtější. `Evil-WinRM` dává:

- stabilní PowerShell session,
- pohodlné spouštění dalších příkazů a skriptů,
- upload a download souborů,
- a mnohem čistší pracovní režim než náhodná kombinace `psexec`, webshellu a jednorázových RPC příkazů.

To je důvod, proč se na blogu po získání účtu často přechází právě na WinRM. Ne proto, že by to bylo "cool", ale protože to zjednoduší všechno další.

## Nejčastější scénáře využití v tomto projektu

### Přihlášení jménem a heslem po prvním identity footholdu

Nejběžnější scénář je jednoduchý: nejprve někde unikne heslo nebo padne na password spray a pak se ověří proti WinRM.

To je dobře vidět třeba na:

- [Resolute](/resolute), kde `Welcome123!` otevře účet `melanie`,
- [Heist](/heist), kde se po Cisco configu a sprayingu přechází na `Chase`,
- [Driver](/driver), kde cracknutý NTLMv2 hash vede k heslu `liltony`,
- [Remote](/remote), kde reuse tajemství otevře `Administrator`.

Prakticky jde o nejčistší variantu:

```bash
evil-winrm -i Resolute.megabank.local -u melanie -p 'Welcome123!'
```

Jakmile je shell otevřený, lze pohodlně pokračovat ve čtení transcriptů, dumpu tajemství, enumeraci služeb nebo dalším laterálním pohybu.

### Stabilní shell po Kerberos nebo AD enumeraci

Na AD strojích je běžné, že heslo nevznikne z webu, ale z práce s identitou: AS-REP roast, veřejná jména, dokumentová metadata nebo defaultní heslo. Jakmile takový účet vyjde a WinRM je dostupný, `Evil-WinRM` bývá první rozumná volba.

Na [Forestu](/forest) se po cracknutí `svc-alfresco:s3rvice` okamžitě přechází na WinRM, protože je potřeba stabilní PowerShell pro sběr dat do BloodHoundu. Na [Sauně](/sauna) totéž platí pro `FSmith` i pozdější pass-the-hash jako `Administrator`. Praktický kontext téhle části rozebírám i v článku [Kerberos útoky z nulového přístupu](/techniky/kerberos-utoky-z-nuloveho-pristupu-as-rep-roast-a-prace-s-kandidatnimi-uzivateli).

```bash
evil-winrm -i 10.10.10.175 -u FSmith -p 'Thestrokes23'
evil-winrm -i 10.10.10.175 -u Administrator -H 'd9485863c1e9e05851aa40cbb4ab9dff'
```

Tady je důležité, že WinRM není cíl. Je to kanál, který zpřehlední další práci po získání identity.

### Kerberos a certifikátové scénáře

Na [Anubis](/anubis) je dobře vidět, že `Evil-WinRM` není vázaný jen na obyčejné heslo. Když už útok vede přes certifikát, PKINIT nebo jiný doménový trust model, může WinRM pořád posloužit jako finální pracovní shell.

```bash
proxychains -f proxychains4.conf evil-winrm -i earth.windcorp.htb -u administrator -r WINDCORP.HTB
```

Právě tady je vidět další praktická věc: problém nemusí být autentizace, ale dostupnost služby. Když WinRM běží jen v interní síti nebo přes pivot, `Evil-WinRM` dává smysl až ve chvíli, kdy je vyřešený tunel nebo proxy. Konkrétní certifikátový kontext rozebírám v článku [AD CS a certifikáty jako cesta k Administratorovi](/techniky/ad-cs-a-certifikaty-jako-cesta-k-administratorovi).

## Kdy je `Evil-WinRM` nejlepší volba

`Evil-WinRM` se vyplácí hlavně tehdy, když:

- WinRM port skutečně odpovídá a je dosažitelný,
- už existuje validní účet, hash nebo kerberosový kontext,
- po footholdu je potřeba delší a pohodlnější PowerShell práce,
- a SMB-only workflow by bylo zbytečně křehké nebo pomalé.

Naopak nedává smysl očekávat od něj něco, co neřeší:

- neobjeví nové heslo,
- neobejde nedostupnou síťovou cestu,
- a sám o sobě nezvýší privilegia.

## Nejčastější chyby v praxi

### Zaměnění nástroje za foothold

Na blogu je dobře vidět, že `Evil-WinRM` se objevuje až po získání identity. Pokud heslo nebo hash ještě neexistuje, hledá se nejdřív v:

- Kerberosu,
- dokumentech,
- webech,
- logách,
- nebo v reuse tajemství.

Teprve potom přichází na řadu WinRM klient.

### Ignorování síťové dosažitelnosti

Na [Controlu](/control) WinRM není zvenku dosažitelný a musí se nejdřív přeposlat. Na [Anubis](/anubis) je potřeba `proxychains`. Na podobných hostech tedy problém neleží v syntaxi příkazu, ale v tom, zda se na službu dá z aktuálního místa vůbec dostat.

### Záměna lokálního a doménového kontextu

U Windows hostů je prakticky důležité vždy vědět:

- jestli se hlásíte lokálním účtem,
- nebo doménovou identitou,
- a zda je potřeba řešit realm, jméno hostu nebo Kerberos kontext.

Na jednoduchém hostu stačí `-u` a `-p`. V doménovém prostředí může rozhodnout i správný hostname, realm nebo práce přes hash.

### Představa, že shell rovná se admin

`Evil-WinRM` často otevře jen běžného uživatele. Na [Resolute](/resolute) je to `melanie`, na [Driveru](/driver) `tony`, na [Heistu](/heist) `Chase`. Skutečná privilege escalation pak stojí až na:

- transcriptech,
- spooleru,
- procesech v paměti,
- nebo delegovaných právech.

## Nejčastější praktické scénáře

### Potvrzení reuse hesla

Když heslo unikne z webu, konfigurace nebo logu, WinRM je rychlý způsob, jak ověřit, zda nejde o širší problém identity. To je přesně případ [Remote](/remote), [Heistu](/heist) nebo [Fuse](/fuse).

### Přechod z jednorázového footholdu na použitelnou pracovní session

Po webshellu, SMB enumeraci nebo cracknutí hashe dává WinRM mnohem pohodlnější prostředí než improvizovaný boční kanál. To je velmi dobře vidět na [Forestu](/forest), [Cascade](/cascade) nebo [Resolute](/resolute).

### Finální shell po AD nebo certifikátové kompromitaci

Když útok skutečně kompromituje doménovou identitu, `Evil-WinRM` bývá už jen čistý a spolehlivý způsob, jak z ní udělat administrátorský shell. To platí pro [Saunu](/sauna), [APT](/apt) i [Anubis](/anubis).

## Související články v projektu

- Rozbory: [Resolute](/resolute), [Forest](/forest), [Sauna](/sauna), [Heist](/heist), [Driver](/driver), [Remote](/remote), [Anubis](/anubis), [Cascade](/cascade)
- Techniky: [Kerberos útoky z nulového přístupu](/techniky/kerberos-utoky-z-nuloveho-pristupu-as-rep-roast-a-prace-s-kandidatnimi-uzivateli), [AD CS a certifikáty jako cesta k Administratorovi](/techniky/ad-cs-a-certifikaty-jako-cesta-k-administratorovi), [Password reuse a rozpad hranic mezi aplikací, SSH, WinRM a admin nástroji](/techniky/password-reuse-a-rozpad-hranic-mezi-aplikaci-ssh-winrm-a-admin-nastroji), [NTLM coercion a vynucená autentizace](/techniky/ntlm-coercion-a-vynucena-autentizace)

## Co si odnést do praxe

- `Evil-WinRM` je praktický shell klient pro situaci, kdy už máte identitu, ne nástroj na její získání.
- Největší hodnotu má na Windows a AD hostech po password reuse, roastu, sprayingu nebo certifikátové kompromitaci.
- Při práci s ním je důležitější dostupnost WinRM a správný autentizační kontext než samotná syntaxe příkazu.
