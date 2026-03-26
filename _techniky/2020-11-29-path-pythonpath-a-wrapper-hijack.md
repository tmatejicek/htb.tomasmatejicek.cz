---
layout: post
author: Tomáš Matějíček
title: "PATH, PYTHONPATH a wrapper hijack"
date: 2020-11-29
tags: linux path pythonpath wrappers privesc sudo
---
## Úvod a kontext

Mnoho lokálních eskalací nevzniká z kernel exploitu ani z exotické SUID binárky. Vzniká z obyčejného provozního zvyku: někdo napíše wrapper nebo helper skript, pustí ho s vyššími právy a předpokládá, že prostředí kolem něj je důvěryhodné. Jenže právě tam leží chyba. Jakmile privilegovaný proces spoléhá na `PATH`, `PYTHONPATH` nebo jiné zděděné prostředí, předává část rozhodování útočníkovi.

Magic, ForwardSlash, Laboratory a Writeup ukazují čtyři varianty stejného vzorce. Jednou jde o binárku, která volá `lshw` bez absolutní cesty, podruhé o Python skript načítající modul z útočníkem určeného `PYTHONPATH`, potřetí o wrapper nad Dockerem a počtvrté o login hook, který zavolá `run-parts` z nesprávného místa. Na povrchu vypadají odlišně. Bezpečnostní jádro je ale stejné: root proces používá jméno nebo import, které už neřeší on sám, ale útočníkovo prostředí.

## Co se při těchto chybách skutečně pokazí

`PATH`, `PYTHONPATH` a podobné proměnné nejsou samy o sobě problém. Problém vzniká ve chvíli, kdy je používá proces s vyššími právy a zároveň:

- nevyčistí zděděné prostředí,
- nevolá binárky absolutní cestou,
- neomezuje, odkud se smí načítat moduly,
- nebo vůbec netuší, že deleguje část rozhodování na volajícího.

Z obranného pohledu je důležité vidět, že nejde o jednu konkrétní syntaxi exploitu. Jde o chybný trust model. Wrapper běžící jako root nebo přes `sudo` si myslí, že volá `docker`, `lshw`, `run-parts` nebo standardní Python knihovnu. Ve skutečnosti ale volá to, co mu podstrčí útočník.

## Tři příbuzné, ale odlišné varianty

### PATH hijack

Program nebo skript spustí externí binárku jen podle jejího jména:

```bash
docker
lshw
run-parts
tar
```

Pokud není použita absolutní cesta a privilegovaný proces přebírá `PATH` z útočníkova prostředí nebo z nesprávně sestaveného login flow, může se místo systémové utility spustit útočníkova verze.

### PYTHONPATH a import hijack

Python wrapper obvykle vypadá méně nebezpečně než shell script, protože nepůsobí jako práce s binárkami. Jenže importy jsou totéž v jiném slovníku. Když root proces importuje `shutil`, `os` nebo jiný modul a dovolí ovlivnit `PYTHONPATH`, uživatel ve skutečnosti rozhoduje, jaký kód se načte a vykoná.

### Wrapper hijack obecně

Největší problém často neleží v cílové binárce, ale v samotném wrapperu. Wrapper vznikl proto, aby něco zjednodušil:

- zabalil opakovanou administrativní akci,
- spustil Docker helper,
- zpracoval login hook,
- nebo provedl zálohu.

Právě tato "pomocná" vrstva pak bývá napsaná méně pečlivě než samotný systémový nástroj a stane se nejslabším místem celé eskalace.

## Jak tenhle vzorec vypadal v praxi

### Magic: SUID binárka a `lshw` bez absolutní cesty

Na Magic vedla root část přes SUID binárku `/bin/sysinfo`. Nešlo ale o to, že by binárka sama obsahovala shell. Důležité bylo, že interně volala `lshw` bez absolutní cesty. Jakmile útočník vytvořil vlastní `/tmp/lshw` a posunul `/tmp` na začátek `PATH`, binárka si jako root spustila podvržený soubor.

