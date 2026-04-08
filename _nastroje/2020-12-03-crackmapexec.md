---
layout: post
author: Tomáš Matějíček
title: "CrackMapExec"
date: 2020-12-03
tags: nastroje active-directory windows smb password-spraying
---
## Úvod a kontext

`CrackMapExec` se v tomhle projektu objevuje méně často než `Impacket`, ale pokaždé v přesně stejném typu okamžiku: už existuje rozumný seznam kandidátních účtů nebo hesel a je potřeba rychle a disciplinovaně ověřit, co skutečně funguje. Prakticky tedy nejde o nástroj na "hacknutí Windows", ale o velmi užitečný validátor identity hypotéz.

Na [Heistu](/heist) zúží několik hesel z Cisco konfigurace na skutečně fungující účet `Chase`. Na [Intelligence](/intelligence) potvrdí, že výchozí heslo z dokumentu opravdu patří konkrétní doménové identitě `Tiffany.Molina`. V obou případech je důležité totéž: `CrackMapExec` neřeší celý útok, ale velmi levně rozhodne, který credential clue stojí za další krok.

## Co tenhle nástroj v praxi řeší

V prostředí AD a Windows se často dostanete do situace, kdy máte:

- seznam uživatelů,
- jedno nebo několik kandidátních hesel,
- a potřebujete rychle zjistit, zda něco z toho skutečně otevírá použitelný přístup.

Přesně tady má `CrackMapExec` největší hodnotu. Umožní rychle odpovědět na otázky typu:

- funguje tohle heslo pro někoho z tohoto seznamu,
- funguje tenhle účet proti konkrétní službě,
- a je credential clue reálná, nebo jen šum.

V tomhle projektu se objevuje hlavně nad SMB, tedy ne jako finální shell, ale jako ověření, že kombinace identita + heslo je platná v doménovém kontextu.

## Nejčastější scénáře využití v tomto projektu

### Jedno heslo proti více validním uživatelům

Na [Intelligence](/intelligence) je nejdřív potřeba vytěžit validní uživatele z dokumentových metadat a veřejných PDF. Teprve potom dává smysl ověřit, zda jedno historické výchozí heslo funguje pro některý z těchto účtů.

```bash
crackmapexec smb intelligence.htb -u users.txt -p NewIntelligenceCorpUser9876
```

Výsledek:

```text
intelligence.htb\Tiffany.Molina:NewIntelligenceCorpUser9876
```

To je velmi praktický pattern:

- nejdřív vznikne slabá identitní indicie,
- potom úzký a realistický seznam uživatelů,
- a teprve nakonec se jedním nástrojem ověří, jestli hint skutečně vede k doménovému účtu.

Širší kontext tohoto řetězce je v článku [Dokumentová metadata jako vstup do domény](/techniky/dokumentova-metadata-jako-vstup-do-domeny).

### Více hesel proti úzkému seznamu kandidátů

Na [Heistu](/heist) je situace obrácená. Účtů je omezené množství, ale z Cisco konfigurace vypadne několik různých hesel. Tady už nedává smysl ruční ověřování po jednom. `CrackMapExec` rychle zodpoví, která kombinace je reálně použitelná:

```bash
crackmapexec 10.10.10.149 -d SUPPORTDESK -u users.txt -p pass.txt
```

Výstup ukáže:

```text
Chase:Q4)sJu\Y8qz*A3?d
```

Tím se velmi levně rozhodne další směr. Není potřeba zkoušet každé heslo ručně proti WinRM nebo SMB. Stačí najít první platný pár a přejít na stabilnější kanál, v tomhle případě WinRM.

### Ověření, že credential clue má systémový dopad

To je možná nejdůležitější praktická role nástroje. `CrackMapExec` často neřeší "jak dostat shell", ale "jestli tahle indicie vůbec někam vede".

Na [Intelligence](/intelligence) ověřuje, že historický dokument nezůstal jen interní poznámkou, ale stále otevírá SMB. Na [Heistu](/heist) potvrzuje, že hesla z konfigurace síťového zařízení nezůstala omezená jen na router, ale fungují i proti doménové infrastruktuře.

