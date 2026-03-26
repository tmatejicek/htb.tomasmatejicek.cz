---
layout: post
author: Tomáš Matějíček
title: "`sudo` nad package, backup a container nástroji"
date: 2020-10-31
tags: sudo linux privesc containers backup packages
---
## Úvod a kontext

`sudo` pravidlo často vypadá bezpečně jen proto, že neukazuje `/bin/sh`. Místo toho dovolí spustit něco, co na první pohled zní úzce administrativně: `snap install`, `pkg install`, `restic backup` nebo `docker exec`. Jenže právě takové nástroje bývají nejnebezpečnější. Nejsou to "omezené utility". Jsou to programy navržené k tomu, aby instalovaly cizí obsah, četly citlivá data nebo manipulovaly s izolovanými runtime prostředími.

Právě proto je chyba hodnotit `sudo` jen podle názvu binárky. Smysluplnější je dívat se na to, co daný nástroj strukturálně umí:

- spouštět instalační hooky,
- číst a exportovat data z privilegovaných cest,
- sahat do container namespace,
- nebo delegovat další vykonávání kódu.

Armageddon, Schooled, Registry a TheNotebook ukazují čtyři různé varianty stejného problému. `sudo` zde nedává "malou administrativu". Dává nepřímý root.

## Proč jsou tyto nástroje rizikové už ze své podstaty

### Package manager není jen kopírování souborů

Balíčkovací nástroje typicky umí:

- rozbalit cizí archiv,
- zapsat soubory na privilegovaná místa,
- spustit pre/post install hook,
- měnit služby nebo metadata systému.

Pokud tedy uživatel smí přes `sudo` instalovat libovolný balíček, smí obvykle i dodat kód, který se při instalaci spustí jako root.

### Backup nástroj není jen archivace

Zálohovací nástroje často čtou:

- libovolnou cestu,
- cizí domácí adresáře,
- SSH klíče,
- nebo celý `/root`.

Jakmile přes `sudo` dovolíte spustit backup s uživatelsky ovlivnitelným cílem nebo backendem, ve skutečnosti delegujete root read access. To je méně nápadné než shell, ale dopadově skoro stejné.

### Container runtime není "jen kontejner"

`docker exec`, `podman`, `ctr` a podobné utility manipulují s procesy, mounty a namespace. Pokud přes `sudo` dovolíte spouštět příkazy uvnitř kontejneru, ve skutečnosti uživateli půjčujete kontrolu nad prostředím, které je těsně navázané na hosta. Na starších verzích runtime to může být přímá cesta k host rootu; i bez známé CVE to ale často znamená přístup k tajemstvím, mountům nebo interním službám.

## Jak tenhle vzorec vypadal v praxi

### Armageddon: `snap install *` jako instalační hook pod rootem

Na Armageddonu vycházelo:

```text
(root) NOPASSWD: /usr/bin/snap install *
```

To vypadá konkrétněji než `sudo su`, ale bezpečnostní význam je téměř stejný. `snap` umí instalovat balíček dodaný uživatelem a během instalace provádět akce definované v jeho metadatech. Jakmile si útočník připraví vlastní snap, změní se zdánlivě omezené `sudo` pravidlo ve spouštění vlastního kódu jako root.

Podstatná není konkrétní syntaxe exploitu. Podstatné je, že instalace balíčku je vždycky výkon privilegované důvěry nad cizím artefaktem.

### Schooled: `pkg install *` a `+PRE_INSTALL`

Schooled ukazuje stejný princip na FreeBSD. Uživatel `jamie` směl přes `sudo` spouštět:

```text
(ALL) NOPASSWD: /usr/sbin/pkg update
(ALL) NOPASSWD: /usr/sbin/pkg install *
```

Rozhodující detail byl v tom, že balíček může obsahovat `+PRE_INSTALL` skript. Stačilo tedy připravit minimální balíček s instalačním hookem a nechat `pkg` udělat zbytek:

```bash
pkg create -m shell/ -r shell/ -o .
sudo /usr/sbin/pkg install --no-repo-update ./shell-1.0_5.txz
```

Na tomhle příkladu je dobře vidět, proč nestačí argumentovat "ale uživatel nemůže spustit shell přímo". Nemusí. Shell spustí instalační životní cyklus nástroje za něj.

### Registry: `restic backup` jako delegované čtení root dat

Na Registry byl problém méně nápadný, ale stejně závažný:

```text
(root) NOPASSWD: /usr/bin/restic backup -r rest*
```

Tady nejde primárně o vykonání kódu, ale o čtení. `restic` je navržený tak, aby četl data s vysokými právy a posílal je na backend. Jakmile si útočník připraví vlastní `rest-server` a přesměruje na něj zálohu, získá obsah cest, které by jinak číst nesměl. Na Registry to vedlo až k rootovu SSH klíči.

To je důležitá lekce: privilegované čtení je plnohodnotná forma eskalace. Pokud může uživatel přes `sudo` nechat root proces vyexportovat `/root`, rozdíl proti přímému shellu je už hlavně kosmetický.

### TheNotebook: `docker exec` a kontrola nad runtime

TheNotebook přidal čtvrtou variantu. Uživatel `noah` směl spouštět:

```text
(ALL) NOPASSWD: /usr/bin/docker exec -it webapp-dev01*
```

