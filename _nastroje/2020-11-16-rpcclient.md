---
layout: post
author: Tomáš Matějíček
title: "rpcclient"
date: 2020-11-16
tags: nastroje active-directory windows rpc smb enumeration
---
## Úvod a kontext

`rpcclient` je v tomto projektu užitečný hlavně jako rychlý, levný a často podceňovaný zdroj identity a provozních informací z Windows prostředí. Není to nástroj na přímý shell. Je to nástroj, který z Microsoft RPC a SAMR udělá čitelná data pro další rozhodnutí.

Na [Cascade](/cascade) vrací seznam účtů ještě bez přihlášení. Na [Resolute](/resolute) odhalí v popisku účtu výchozí heslo `Welcome123!`. Na [Fuse](/fuse) pomůže přejít od běžného doménového účtu k servisním identitám a nakonec i k heslu ukrytému v popisu tiskárny. To je jeho praktická role: neřeší exploit, ale překládá RPC metadata do dalších útokových hypotéz.

## Co `rpcclient` v praxi řeší

`rpcclient` je klient pro různé RPC operace nad Windows a Samba službami. Prakticky odpovídá na otázky:

- jaké účty v doméně nebo na hostu existují,
- co je o nich uložené v popiscích a atributech,
- jaké tiskárny, sdílení nebo další objekty jsou přes RPC viditelné,
- a jestli i minimální účet nebo anonymní přístup nevydá víc, než by měl.

Jeho největší hodnota je v tom, že často poskytne strukturovanější a rychlejší výstup než ruční klikání nebo těžší LDAP workflow.

## Nejčastější scénáře využití v tomto projektu

### Anonymní enumerace účtů

Na [Cascade](/cascade) je `rpcclient` první opravdu hodnotný krok. Bez přihlášení vrátí seznam účtů:

```bash
rpcclient -W "" -U "%" -c querydispinfo $IP
```

To je důležité, protože útok tím získá:

- jmenný prostor organizace,
- přehled o relevantních identitách,
- a základ pro další vazbu na LDAP nebo SMB.

Tady `rpcclient` neřeší hesla. Řeší mapu identity prostoru, bez které další práce tápe.

### Čtení provozních poznámek v popiscích účtů

Na [Resolute](/resolute) je nejdůležitější právě to, že `querydispinfo` nevrátí jen seznam jmen, ale i lidský popisek:

```bash
rpcclient -W "" -U "%" -c querydispinfo $IP
```

```text
Account: marko Name: Marko Novak Desc: Account created. Password set to Welcome123!
```

Tady je praktická hodnota obrovská:

- nejde o crack,
- nejde o exploit,
- jde o prosté přečtení operativní poznámky,
- která otevře realistickou credential hypotézu pro další účty.

To je přesně důvod, proč se `rpcclient` vyplatí nepřehlížet ani v době bohatších AD nástrojů.

### Přechod od běžného účtu k servisním identitám

Na [Fuse](/fuse) už `rpcclient` neběží anonymně, ale pod doménovým účtem `tlavel`. Přesto řeší podobnou úlohu: vytáhnout informace, které posunou útok k hodnotnějším identitám.

```bash
rpcclient -U FABRICORP\\tlavel -c enumdomusers $IP
rpcclient -U FABRICORP\\tlavel -c enumprinters $IP
```

Právě `enumprinters` odhalí v popisku tiskárny heslo `scan2docs password: $fab@s3Rv1ce$1`. To je velmi dobrá ukázka, že `rpcclient` není jen "výpis uživatelů". Umí se stát nástrojem pro čtení provozních detailů, které mají okamžitý dopad na laterální pohyb.

## Kdy je `rpcclient` nejlepší volba

Největší smysl dává tehdy, když:

- cílem je windowsový host nebo AD prostředí s otevřeným SMB/RPC,
- potřebujete rychle vytáhnout účty, popisky nebo tiskárny,
- a existuje šance, že i minimální nebo anonymní přístup vydá hodnotná metadata.

V takových situacích bývá rychlejší než složitější LDAP enumerace a zároveň přesnější než slepé zkoušení dalších služeb.

## Co `rpcclient` neumí vyřešit za vás

`rpcclient`:

- sám nevytvoří foothold,
- neprolomí heslo,
- neříká, který účet bude provozně použitelný,
- a závisí na tom, co cílový host nebo doména přes RPC skutečně vystaví.

Na [Cascade](/cascade) je cenný proto, že anonymně vydá účty. Na [Resolute](/resolute) proto, že vydá heslovou nápovědu. Na [Fuse](/fuse) proto, že pod běžným účtem ukáže servisní stopy. Bez tohohle kontextu by šlo jen o další enumeraci bez dopadu.

## Nejčastější chyby při použití

### Ignorování popisků a vedlejších polí

Největší praktická hodnota v projektu neleží v samotném seznamu uživatelů. Leží v tom, co je u nich nebo u jiných objektů napsané:

- výchozí heslo,
- provozní poznámka,
- nebo interní servisní detail.

### Příliš brzké přeskočení na těžší nástroje

Když `rpcclient` umí rychle vrátit účty nebo tiskárny, není důvod začínat vždy rovnou složitější LDAP sadou. Naopak často dává nejlepší poměr cena/výsledek.

### Záměna enumerace za kompromitaci

Úspěšný výpis ještě neznamená foothold. Je to ale velmi často přesně ten krok, který otevře další heslovou, smb nebo winrm hypotézu.

## Nejčastější praktické scénáře

### Potřebuji rychle zjistit, jaké účty v prostředí existují

To je [Cascade](/cascade).

### Potřebuji přečíst provozní poznámky v účtech

To je [Resolute](/resolute).

### Potřebuji zjistit, jaké servisní stopy a tiskárny jsou dostupné přes RPC

To je [Fuse](/fuse).

## Související články v projektu

- Rozbory: [Cascade](/cascade), [Resolute](/resolute), [Fuse](/fuse)
- Techniky: [Kerberos útoky z nulového přístupu](/techniky/kerberos-utoky-z-nuloveho-pristupu-as-rep-roast-a-prace-s-kandidatnimi-uzivateli), [Password reuse a rozpad hranic mezi aplikací, SSH, WinRM a admin nástroji](/techniky/password-reuse-a-rozpad-hranic-mezi-aplikaci-ssh-winrm-a-admin-nastroji), [Delegovaná práva v AD: DCSync, gMSA, relay, deleted objects](/techniky/delegovana-prava-v-ad-dcsync-gmsa-relay-deleted-objects)

## Co si odnést do praxe

- `rpcclient` je v praxi nejcennější jako rychlý zdroj identitních a provozních metadat z Windows prostředí.
- Největší hodnotu má tehdy, když z něj nečtete jen seznam uživatelů, ale i popisky, tiskárny a další vedlejší objekty.
- Sám o sobě útok neuzavírá, ale často velmi levně otevře další credential nebo laterální hypotézu.
