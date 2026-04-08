---
layout: post
author: Tomáš Matějíček
title: "ldapsearch"
date: 2020-11-16
tags: nastroje ldap active-directory enumeration attributes directory
---
## Úvod a kontext

`ldapsearch` je nízkoúrovňový klient pro přímé dotazy do LDAP adresáře. V místních rozborech se častěji objevují specializované utility jako [windapsearch](/nastroje/windapsearch), [Kerbrute](/nastroje/kerbrute) nebo [Impacket](/nastroje/impacket), ale vrstva dat, ze které tyto nástroje žijí, je pořád stejná: LDAP.

Právě proto má `ldapsearch` praktický smysl. Když nepotřebujete širokou enumeraci, ale chcete si ručně ověřit konkrétní atribut, objekt nebo search base, bývá to nejpřímější cesta. To je důležité hlavně u AD scénářů typu [Cascade](/cascade), [Monteverde](/monteverde), [Forest](/forest) nebo [Intelligence](/intelligence), kde rozhodují atributy, skupiny a directory metadata, ne jen otevřené porty.

## Co `ldapsearch` v praxi řeší

`ldapsearch` je užitečný tehdy, když potřebujete:

- zjistit naming context a základ adresáře,
- potvrdit, že bind skutečně funguje,
- vypsat konkrétní objekty nebo atributy,
- ověřit, jestli LDAP vrací něco citlivého i s nízkými právy,
- nebo si zúžit pohled na jeden účet, skupinu či OU bez hlučného dumpu celé domény.

Jeho síla není v automatizaci. Je v přesnosti. Přesně proto se hodí jako doplněk ve chvíli, kdy už nějaký AD kontext máte a potřebujete ověřit jednu hypotézu nad adresářovými daty.

Typicky to znamená nejdřív potvrdit naming context a potom cíleně vytáhnout konkrétní objekt:

```bash
ldapsearch -x -H ldap://10.10.10.182 -s base namingContexts
ldapsearch -x -H ldap://10.10.10.182 -D 'SABatchJobs@MEGABANK.LOCAL' -w 'SABatchJobs' -b 'DC=MEGABANK,DC=LOCAL' '(sAMAccountName=SABatchJobs)' dn memberOf
```

## Nejčastější scénáře využití v tomto projektu

### Ověření podezřelého atributu nebo konkrétního objektu

[Cascade](/cascade) ukazuje, jak velkou hodnotu může mít jediný LDAP atribut. Praktický průlom tam nepřišel z exploitu služby, ale z atributu `cascadeLegacyPwd`, který byl uložený přímo u účtu.

V místním rozboru se to vytahuje přes [windapsearch](/nastroje/windapsearch), ale právě tady by `ldapsearch` byl přirozený nástroj pro cílené ověření:

- vrátit jeden konkrétní účet,
- zobrazit jeho atributy,
- a rychle zjistit, zda v nich neleží historické heslo nebo interní poznámka.

To je praktická síla `ldapsearch`: když už tušíte, co hledat, není potřeba spouštět široký dump celé domény.

### Potvrzení bindu a search base po získání prvních doménových údajů

[Monteverde](/monteverde) je dobrá ukázka chvíle, kdy LDAP přestává být jen služba na portu `389` a stává se skutečným zdrojem dalších stop. Jakmile funguje jednoduchý spray a účet `SABatchJobs`, dává smysl ověřit:

- jaký je naming context,
- jaké objekty jsou pod danou base dostupné,
- a co všechno vrací obyčejný bind.

Právě v takové chvíli `ldapsearch` šetří čas. Místo slepého zkoušení dalších utilit si můžete nejdřív potvrdit, co LDAP skutečně vrací a jak hluboko s daným účtem do adresáře dohlédnete.

### Ruční ověření toho, co našel širší enumerátor

[Forest](/forest), [Intelligence](/intelligence) i [Monteverde](/monteverde) ukazují širší AD řetězce, kde se střídají:

- seznamy účtů,
- servisní identity,
- gMSA,
- delegovaná práva,
- nebo podezřelé skupiny.

