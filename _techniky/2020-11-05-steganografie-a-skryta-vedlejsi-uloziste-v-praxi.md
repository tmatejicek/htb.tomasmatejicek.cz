---
layout: post
author: Tomáš Matějíček
title: "Steganografie a skrytá vedlejší úložiště v praxi"
date: 2020-11-05
tags: steganography ads hidden-storage artifacts credentials
---
## Úvod a kontext

Ne každé tajemství leží v `config.php`, `.env` nebo `authorized_keys`. V některých prostředích se informace schovávají „nenápadněji“:

- do obrázku,
- do alternativního datového proudu,
- do vedlejšího metadata souboru,
- nebo do jiného úložiště, které běžný průzkum snadno přehlédne.

Irked a BigHead ukazují dvě užitečné varianty téhož problému. Na Irked je tajemství ukryté přímo ve steganografickém obsahu obrázku. Na BigHead neleží v obrázku samotném, ale v alternativním datovém proudu `Zone.Identifier`, který dovede útočníka ke KeePass databázi a key file.

Tyto případy nejsou hlavní proud většiny průniků. Právě proto ale bývají snadno podceněné. Kdo hledá jen plaintext hesla, podobný řetězec prostě přehlédne.

## O čem tenhle článek je a o čem není

Tenhle text není obecný úvod do steganografie jako disciplíny. Neřeší obrazové algoritmy, LSB ani forenzní klasifikaci.

Je o užším a praktičtějším tématu:

- kdy má smysl uvažovat, že tajemství není v hlavním souboru,
- jaké vedlejší stopy na to ukazují,
- a proč „schované“ neznamená „chráněné“.

Jinými slovy: řešíme stego a skrytá vedlejší úložiště jako provozní anti-pattern, ne jako akademickou disciplínu.

## Dvě hlavní rodiny problému

### 1. Tajemství schované v nosiči

To je klasický případ steganografie:

- obrázek,
- audio,
- nebo jiný soubor,

který nese další skrytý obsah.

### 2. Tajemství schované vedle souboru nebo v jeho boční vrstvě

Sem patří:

- alternativní datové proudy,
- metadata,
- vedlejší konfigurační soubory,
- a různé sidecar artefakty, které samy o sobě vypadají bezvýznamně.

Irked reprezentuje první rodinu. BigHead druhou. Obranně je ale pointa stejná: někdo považoval skrytí za dostatečnou ochranu.

## Irked: stego jako mezikrok k SSH

Irked je přímočarý a výukově čistý případ. Po prvním footholdu ležel v `/home` soubor `.backup`, který říkal:

```text
Super elite steg backup pw
UPupDOWNdownLRlrBAbaSSss
```

To je přesně typ stopy, který se nevyplatí ignorovat. Sám o sobě ještě nic neotevírá, ale říká dvě důležité věci:

- někde existuje steganografický nosič,
- a heslo k němu už leží lokálně.

Na webu byl obrázek `irked.jpg`, takže další krok byl racionální:

```text
steghide extract -p UPupDOWNdownLRlrBAbaSSss -sf irked.jpg
=> Kab6h+m+bbp2J:HG
```

Výsledný řetězec fungoval jako SSH heslo pro `djmardov`.

To je hlavní lekce Irked: stego tu není trik pro efekt. Je to jen jiná forma credential storage, která se po nalezení nápovědy redukuje na další tajemství v plaintextu.

## BigHead: ADS a boční úložiště jako cesta k finálnímu tajemství

BigHead ukazuje jiný, ale velmi praktický model. `root.txt` samo o sobě nebylo přímo čitelné tak, jak by člověk čekal. Rozhodující byla až vedlejší vrstva:

```text
root.txt:Zone.Identifier
```

Alternativní datový proud neobsahoval finální heslo, ale navedl na další významný artefakt:

- KeePass databázi,
- a později i vazbu na key file `admin.png`.

To je důležité pochopit správně. Na BigHead není pointa v tom, že ADS samo o sobě „ukrývá root“. Pointa je v tom, že vedlejší úložiště drží právě tu kontextovou informaci, bez které by hlavní tajemství nedávalo smysl.

Právě taková boční vrstva bývá po kompromitaci často rozhodující:

- asociace mezi databází a klíčem,
- nápověda ke správnému souboru,
- nebo metadata, která řeknou, jak má být artefakt použitý.

## Co mají Irked a BigHead společné

