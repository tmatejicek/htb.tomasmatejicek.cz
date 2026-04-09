---
layout: post
author: Tomáš Matějíček
title: "SUID/GTFOBins a netypické binárky"
date: 2020-12-07
tags: suid gtfobins linux privesc binaries
---
## Úvod a kontext

SUID audit se často redukuje na krátký checklist: najdi `find`, `vim`, `nmap`, porovnej výstup s GTFOBins a zkus známý trik. To je užitečné pro rychlý screening, ale nestačí to pro skutečné posouzení rizika. Na produkčních hostech se totiž častěji objeví binárka, která na první pohled nevypadá jako "klasický eskalační kandidát": `systemctl`, `jjs`, vlastní helper nebo dokonce shell s omylem ponechaným SUID bitem.

Právě proto dává smysl uvažovat o SUID systematicky. Důležitá není popularita binárky, ale její chování:

- umí spouštět další procesy,
- umí číst nebo zapisovat citlivé soubory,
- umí načítat skripty, pluginy nebo jednotky,
- nebo je sama interpretem či shellem.

[Jarvis](/jarvis), [Mango](/mango) a [Rope](/rope) ukazují tři různé varianty stejného principu. Jedna binárka spravuje systemd, druhá interpretuje JavaScript s přístupem k Java API a třetí je obyčejný shell. Všechny ale při SUID bitu překračují hranici mezi běžným uživatelem a rootem.

## Proč seznam SUID binárek sám nestačí

Výpis jako:

```bash
find / -perm -4000 -type f 2>/dev/null
```

je jen začátek. Samotný seznam neřekne:

- zda binárka umí spouštět externí program,
- jestli bere uživatelský vstup jako skript nebo konfiguraci,
- jestli umí pracovat se soubory mimo vlastní sandbox,
- nebo zda jen nepůsobí neškodněji, než ve skutečnosti je.

GTFOBins je dobrý index, ale není to rozhodovací model. Jakmile se na hostu objeví méně běžný nástroj nebo distribuční specifikum, je potřeba přemýšlet nad tím, co ta binárka dělá, ne jen jestli už na ni někdo sepsal hotový one-liner.

## Jak přemýšlet nad netypickou SUID binárkou

### 1. Umí spustit jiný proces

To je nejpřímější cesta k eskalaci. Nemusí jít o explicitní `execve()`. Stačí, že nástroj umí:

- vytvořit službu,
- zavolat shell,
- otevřít editor,
- nebo vyvolat jiný program přes hook či callback.

`systemctl` na Jarvisu je přesně ten případ. Není to shell, ale umí vytvořit a spustit systemd jednotku s libovolným `ExecStart`.

### 2. Umí číst nebo zapisovat libovolné cesty

Ne každá eskalace potřebuje shell. Pokud binárka s efektivním UID root umí otevřít libovolný soubor, už sama o sobě může stačit ke kompromitaci:

- přečte `/root/root.txt`,
- vytáhne SSH klíč,
- nebo změní obsah citlivého souboru.

`jjs` na Mangu je dobrý příklad. Neotevíral rovnou root shell, ale umožnil přes Java API číst `/root/root.txt`. V praxi je to plnohodnotný root-read primitivum.

### 3. Je to interpreter, runtime nebo shell

Jakmile je SUID nastavený na obecném interpretru nebo shellu, bývá situace téměř vždy fatální. Shell, který zachová efektivní UID, nepotřebuje žádnou kreativní post-exploitation logiku. Prostě běží s root právy.

Rope je ukázkový extrém. Po technicky náročné user části stačilo najít `/bin/bash` se SUID bitem a spustit:

```bash
/bin/bash -p
```

Taková chyba je obranně katastrofální právě proto, že ruší smysl celé předchozí hardening vrstvy.

### 4. Umí pracovat s konfigurací, pluginem nebo jednotkou

Mnoho administrativních nástrojů není nebezpečných kvůli jednomu argumentu, ale kvůli tomu, že zpracovávají uživatelsky dodanou konfiguraci. To je opět Jarvis:

- `systemctl` samo o sobě není "exploit tool",
- ale umí linkovat, povolovat a spouštět jednotky,
- a jednotka nese vlastní příkaz, který poběží jako root.

To je obecně silná heuristika. Pokud SUID binárka zpracovává externí jednotku, skript, šablonu nebo manifest, je potřeba ji auditovat jako cestu ke spuštění kódu.

## Jak to vypadalo v konkrétních případech

### Jarvis: SUID `systemctl` a vlastní systemd jednotka

Na [Jarvisu](/jarvis) ukázal [linpeas](/nastroje/linpeas) mezi SUID binárkami `systemctl`. To samo o sobě už je varovný signál, protože systemd je centrální orchestrátor služeb. Praktický dopad byl přímočarý: stačilo vytvořit dočasný `.service` soubor s vlastním `ExecStart`, přilinkovat ho a nechat `systemctl` službu spustit.

Tady je důležité pochopit, proč je to nebezpečné. `systemctl` není problém jen proto, že "někdo našel trik". Je nebezpečné z definice, protože umí delegovat spouštění procesu do systemd s root kontextem.

### Mango: SUID `jjs` a file-read přes Java API

Na [Mangu](/mango) byl root mnohem méně nápadný. Mezi SUID soubory se objevil `jjs`, JavaScript shell pro Nashorn. Kdo čeká jen shellové utility, mohl by ho snadno přeskočit. Jenže `jjs` je interpreter s přímým přístupem k Java třídám. To znamená, že SUID `jjs` může:

- otevřít soubor přes `java.io`,
- číst jeho obsah,
- a vrátit ho běžnému uživateli.

```bash
echo 'var BufferedReader = Java.type("java.io.BufferedReader");
var FileReader = Java.type("java.io.FileReader");
var br = new BufferedReader(new FileReader("/root/root.txt"));
while ((line = br.readLine()) != null) { print(line); }' | jjs
```

To je přesně důvod, proč se SUID audit nesmí omezit na známé binárky z cheat sheetu. Interpreter s bohatým standardním API je často stejně silný jako shell.

### Rope: SUID `bash` a zachování efektivního UID

[Rope](/rope) ukazuje nejjednodušší, ale zároveň nejdůležitější variantu: shell s omylem ponechaným SUID bitem. V takové situaci není co "exploitovat". Stačí použít správný přepínač a shell si ponechá efektivní UID roota.

Technická náročnost předešlé části útoku zde není podstatná. Podstatné je, že jediný špatný permission bit na `/bin/bash` zrušil všechnu další obranu hostu.

[Wall](/wall) přidává další klasickou, ale pořád podceňovanou variantu: starý SUID `screen-4.5.0`. Na první pohled nejde o shell ani interpreter, ale historicky jde o velmi známý privesc kandidát přes `ld.so.preload`. Je to dobrá připomínka, že do stejné kategorie nepatří jen „moderní GTFOBins triky“, ale i staré distribuční binárky, které už mají dávno zmapovaný lokální exploit chain.

## Praktický postup po nalezení SUID binárky

### 1. Nejdřív priorizovat, ne zkoušet všechno

Po výpisu SUID souborů má smysl seřadit kandidáty podle chování:

- shell a interpretry,
- service management,
- nástroje pracující se soubory,
- síťové utility,
- vlastní helpery a wrappery.

Právě tahle priorizace bývá cennější než slepé opisování všech GTFOBins postupů.

### 2. Přečíst `--help`, man page a distribuční kontext

U netypické binárky je nejrychlejší zjistit:

- co umí načíst,
- co umí spustit,
- jestli má pluginy nebo hooky,
- a zda je běžné, že pracuje s privilegovaným stavem systému.

`systemctl`, `jjs` i `bash` jsou odlišné nástroje, ale všechny už z dokumentace prozrazují schopnosti, které se se SUID bitem neslučují.

### 3. Nepřehlédnout read-only dopad

Někdy se po hledání root shellu snadno přehlédne, že binárka "jen" čte citlivé soubory. Obranně je to ale skoro stejný problém. Rootův SSH klíč nebo konfigurační tajemství mají často vyšší hodnotu než jednorázový interaktivní shell.

### 4. Vlastní binárky auditovat stejně přísně jako známé utility

Jakmile je SUID na interním helperu nebo méně známé utility, neznamená to menší riziko. Znamená to jen, že místo GTFOBins je potřeba krátká reverzní analýza nebo aspoň čtení dokumentace a argumentů.

## Obrana a bezpečnější návrh

### 1. Pravidelně baselineovat SUID seznam

SUID soubory mají být malá, dobře odůvodněná množina. Jakmile se v ní objeví shell, interpreter nebo service manager, je to silný signál k okamžité revizi.

### 2. Nepoužívat SUID tam, kde stačí capabilities nebo úzké `sudo`

Pokud program potřebuje jen malý kousek privilegované funkce, je bezpečnější řešit to přes capabilities, socket activation nebo úzký wrapper než přes plný SUID root.

### 3. Odebrat SUID z obecných interpreterů a shellů

Shell, JavaScript runtime, Python nebo podobné nástroje nemají mít SUID bit. Jejich samotný účel je dostatečně obecný na to, že s efektivním UID roota z nich vzniká univerzální eskalační primitivum.

### 4. Auditovat distribucí dodané i lokálně přidané utility

Chyba nemusí být jen v custom helperu. Jarvis i Mango ukazují, že problém může vzniknout i na běžných součástech systému, pokud jim někdo omylem změní práva nebo způsob nasazení.

### 5. Sledovat neobvyklé používání SUID binárek

Spouštění `systemctl`, `jjs` nebo `bash -p` z nečekaného kontextu je silný detekční signál. Praktická obrana stojí nejen na odstranění SUID bitu, ale i na monitoringu toho, co se s podobnými binárkami děje.

## Shrnutí klíčových poznatků

- U SUID auditu nerozhoduje popularita binárky, ale její chování.
- `systemctl`, `jjs` a `bash` jsou odlišné nástroje, ale všechny se při SUID bitu mění v cestu k rootovi nebo k root-only datům.
- GTFOBins je užitečný index, ne úplná metodika. Pro skutečné posouzení rizika je potřeba ptát se, zda binárka umí spouštět procesy, číst citlivé soubory nebo zpracovávat cizí konfiguraci.

## Co si odnést do praxe

- Po výpisu SUID souborů nehledejte jen známé "hity". Hledejte hlavně interpretry, shelly, service managery a utility s přístupem k filesystemu.
- Root shell není jediný relevantní dopad. Pokud SUID binárka umí číst `/root`, SSH klíče nebo jiné tajemství, je kompromitace hostu prakticky stejně vážná.
- Obranně je správná otázka jednoduchá: proč má tahle konkrétní binárka vůbec potřebovat SUID root? Když na ni neexistuje velmi přesvědčivá odpověď, pravděpodobně tam ten bit nemá co dělat.