To je velmi čistý PATH hijack. Rozhodující není SUID sám o sobě, ale fakt, že privilegovaný proces nechal vyhledání binárky na shellovém prostředí místo pevně dané cesty.

### Laboratory: `docker-security` a slepá důvěra v `PATH`

Na Laboratory nevznikl root v Dockeru, ale ve wrapperu nad Dockerem. Uživatel `dexter` mohl se zvýšenými právy spustit `docker-security`, který interně volal `docker` jen podle jména. Stačilo tedy připravit vlastní soubor `/tmp/docker`, upravit `PATH` a spustit wrapper znovu:

```bash
cat > /tmp/docker <<'EOF'
#!/bin/sh
/bin/sh
EOF
chmod +x /tmp/docker
PATH=/tmp:$PATH sudo /usr/local/bin/docker-security
```

Tohle je důležitý příklad hlavně proto, že na první pohled nevypadá jako "PATH hack". Vypadá jako sudo nad interní utilitou. Ve skutečnosti ale root nepřinesl Docker, nýbrž nebezpečný wrapper, který nepřestal důvěřovat útočníkovu prostředí.

### Writeup: `run-parts` v root login workflow

Writeup ukazuje méně nápadnou variantu. Root při přihlášení volal `run-parts` bez plně kvalifikované cesty. `pspy` odhalilo, že rozhoduje `PATH`, a ten obsahoval zapisovatelné cesty dřív než systémové binárky. Jakmile se do `/usr/local/bin` podstrčil vlastní `run-parts`, spustil se při dalším loginu místo originálu.

Tady je důležité něco jiného než u Magica. Nešlo o ruční `sudo` volání z interaktivního shellu, ale o privilegovaný login workflow. To dobře připomíná, že wrapper hijack není jen post-exploitation trik nad `sudo -l`. Může sedět i v `cron`, PAM, login hooku nebo jiné automatizaci.

### ForwardSlash: `PYTHONPATH` a import modulu `shutil`

ForwardSlash přidává Python variantu stejného problému. Účet `chiv` mohl přes `sudo` spouštět `/usr/bin/backup`, Python wrapper, který importoval standardní moduly bez ochrany proti změně importní cesty. Jakmile se do `/dev/shm` vytvořil škodlivý `shutil.py` a `PYTHONPATH` se nasměroval tam, root proces načetl útočníkův kód místo standardní knihovny:

```bash
cat > /dev/shm/shutil.py <<'EOF'
import os
os.system("cp /bin/bash /tmp/rootshell && chmod 4777 /tmp/rootshell")
EOF

sudo PYTHONPATH=/dev/shm /usr/bin/backup
```

To je přesně chvíle, kdy je dobré oddělit PATH a PYTHONPATH. Mechanika je jiná, ale chyba důvěry stejná: privilegovaný proces si nechá od útočníka vysvětlit, odkud má načíst to, co považuje za "interní" součást svého běhu.

## Jak tyto chyby hledat po footholdu

### 1. `sudo -l` číst jako mapu wrapperů, ne jen binárek

Výpis `sudo -l` je dobrý začátek, ale rozhodující bývá až další otázka: co ten nástroj interně volá dál.

```bash
sudo -l
```

Zvýšenou pozornost si zaslouží:

- interní helpery v `/usr/local/bin`,
- Python a shell wrappery,
- backup utility,
- docker helpery,
- a všechno, co zní jako "security", "maintenance", "status" nebo "check".

### 2. Číst skript nebo binárku před pokusem o exploit

U shell scriptu stačí často jeden pohled:

```bash
cat /usr/local/bin/suspektni-wrapper
```

U binárky pomůže `strings`, `ltrace`, `strace` nebo prosté čtení chování v `pspy`. Cílem není hned exploit. Cílem je zjistit:

- jaké externí nástroje wrapper volá,
- zda jsou zadané absolutní cestou,
- a zda běh přebírá prostředí od uživatele.