Samo o sobě to znamená přístup do běžícího kontejneru. Jenže container runtime je natolik privilegovaná součást hostu, že i takto úzce vypadající právo může být kritické. V tomto případě běžela stará verze Dockeru a `runc`, takže šlo přes `docker exec` zneužít `CVE-2019-5736` a přepsat hostitelský `runc`.

I bez konkrétní CVE by ale podobné oprávnění bylo vysoce rizikové. `docker exec` dává možnost pracovat s mounty, tajemstvími a procesy v prostředí, které je bezpečnostně mnohem blíž rootovi než běžný uživatelský shell.

## Jak takové `sudo` pravidlo hodnotit při review

### 1. Ptát se, kdo ovládá vstup

Nejdůležitější otázka není "je cesta k binárce pevná?", ale "kdo ovládá to, co ten nástroj zpracuje?".

U package manageru je to:

- balíček,
- manifest,
- instalační hook,
- repozitář nebo lokální soubor.

U backup toolu je to:

- zdrojová cesta,
- backend,
- snapshot nebo restore target.

U container nástroje je to:

- cílový kontejner,
- spouštěný příkaz,
- bind mount,
- nebo runtime verze.

### 2. Rozlišit write, read a execute dopad

Ne každé riziko končí reverzním shellem. Prakticky ale stačí i to, že nástroj:

- umí zapsat root-owned soubor,
- umí přečíst root-only data,
- nebo umí spustit cizí hook.

Registry je přesně ten případ, kde čtení citlivých dat bylo stejně fatální jako přímé RCE.

### 3. Wildcard a prefix restrikce nebrat jako skutečné omezení

`snap install *`, `pkg install *` nebo `restic backup -r rest*` vypadají restriktivně. V praxi ale často jen říkají, že nástroj musí dostat nějaký argument určitého tvaru. Pokud ten argument pořád ukazuje na útočníkem kontrolovaný artefakt nebo backend, bezpečnostní hodnota takového omezení je malá.

### 4. Hodnotit nástroj podle jeho životního cyklu

Správný audit neřeší jen vstupní argumenty, ale celý životní cyklus programu:

- co dělá při instalaci,
- odkud načítá metadata,
- kam sahá při záloze,
- co se stane při `exec` do kontejneru,
- jaké hooky, pluginy nebo callbacky umí.

Právě tam obvykle leží skutečná cesta k eskalaci.

## Jak takové případy hledat po footholdu

Po získání shellu dává smysl `sudo -l` nečíst jako prostý seznam příkazů, ale jako mapu privilegií.

```bash
sudo -l
```

Zvýšenou pozornost si zaslouží hlavně nástroje kolem:

- instalace balíčků,
- záloh a restore,
- kontejnerů a image,
- správy služeb,
- a všech wrapperů, které jen předávají práci dál jinému privilegovanému procesu.

Praktická heuristika je jednoduchá: čím víc je nástroj určený pro správu systému, tím menší smysl dává považovat ho za "bezpečně omezené sudo".

## Obrana a bezpečnější návrh

### 1. Nedávat `sudo` přímo na silné administrační utility

Pokud uživatel potřebuje instalovat konkrétní balíček nebo spustit konkrétní backup, bezpečnější je úzký wrapper s pevnými parametry, ne přímý přístup k univerzálnímu nástroji.

### 2. Omezit vstupy, ne jen binárku

Bezpečné není "smí spustit `restic`". Bezpečné je teprve "smí spustit přesně tenhle wrapper, který zálohuje přesně tuhle cestu na přesně tenhle backend". Totéž platí pro `docker exec` i balíčkovací nástroje.

### 3. Oddělit role pro čtení, instalaci a runtime správu

Účet, který spravuje balíčky, zálohy nebo kontejnery, má z definice vysokou hodnotu. Nemá být zároveň běžným uživatelem aplikace nebo webového procesu.

### 4. Auditovat i read-only dopad

Obranná revize nesmí sledovat jen "lze spustit příkaz?". Musí sledovat i "lze tím přečíst citlivá data nebo exportovat tajemství?". U backup toolů je právě tohle hlavní riziko.

### 5. Monitorovat neobvyklé použití těchto nástrojů přes `sudo`

Na produkčním hostu bývá podezřelé:

- ruční `snap install` nebo `pkg install` z netypické cesty,
- backup na nečekaný backend,
- nebo `docker exec` z účtu, který běžně kontejnery neobsluhuje.

## Shrnutí klíčových poznatků

- `sudo` nad package, backup a container nástroji je nebezpečné ne kvůli názvu binárky, ale kvůli jejímu účelu.
- Package managery často dovolují spustit instalační hooky, backup nástroje čtou citlivá data a container utility manipulují s vysoce privilegovaným runtime prostředím.
- Armageddon, Schooled, Registry a TheNotebook ukazují čtyři různé cesty ke stejnému závěru: zdánlivě omezené administrační `sudo` pravidlo může být ve skutečnosti nepřímý root shell nebo root read access.

## Co si odnést do praxe

- Při auditu `sudo` se neptejte jen "je tam shell?". Ptejte se, jestli daný nástroj umí číst, zapisovat nebo vykonávat něco s root důvěrou nad útočníkem ovlivnitelným vstupem.
- U package managerů, backup toolů a container utility je správný výchozí předpoklad opačný: nejsou bezpečné, dokud se velmi pečlivě neprokáže opak.
- Wildcard nebo prefix v argumentu obvykle nestačí. Jestli nástroj pořád zpracovává cizí balíček, cizí backend nebo cizí runtime kontext, jde dál o plnohodnotnou cestu k eskalaci.