Právě proto se vyplatí vnímat `CrackMapExec` jako nástroj na redukci nejistoty. Zrychlí rozhodnutí, co je skutečný credential reuse a co jen kandidát.

## Kdy je `CrackMapExec` nejlepší volba

Použitelný je hlavně tehdy, když:

- už existuje kvalitní user list nebo kandidátní heslo,
- scope je úzký a odůvodněný,
- cílem je ověřit dopad přes konkrétní protokol,
- a další krok závisí na tom, jestli kombinace skutečně funguje.

To je důležité i obranně. Nástroj sám o sobě není "silný" tím, že by obejde ochrany. Silný je tím, že zlevňuje práci se správně připravenými daty.

## Co `CrackMapExec` neumí vyřešit za vás

Nejčastější omyl je čekat od něj příliš mnoho. `CrackMapExec`:

- neobjeví uživatele sám od sebe,
- nenahradí čtení kontextu dokumentu, logu nebo konfigurace,
- a úspěch na SMB ještě automaticky neznamená interaktivní shell přes WinRM nebo jiný kanál.

Na [Intelligence](/intelligence) úspěch na SMB otevře share a čtení skriptů, ne rovnou administrátorský shell. Na [Heistu](/heist) je zase třeba po validaci účtu přejít na WinRM, protože právě tam dává další práce největší smysl.

## Nejčastější chyby při použití

### Příliš široký spray bez dobré hypotézy

Praktická hodnota nástroje rychle klesá, když se z něj stane jen hlučný password spray nad velkým prostorem. V obou relevantních rozborech funguje právě proto, že scope je úzký:

- validní uživatelé z dokumentových metadat,
- nebo krátký seznam účtů a hesel z konkrétní konfigurace.

### Záměna validace za foothold

Úspěšný výstup ještě není konec. Je to rozhodnutí, kam pokračovat.

- Na [Heistu](/heist) vede dál WinRM.
- Na [Intelligence](/intelligence) vede dál SMB share a `downdetector.ps1`.

### Ignorování charakteru cílové služby

V tomhle projektu je `CrackMapExec` použit hlavně nad SMB. To je důležité proto, že ověření proti SMB odpovídá na konkrétní otázku: "je tahle identita v doméně platná a použitelná proti SMB?" Neodpovídá na každou jinou otázku v prostředí.

## Nejčastější praktické scénáře

### Dokument, konfigurace nebo log vydá jedno heslo

Pak dává smysl ověřit ho proti úzkému seznamu validních účtů. To je čistý případ [Intelligence](/intelligence).

### Konfigurace vydá více hesel a je potřeba je rychle zúžit

Pak dává smysl vzít krátký seznam účtů a kandidátních hesel a potvrdit první skutečně použitelnou kombinaci. To je přesně [Heist](/heist).

### Potřeba rychle potvrdit credential reuse

Když existuje podezření, že heslo z jedné vrstvy funguje i jinde, `CrackMapExec` je praktický právě proto, že rychle potvrdí nebo vyvrátí dopad bez zbytečného ručního zkoušení každé dvojice.

## Související články v projektu

- Rozbory: [Heist](/heist), [Intelligence](/intelligence)
- Techniky: [Dokumentová metadata jako vstup do domény](/techniky/dokumentova-metadata-jako-vstup-do-domeny), [Konfigurace síťových zařízení jako zdroj reuse hesel](/techniky/konfigurace-sitovych-zarizeni-jako-zdroj-reuse-hesel), [Password reuse a rozpad hranic mezi aplikací, SSH, WinRM a admin nástroji](/techniky/password-reuse-a-rozpad-hranic-mezi-aplikaci-ssh-winrm-a-admin-nastroji)

## Co si odnést do praxe

- `CrackMapExec` je nejpraktičtější jako rychlý validátor credential hypotéz, ne jako nástroj, který má sám vytvořit celý foothold.
- Největší hodnotu má při úzkém scope a dobře připraveném seznamu uživatelů nebo hesel.
- Úspěch v něm je nejlepší chápat jako rozcestník: potvrzuje, která identita stojí za další práci přes SMB, WinRM nebo jiný kanál.