Na technické úrovni jde o odlišné věci:

- stego v obrázku,
- alternativní datový proud ve Windows,
- key file v obrázku,

ale mentální model je stejný:

- tajemství není v hlavním souboru tam, kde ho čekáš,
- existuje pomocná stopa, která na skryté místo navede,
- a po nalezení celé skládanky se z „nenápadného artefaktu“ stává normální credential.

To je důvod, proč podobné techniky nepodceňovat. Nejde o kryptografickou ochranu. Jde jen o concealment.

## Kdy má smysl podobné věci hledat

Prakticky dávají smysl tehdy, když narazíš na některý z těchto signálů:

### 1. Explicitní nápověda

`backup pw`, `steg`, `hidden`, `secret image`, zvláštní název souboru nebo poznámka v domácím adresáři. To je nejčistší případ Irked.

### 2. Podezřelé obrázky nebo média v neobvyklém kontextu

Obrázek v admin profilu, key file v PNG, „jen dekorativní“ soubor u tajemství nebo vaultu. BigHead to ukazuje dobře.

### 3. Vedlejší datové proudy a metadata

Na Windows je dobré myslet na:

- ADS,
- `Zone.Identifier`,
- metadata dokumentů,
- a další vrstvy, které běžné `dir` nebo prostý copy nemusí zvýraznit.

### 4. Artefakt, který sám o sobě nedává smysl

Databáze bez klíče, klíč bez databáze, obrázek bez zjevného účelu, nápověda bez cíle. Tyto věci často dávají význam až v kombinaci.

## Rozdíl proti článku o klientských a ne-textových artefaktech

Je užitečné tenhle článek držet odděleně od textu o desktopových a ne-textových artefaktech po footholdu.

Tam jde o širší rodinu:

- KeePass,
- TeamViewer,
- mailboxy,
- CliXml,
- reverzibilní databáze.

Tady je těžiště užší:

- tajemství schované v nosiči,
- alternativním proudu,
- nebo vedlejší vrstvě, kterou člověk při běžném průzkumu snadno mine.

## Obrana a hardening

### 1. Neschovávat tajemství do obrázků nebo „nenápadných“ souborů

Jakmile existuje nápověda nebo přístup k nosiči, stego přestává být ochranou a stává se jen odloženým plaintextem.

### 2. Auditovat vedlejší úložiště a metadata

Bezpečnostní review nesmí končit u hlavního obsahu souboru. Je potřeba kontrolovat i:

- ADS,
- metadata,
- sidecar konfigurace,
- a vazby mezi hlavním artefaktem a jeho pomocnými soubory.

### 3. Neukládat databázi i key file na stejný host

BigHead dobře ukazuje, že i silnější ochranný model ztrácí smysl, pokud všechny jeho části leží na jednom kompromitovatelném systému.

### 4. Nepovažovat concealment za hardening

To, že tajemství není hned vidět v `cat`, neznamená, že je bezpečně chráněné. Pokud ho lze po pár mezikrocích normálně použít, jde stále o běžný credential leak.

### 5. Všímat si neobvyklých artefaktů v domácích adresářích a admin profilech

Právě tam se podobné „chytré“ skrývání objevuje nejčastěji: obrázky, poznámky, staré zálohy, key file a pomocné metadata.

## Shrnutí klíčových poznatků

- Steganografie a skrytá vedlejší úložiště nejsou silná ochrana, ale často jen méně nápadná forma uložení tajemství.
- Irked ukazuje klasický případ: lokální nápověda -> stego heslo -> skrytý obsah v obrázku -> SSH credential.
- BigHead ukazuje druhou variantu: ADS a vedlejší metadata jako vodítko k KeePass databázi a key file.
- Obranně je nejdůležitější nepřehlížet boční vrstvy souborů a nepovažovat „skryté“ za „bezpečné“.

## Co si odnést do praxe

- Když narazíš na nápovědu typu `steg`, `backup pw` nebo na neobvyklý obrázek u citlivých souborů, ber to jako relevantní útokovou stopu, ne jako kuriozitu.
- Na Windows i Linuxu hledej nejen hlavní obsah souborů, ale i to, co leží vedle: metadata, ADS, sidecar configy a vazby na key file.
- Skrytí tajemství do obrázku nebo vedlejší vrstvy nevytváří ochranu. Jen zvyšuje šanci, že si ho někdo všimne až později.
