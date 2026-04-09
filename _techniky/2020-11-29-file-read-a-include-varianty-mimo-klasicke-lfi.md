---
layout: post
author: Tomáš Matějíček
title: "File read a include varianty mimo klasické LFI"
date: 2020-11-29
tags: lfi web file-read include
---
## Úvod a kontext

Klasické LFI si většina lidí vybaví jako `../../../../etc/passwd`. To je dobrý základ, ale v praxi se file read a include chyby často schovávají v méně nápadných variantách. Aplikace sice zablokuje obyčejný traversal, ale stále přijme `php://filter`, UNC cestu na SMB share nebo vstup, který se po unicode normalizaci změní na zakázanou sekvenci až příliš pozdě.

Ten rozdíl je důležitý i metodicky. Nejde jen o to, jestli "mám traversal". Důležité je, jak aplikace interpretuje zadanou cestu, jakou vrstvu abstrahuje jazyk nebo framework a v jakém pořadí probíhá validace, normalizace a samotné otevření souboru. Pokud vás zajímá klasický základ, navazuje tento text na starší článek o [Local File Inclusion](/techniky/local-file-inclusion).

## Co mají tyto chyby společné

Na první pohled jde o různé techniky, ale všechny stojí na stejném omylu: aplikace si myslí, že kontroluje souborovou cestu, zatímco runtime ve skutečnosti pracuje s jiným významem vstupu.

V praxi bývá problém v jedné z těchto vrstev:

- aplikace validuje textový řetězec, ale runtime akceptuje i stream wrapper,
- aplikace očekává lokální soubor, ale operační systém připustí síťovou cestu,
- filtr kontroluje vstup před normalizací, zatímco otevření souboru proběhne až po ní,
- vývojář se soustředí na `../`, ale přehlédne jiný způsob, jak runtime přimět ke čtení souboru.

Proto dává smysl mluvit spíš o rodině file read a include chyb než jen o jedné LFI zkratce.

## Varianta 1: stream wrapper místo klasické cesty

V PHP není vstup do `include`, `file_get_contents()` nebo podobné funkce vždy jen "cesta k souboru". Může jít i o wrapper, tedy speciální schéma, které říká, jak má být zdroj otevřený a zpracovaný.

Typický příklad:

```php
$path = $_POST['url'];
echo file_get_contents($path);
```

Pokud aplikace očekává cestu k obrázku nebo konfiguračnímu souboru, ale nepoužije pevný allowlist, může vstup typu:

```text
php://filter/convert.base64-encode/resource=/etc/passwd
```

změnit význam celé operace. Už nejde o běžné načtení lokálního souboru, ale o wrapper, který obsah nejdřív přečte a pak ho před vrácením zakóduje.

[ForwardSlash](/forwardslash) je přesně ten typ případu, kde se tohle chování neukázalo jako "zajímavá syntaxe", ale jako plnohodnotný foothold mezikrok. `api.php` akceptovalo `php://filter`, takže místo prostého helperu na načtení zdroje vznikl spolehlivý čtecí kanál ke `config.php` i dalším lokálním souborům.

Praktický dopad je dvojí:

- obejde se jednoduchá ochrana, která počítá jen s relativní cestou,
- čtení binárních nebo "nehezkých" souborů je přes base64 mnohem spolehlivější.

To je přesně důvod, proč wrappery patří do stejné mentální kategorie jako LFI, i když na payloadu není ani jedno `../`.

## Varianta 2: síťová cesta, která vypadá jako soubor

Na Windows je další zrádná vlastnost: mnoho funkcí pracujících se soubory akceptuje i UNC cesty ve tvaru `\\\\server\\share\\soubor`.

Pokud aplikace dělá něco jako:

```php
include($_GET['lang'] . '.php');
```

vývojář často myslí na:

- traversal,
- lokální include,
- případně URL wrapper.

Nemusí ale myslet na to, že operační systém a runtime mohou považovat síťový share za "normální souborový zdroj". Na Windows tak může parametr pro jazykový soubor skončit u vzdáleného SMB share a aplikace pak includuje obsah, který neleží na lokálním disku, ale na stroji útočníka.

Tohle už není jen file read. Pokud daný runtime include zároveň vykonává, mění se chyba velmi rychle v RCE bez klasického uploadu.

## Varianta 3: validace před jinou normalizací než používá aplikace

Další častý vzorec je, že filtr kontroluje surový vstup, ale otevření souboru proběhne až po normalizaci, canonicalizaci nebo jiném převodu znaků.

To je dobře vidět u unicode traversal variant. Aplikace zakáže `../`, ale pracuje s řetězcem, který se později normalizuje na stejný význam. Znak, který původně neobsahuje doslova dvě tečky nebo lomítko, se po zpracování změní právě na ně.

Praktický problém pak není v tom, že by filtr "zapomněl" na traversal. Problém je v pořadí operací:

1. aplikace zkontroluje původní tvar vstupu,
2. jiná vrstva ho normalizuje,
3. teprve normalizovaný tvar rozhodne, jaký soubor se otevře.

Výsledek je stejný jako u klasického LFI, ale root cause leží v nesouladu mezi tím, co validuje filtr, a tím, co čte runtime.