### 3. U Pythonu nehledat jen `os.system`

U Python wrapperu není důležité jen to, zda explicitně volá shell. Stejně důležité je:

- odkud importuje moduly,
- zda běží s `PYTHONPATH` převzatým z prostředí,
- a zda používá vlastní adresáře nebo virtuální prostředí bez pevné kontroly importů.

### 4. Sledovat periodické a login procesy

Writeup dobře ukazuje, že privilegované volání bez absolutní cesty nemusí sedět v souboru, který hned vidíte. Může se objevit až v periodické úloze nebo při loginu. Proto mají nástroje jako `pspy` po footholdu tak vysokou hodnotu.

## Jak si tyto chyby nesplést s jinými lokálními privescy

PATH/PYTHONPATH hijack se snadno plete s jinými kategoriemi:

- není to totéž co SUID na shellu,
- není to totéž co `sudo` nad package managerem,
- a není to totéž co wildcard injection.

Spojuje je až to, že privilegovaný proces pustí do rozhodování něco, co má být naopak pevně pod kontrolou systému. U PATH hijacku je to jméno binárky. U PYTHONPATH hijacku import modulu. U wildcard injection interpretace argumentu jiným nástrojem. Prakticky se vyplatí tyto kategorie oddělovat, protože každá má jinou obranu.

## Obrana a bezpečnější návrh

### 1. Volat binárky absolutní cestou

Privilegovaný skript nebo binárka nemají volat `docker`, `tar` nebo `run-parts` jen jménem. Mají volat přesně `/usr/bin/docker`, `/bin/tar` nebo jinou explicitní cestu.

### 2. Resetovat a minimalizovat prostředí

Wrapper běžící přes `sudo`, cron nebo login hook má dostat minimální a předvídatelné prostředí. Pokud root proces přebírá uživatelský `PATH` nebo `PYTHONPATH`, je trust boundary rozbitá už v návrhu.

### 3. Python importy držet na pevných, důvěryhodných cestách

U Pythonu nestačí říct "nevoláme shell". Pokud script importuje moduly z neřízeného prostředí, může se vlastní import stát ekvivalentem spuštění cizího kódu.

### 4. Wrappery auditovat stejně přísně jako cílové utility

Interní helper často vznikne jako rychlé provozní zjednodušení. Právě proto potřebuje pečlivější review, ne méně. Laboratory i ForwardSlash ukazují, že skutečný problém neležel v Dockeru ani v Pythonu samotném, ale v tenké vrstvě okolo.

### 5. Monitorovat neobvyklé změny `PATH` a importního prostředí

Na produkčním systému je podezřelé:

- když privilegovaný wrapper běží s `PATH` začínajícím v `/tmp`,
- když někdo přes `sudo` nastavuje `PYTHONPATH`,
- nebo když login hook sáhne po binárce z `/usr/local/bin` místo ze systémové cesty.

## Shrnutí klíčových poznatků

- PATH, PYTHONPATH a wrapper hijack jsou různé projevy stejné chyby: privilegovaný proces důvěřuje prostředí, které ovládá méně privilegovaný uživatel.
- Magic, Laboratory, Writeup a ForwardSlash ukazují čtyři varianty stejného vzorce: SUID binárku, `sudo` wrapper, login hook a Python import.
- Největší hodnotu má při auditu otázka "co tento proces volá dál a odkud to bere", ne samotný název nástroje.

## Co si odnést do praxe

- Když uvidíte privilegovaný wrapper, neptejte se jen co dělá. Ptejte se, co spouští, odkud to bere a kdo kontroluje jeho prostředí.
- Absolutní cesty nejsou kosmetika. U root procesů rozhodují o tom, zda se spustí systémová binárka, nebo útočníkův podvrh.
- Python wrapper bez pevně řízených importů není bezpečný jen proto, že "to není shell script". Pokud lze ovlivnit `PYTHONPATH`, je to plnohodnotná cesta k vykonání kódu s vyššími právy.