V takových situacích často dává smysl použít `ldapsearch` jako ruční kontrolu:

- jestli objekt opravdu má daný atribut,
- pod jakým DN leží,
- jak se jmenuje skupina nebo service account,
- a jak přesně vypadá jeho directory záznam.

Nejde o náhradu specializovaných nástrojů. Jde o přesné ověření mezi dvěma kroky, kdy už nechcete hádat, co enumerátor vlastně našel.

### LDAP jako zdroj dat pro další identity hypotézy

Na [Sauně](/sauna) nebo [Intelligence](/intelligence) stojí další směr útoku na dobrém seznamu jmen a identit. `ldapsearch` se v podobných chvílích hodí tehdy, když chcete:

- ověřit existenci konkrétního účtu,
- najít jeho DN nebo UPN,
- nebo zúžit výstup jen na malou skupinu objektů místo celé domény.

Prakticky je to často levnější než rovnou sahat po těžší orchestraci nad celou AD mapou.

## Kdy je `ldapsearch` nejlepší volba

Největší smysl dává tehdy, když:

- už máte alespoň anonymní nebo nízkoprivilegovaný bind,
- potřebujete cílený dotaz na konkrétní objekt nebo atribut,
- chcete si potvrdit search base, DN nebo naming context,
- nebo ručně ověřujete výsledek z jiného AD nástroje.

Naopak není ideální jako jediný nástroj pro široký první recon celé domény. Tam bývají praktičtější [Kerbrute](/nastroje/kerbrute), [windapsearch](/nastroje/windapsearch), [rpcclient](/nastroje/rpcclient) nebo další specializované utility.

## Nejčastější chyby v praxi

### Očekávání, že LDAP vrátí vše bez ohledu na bind

`ldapsearch` je přesný, ale pořád naráží na práva adresáře. Když účet nic nevidí, problém není v syntaxi příkazu, ale v tom, jaký máte bind kontext.

### Záměna široké enumerace za cílený dotaz

Jestli teprve hledáte první stopu v celé doméně, ruční `ldapsearch` bývá zbytečně pomalý. Je silný až tehdy, když už víte, co nebo koho chcete potvrdit.

### Podcenění konkrétních atributů

[Cascade](/cascade) dobře ukazuje, že nejcennější bývá někdy právě nenápadný atribut. `ldapsearch` má smysl tehdy, když se umíte ptát konkrétně, ne když jen bez cíle dumpujete vše.

### Ignorování toho, že LDAP je zdroj, ne cíl

Na místních strojích LDAP většinou nevede přímo k shellu. Vede k:

- heslu,
- skupině,
- delegaci,
- servisní identitě,
- nebo lepšímu pochopení AD struktury.

To je jeho skutečná role.

## Související články v projektu

- Rozbory: [Cascade](/cascade), [Monteverde](/monteverde), [Forest](/forest), [Intelligence](/intelligence), [Sauna](/sauna)
- Nástroje: [windapsearch](/nastroje/windapsearch), [Kerbrute](/nastroje/kerbrute), [rpcclient](/nastroje/rpcclient), [Impacket](/nastroje/impacket)
- Techniky: [Kerberos útoky z nulového přístupu: AS-REP roast a práce s kandidátními uživateli](/techniky/kerberos-utoky-z-nuloveho-pristupu-as-rep-roast-a-prace-s-kandidatnimi-uzivateli), [Delegovaná práva v AD: DCSync, gMSA, relay, deleted objects](/techniky/delegovana-prava-v-ad-dcsync-gmsa-relay-deleted-objects), [Dokumentová metadata jako vstup do domény](/techniky/dokumentova-metadata-jako-vstup-do-domeny)

## Co si odnést do praxe

- `ldapsearch` je užitečný jako přesný nízkoúrovňový klient pro ruční dotazy do LDAP, ne jako hlučný všeobjímající enumerátor.
- Největší hodnotu má po získání prvního AD kontextu, když potřebujete ověřit konkrétní objekt, atribut nebo search base.
- V praxi dobře doplňuje specializované AD utility, protože dovolí rychle potvrdit přesně to, co ostatní nástroje jen naznačily.