## Proč tyto varianty unikají pozornosti

Na podobných chybách je zrádné hlavně to, že ochrana může na první pohled vypadat rozumně:

- blokuje `../`,
- zakazuje `http://`,
- omezuje vstup na "souborovou cestu",
- filtruje mezery nebo speciální znaky.

Jenže útok jde o úroveň níž. Neútočí na přesný text, který si představoval vývojář, ale na interpretaci cesty v jazyce, frameworku nebo operačním systému.

To je důvod, proč nestačí testovat jen několik obvyklých traversal payloadů. U file read chyb je často důležitější pochopit, jaký typ vstupu daná funkce ve skutečnosti akceptuje.

## Jak o tom přemýšlet při analýze

Při review nebo testu je užitečné si položit několik konkrétních otázek.

### Co přesně aplikace očekává

- skutečný soubor z pevného adresáře,
- jazykový soubor,
- cestu k obrázku nebo šabloně,
- URL ke stažení,
- obecný string, který pak runtime sám interpretuje.

### Kdo interpretuje význam vstupu

- samotná aplikace,
- PHP wrappery,
- operační systém,
- knihovna pro normalizaci,
- reverse proxy nebo framework.

### Kdy probíhá validace

- před normalizací,
- po normalizaci,
- nad původním stringem,
- nebo nad canonical path, která se skutečně otevře.

### Co se stane po úspěšném čtení

- jen se vrátí obsah souboru,
- obsah se zakóduje nebo přepošle dál,
- include soubor i vykoná,
- nebo chyba otevře cestu k tajemstvím použitelným v jiné službě.

## Kde se z file read stává větší problém

Samotné čtení souboru ještě nemusí znamenat shell. Prakticky ale velmi často vede k jedné z těchto dalších vrstev:

- konfigurační tajemství použitelné pro SSH nebo WinRM,
- JWT secret nebo aplikační klíč,
- přístup do databáze,
- lokální klíče a certifikáty,
- zdrojové kódy, které odhalí další slabinu,
- remote include nebo execute kontext.

To je přesně důvod, proč je file read tak cenné primitivum. Často nejde o finální exploit, ale o nejspolehlivější způsob, jak z aplikace dostat tajemství a pochopit její vnitřní vazby.

## Obrana a bezpečnější návrh

### 1. Nepřijímat obecnou cestu tam, kde stačí identifikátor

Pokud aplikace přepíná jazyk, šablonu nebo dokument, bezpečnější je přijmout pevný identifikátor:

```php
$allowed = ['cz' => 'lang/cz.php', 'en' => 'lang/en.php'];
include($allowed[$_GET['lang']] ?? 'lang/cz.php');
```

Ne obecnou cestu od uživatele.

### 2. Validovat až po stejné normalizaci, jakou používá runtime

Kontrola nad surovým vstupem nestačí, pokud se jeho význam později změní. Bezpečnostní rozhodnutí se musí dělat nad canonicalizovaným tvarem cesty, ne nad tím, co uživatel poslal doslova.

### 3. Myslet na wrappery a síťové cesty jako na jiné typy vstupu

`php://`, `file://`, UNC share nebo jiný síťový path nejsou "detail syntaxe". Jsou to jiné režimy práce se zdrojem. Pokud je aplikace neumí bezpečně potřebovat, nemá je akceptovat.

### 4. Oddělit file read od execute kontextu

Funkce typu `include` nebo templating helper jsou rizikovější než prosté čtení souboru, protože kombinují dva problémy:

- výběr zdroje,
- a jeho vykonání nebo interpretaci.

Tam, kde je to možné, je bezpečnější pracovat s datovým obsahem, ne s dynamickým include mechanismem.

### 5. Chránit konfigurační tajemství i proti "pouhému" čtení

Jakmile aplikace umí číst své vlastní configy, je často pozdě uvažovat o dopadu jen jako o informačním úniku. V praxi to bývá přímá cesta k dalšímu přihlášení nebo k obejití jiné vrstvy systému.

## Shrnutí klíčových poznatků

- File read a include chyby nejsou omezené na klasické `../` traversal payloady.
- Stejný dopad může vzniknout přes stream wrapper, síťovou cestu nebo chybnou normalizaci vstupu.
- Rozhodující není tvar payloadu, ale to, jak jeho význam interpretuje jazyk, framework a operační systém.
- Bezpečná obrana stojí na pevném allowlistu, canonicalizaci a na tom, že aplikace nepředává uživatelský vstup přímo do obecných file/include funkcí.

## Co si odnést do praxe

- Když aplikace pracuje se soubory, netestujte jen `../../etc/passwd`. Ptejte se, jaké další režimy práce se zdrojem runtime podporuje.
- Wrappery, UNC cesty a unicode normalizace nejsou okrajové speciality. V praxi právě ony často obejdou filtr, který proti klasickému traversal vypadá správně.
- Jakmile file read otevře konfigurační soubory nebo zdrojáky, nejde už jen o čtení. Velmi často je to první krok k dalšímu shellu, tokenu nebo privilegovanému účtu.
