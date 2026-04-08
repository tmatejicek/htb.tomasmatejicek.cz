---
layout: post
author: Tomáš Matějíček
title: "windapsearch"
date: 2020-11-16
tags: nastroje windows active-directory ldap enumeration recon
---
## Úvod a kontext

`windapsearch` je v projektu užší nástroj než [Impacket](/nastroje/impacket), [CrackMapExec](/nastroje/crackmapexec) nebo [Kerbrute](/nastroje/kerbrute), ale v Active Directory dává velmi praktický smysl. Není určený k exploitu. Je to rychlý LDAP enumerátor pro okamžik, kdy chcete z adresáře vytáhnout uživatele, skupiny, SPN, admin objekty a hlavně nečekané atributy, které v sobě mohou nést užitečné tajemství.

Na [Cascade](/cascade) právě `windapsearch` ukáže `cascadeLegacyPwd`, tedy historické heslo uložené přímo v adresáři. V [Bankrobberu](/bankrobber) se stejný nástroj používá jako široká LDAP inventura po získání doménového kontextu. To je jeho typická role i obecně: nejde o "další skener", ale o rychlý způsob, jak se podívat, co o prostředí říká samotný AD.

## Co `windapsearch` v praxi řeší

V AD prostředí často potřebujete rychle odpovědět na otázky jako:

- jaké účty v doméně existují,
- které objekty vypadají administrativně zajímavě,
- kde jsou SPN,
- které účty mají nezvyklé atributy,
- a jestli se v LDAP datech neskrývá něco, co vypadá jako historické heslo nebo interní poznámka.

`windapsearch` je užitečný právě tím, že tyto odpovědi vrací rychle a v jedné pracovní sadě. Nepotřebuje složité grafové zpracování. Hodí se do chvíle, kdy si teprve skládáte mapu domény a chcete najít první použitelné stopy.

## Nejčastější scénáře využití v tomto projektu

### Hledání nečekaných atributů v nízkoprivilegovaném LDAP kontextu

[Cascade](/cascade) je nejčistší ukázka. Po anonymním RPC a základním přehledu účtů přidá `windapsearch` to, co `rpcclient` sám nevrátí tak přehledně: širší LDAP atributy, mezi nimi i `cascadeLegacyPwd`.

```bash
./windapsearch/windapsearch.py --dc-ip $IP --full --functionality -G -U -PU -C --da --admin-objects --user-spns --unconstrained-users --unconstrained-computers --gpos
```

Výstup pak vrátí:

```text
=> cascadeLegacyPwd: clk0bjVldmE=
=> userPrincipalName: r.thompson@cascade.local
```

Rozhodující není samotný nástroj, ale to, že pomůže rychle narazit na atribut, který by při úzkém hledání snadno zapadl. Tady nejde o crack. Jde o prosté přečtení historického hesla uloženého v directory datech.

### Široká LDAP inventura po získání prvních přihlašovacích údajů

Na [Bankrobberu](/bankrobber) se `windapsearch` objevuje ve chvíli, kdy už dává smysl projet celou doménu důkladněji, a to jak bez přihlášení, tak s validními údaji:

```bash
./windapsearch/windapsearch.py --dc-ip $IP --full --functionality -G -U -PU -C --da --admin-objects --user-spns --unconstrained-users --unconstrained-computers --gpos > windapsearch.txt
./windapsearch/windapsearch.py --dc-ip $IP -u user -p pass --full --functionality -G -U -PU -C --da --admin-objects --user-spns --unconstrained-users --unconstrained-computers --gpos > windapsearch.txt
```

Praktický přínos není v jedné konkrétní zranitelnosti, ale ve zkrácení první orientace:

- které účty mají SPN,
- které objekty vypadají privilegovaně,
- kde jsou unconstrained delegace,
- a jaké skupiny nebo GPO stojí za bližší kontrolu.

`windapsearch` tedy dobře funguje jako první LDAP sweep po footholdu, když už nechcete ručně sestavovat jednotlivé dotazy.

### Doplňkový nástroj vedle zbytku AD toolchainu

V místních rozborech je dobré číst `windapsearch` vedle dalších nástrojů, ne místo nich:

- [rpcclient](/nastroje/rpcclient) se hodí na první enumeraci přes RPC,
- [Kerbrute](/nastroje/kerbrute) na validaci jmenného prostoru a Kerberos foothold,
- [Impacket](/nastroje/impacket) na další práci s autentizací, relayingem a dumpem tajemství,
- a `windapsearch` mezi tím na rychlý LDAP přehled atributů a objektů.

Právě v tom je jeho praktická hodnota. Neřeší celý řetězec, ale umí dodat první kvalitní seznam indicií, ze kterého se pak vybírá další směr.

## Kdy je `windapsearch` nejlepší volba

Největší smysl dává tehdy, když:

- máte anonymní nebo nízkoprivilegovaný LDAP přístup,
- potřebujete rychle vytáhnout širší sadu atributů a objektů,
- hledáte historická hesla, SPN, admin objekty nebo neobvyklé delegace,
- a nechcete začínat složitým ručním LDAP dotazováním.

Naopak není ideální, když už přesně víte, jaký jediný atribut chcete z konkrétního objektu. Tam bývá přímý LDAP dotaz nebo specializovanější nástroj přesnější.

## Nejčastější chyby v praxi

### Očekávání, že nástroj sám "najde exploit"

Na [Cascade](/cascade) je důležité, že `windapsearch` neprolomil heslo. Jen ukázal atribut, který v sobě heslo ukrýval. Rozhodující je pořád lidská interpretace výstupu.

### Slepé pouštění širokého switche bez otázky, co hledám

Přepínače jako `--user-spns`, `--admin-objects` nebo `--unconstrained-users` jsou užitečné, jen pokud víte, proč vás tyto objekty zajímají. Jinak se z nástroje stane jen hlučný dump.

### Podcenění "nudných" atributů

Místní rozbory dobře ukazují, že nejcennější bývají právě poznámkové nebo historické atributy. Ne jen slavné věci jako SPN. `cascadeLegacyPwd` je typický příklad.

### Záměna za plnohodnotnou AD mapu

`windapsearch` je rychlý enumerátor, ne kompletní model vztahů v doméně. Jakmile jde o hlubší delegace, ACL nebo relay cesty, je potřeba ho doplnit dalšími nástroji a ruční analýzou.

## Související články v projektu

- Rozbory: [Cascade](/cascade), [Bankrobber](/bankrobber)
- Nástroje: [rpcclient](/nastroje/rpcclient), [Kerbrute](/nastroje/kerbrute), [Impacket](/nastroje/impacket)
- Techniky: [Kerberos útoky z nulového přístupu: AS-REP roast a práce s kandidátními uživateli](/techniky/kerberos-utoky-z-nuloveho-pristupu-as-rep-roast-a-prace-s-kandidatnimi-uzivateli), [Delegovaná práva v AD: DCSync, gMSA, relay, deleted objects](/techniky/delegovana-prava-v-ad-dcsync-gmsa-relay-deleted-objects), [Dokumentová metadata jako vstup do domény](/techniky/dokumentova-metadata-jako-vstup-do-domeny)

## Co si odnést do praxe

- `windapsearch` je užitečný jako rychlá LDAP inventura, ne jako exploit nástroj.
- Největší hodnotu má při hledání nečekaných atributů, SPN a admin objektů po získání prvního AD kontextu.
- V praxi dobře doplňuje `rpcclient`, `Kerbrute` a `Impacket`, protože zkracuje cestu od "mám doménu" k "vím, kde hledat další stopu".
