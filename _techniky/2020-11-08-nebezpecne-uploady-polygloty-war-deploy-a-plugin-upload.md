---
layout: post
author: Tomáš Matějíček
title: "Nebezpečné uploady: polygloty, WAR deploy a plugin upload"
date: 2020-11-08
tags: upload web cms tomcat files
---
## Úvod a kontext

Upload je jedna z nejzrádnějších funkcí v celé webové bezpečnosti. Na první pohled vypadá jednoduše: uživatel pošle soubor, aplikace ho uloží a hotovo. Ve skutečnosti ale upload skoro nikdy nekončí uložením. Soubor se někde pojmenuje, přesune, zobrazí, rozbalí, přečte plugin loader, zpracuje aplikační server nebo obslouží CMS administrace. Právě v tom se skrývá skutečné riziko.

Proto nedává smysl mluvit o uploadu jen jako o "nahrání shellu". Nebezpečný upload má několik odlišných podob: filtrační bypass přes polyglot nebo dvojitou příponu, legitimní deployment rozhraní typu WAR upload v Tomcatu a plugin nebo theme mechanizmy, které samy slouží k nahrání a spuštění kódu. Všechny vedou k RCE, ale každá jiným způsobem a vyžaduje jinou obranu.

## Co přesně je na uploadu nebezpečné

Upload nebývá problém ve chvíli, kdy server přijme bajty. Problém vzniká ve chvíli, kdy aplikace:

- špatně rozhodne, co je to za soubor,
- uloží ho do vykonatelného prostoru,
- nechá ho interpretovat jinou komponentou,
- nebo dá administrátorovi možnost nahrát něco, co je z principu spustitelné.

Jinými slovy: riziko uploadu je vždy kombinací tří otázek:

1. Co server přijme?
2. Kam to uloží?
3. Kdo a jak to později zpracuje?

## Varianta 1: polyglot a obcházení validační logiky

Nejznámější upload pattern je ten, kdy aplikace věří koncovce, MIME typu nebo povrchní kontrole obsahu, ale uloží soubor do místa, kde se může vykonat.

Typický scénář:

- aplikace čeká obrázek,
- útočník dodá soubor, který projde jako JPG nebo PNG,
- ale zároveň obsahuje interpretovatelný payload,
- a server ho uloží do webrootu nebo jiného vykonatelného prostoru.

To může vypadat třeba jako:

- dvojitá přípona,
- polyglot obrázek,
- komentář nebo metadata s vloženým kódem,
- nebo soubor, který je pro validátor "obrázek", ale pro jinou vrstvu už skript.

V takovém řetězci nebývá největší chyba v tom, že validace není perfektní. Největší chyba je v tom, že se uživatelský upload ukládá tam, kde jeho pozdější interpretace znamená spuštění kódu.

## Varianta 2: upload jako legitimní deploy rozhraní

Jiná situace nastává tam, kde nahrání souboru není bezpečnostní omyl, ale zamýšlená funkce. Typickým příkladem je aplikační server nebo administrační rozhraní, které dovoluje deploy aplikace.

Tomcat Manager je učebnicový případ:

- uživatel s dostatečnými právy nahraje `WAR`,
- server ho rozbalí a nasadí,
- výsledkem je spuštěná aplikace.

To technicky není "bypass upload filtru". Je to správcovská funkce, která sama představuje deployment kanál. Bezpečnostní problém vzniká ve chvíli, kdy:

- je Manager veřejně dostupný,
- chrání ho slabé nebo výchozí heslo,
- nebo Tomcat běží s příliš vysokými právy.

Upload tedy není zranitelnost v klasickém smyslu. Je to přímý RCE endpoint, který byl špatně zpřístupněný.

## Varianta 3: plugin, theme nebo modul jako oficiální cesta ke kódu

CMS a blogovací systémy často dovolují do administrace nahrát:

- plugin,
- theme,
- obrázek, který se pak dá dál zneužít,
- nebo jiný rozšiřující balíček.

Tady je zásadní rozdíl oproti polyglotu. Útočník neobchází ochranu tím, že soubor vydává za něco jiného. On využívá fakt, že administrace sama nabízí cestu, jak dostat na server kód nebo soubor, který se později vykoná.

Prakticky se to láme ve dvou bodech:

- jak obtížné je získat admin přístup,
- a jak striktně CMS odděluje upload obsahu od instalace kódu.

Jakmile se tyto dvě věci potkají špatně, stává se z admin uploadu nejkratší cesta k shellu.

## Jak tenhle vzorec vypadal v různých případech

Na jednom hostu upload přijímal obrázek dostatečně povrchně na to, aby prošel soubor s vloženým PHP payloadem a dvojitou příponou. Rozhodující nebyla "kreativita payloadu", ale fakt, že server nahraný soubor obsloužil z vykonatelného webového prostoru.

Jinde byl celý řetězec ještě přímočařejší: veřejně dostupný Tomcat Manager chránilo slabé heslo a samotná administrace umožnila nahrát `WAR`. Tam už nešlo o obcházení filtru. Šlo o to, že deployment rozhraní bylo fakticky otevřeným RCE kanálem.

U další dvojice případů byl problém v tom, že blogovací nebo CMS administrace sama dovolovala nahrát soubor, plugin nebo obrázek způsobem, který vedl ke spuštění kódu. Jakmile se podařilo získat přístup do adminu, upload už nebyl omezením. Byl právě tou oficiální funkcí, přes kterou se na host dostal payload.

## Proč je chyba často jinde než v validaci

Upload review se často smrskne na otázku:

- kontroluje se MIME,
- kontroluje se koncovka,
- kontroluje se magic bytes.

To je důležité, ale samo o sobě nestačí. Ve skutečnosti jsou stejně podstatné tyto otázky:

- ukládá se upload mimo webroot,
- mění se při uložení název a koncovka,
- dochází k bezpečné rekonverzi obsahu,
- nebo soubor zůstává v podobě, v jaké ho poslal uživatel,
- čte ho jen pasivní storage vrstva,
- nebo ho později interpretuje PHP, Tomcat, CMS nebo plugin loader.

Právě tady se rozhoduje, jestli chyba skončí jako neškodný obsahový problém, nebo jako plnohodnotné RCE.

## Co při analýze ověřovat

### Kam se upload ukládá

- přímo do vykonatelného webrootu,
- do objektového úložiště,
- do dočasného adresáře,
- nebo do prostoru, který později čte jiný proces.

### Kdo upload později interpretuje

- Apache nebo nginx jako statický obsah,
- PHP interpreter,
- aplikační server jako Tomcat,
- CMS plugin mechanismus,
- nebo konverzní a náhledová služba.

### Jak silná je skutečná kontrola typu

- jen koncovka,
- jen `Content-Type`,
- magic bytes,
- whitelist povolených struktur,
- nebo plná rekonverze například do nového obrázku.

### Kdo smí nahrávat

- anonymní uživatel,
- běžný uživatel,
- administrátor,
- nebo servisní workflow.

U deploy a plugin cest je tohle klíčové. I "autentizovaný upload" je často jen jinak pojmenované RCE, pokud je admin účet dosažitelný nebo špatně chráněný.

## Obrana a bezpečnější návrh

### 1. Upload ukládat mimo vykonatelný prostor

Pokud nahraný soubor není určený k vykonání, nemá ležet tam, kde ho může interpretovat PHP, JSP nebo jiná runtime vrstva.

### 2. Rekonvertovat, ne jen kontrolovat

U obrázků a podobných formátů je bezpečnější obsah znovu vyrenderovat nebo převést do bezpečného výstupu, než spoléhat na to, že validátor rozpozná každý škodlivý trik.

### 3. Přísně oddělit upload obsahu od deploymentu kódu

Plugin upload, WAR deploy nebo instalace modulu jsou z principu vysoce citlivé funkce. Nemají být dostupné na veřejném rozhraní bez silné autentizace, víceúrovňového schválení a síťového omezení.

### 4. CMS administraci chápat jako code-execution hranici

Jestli administrace dovoluje nahrát plugin, theme nebo skript, nejde o obyčejný backoffice. Jde o rozhraní, které samo rozhoduje o spuštění kódu na serveru.

### 5. Minimalizovat práva procesu, který uploady obsluhuje

I při kompromitaci uploadu má rozhodovat, s jakými právy běží cílový proces. Pokud webserver nebo aplikační server běží příliš vysoko, zkracuje se celá cesta k plnému kompromisu.

## Shrnutí klíčových poznatků

- Nebezpečný upload není jedna chyba, ale rodina problémů od polyglot bypassu přes WAR deploy až po plugin a CMS mechanismy.
- Rozhodující není jen to, co server přijme, ale hlavně kam soubor uloží a kdo ho pak interpretuje.
- Authenticated upload v admin rozhraní není nutně "méně závažný". Pokud administrace sama umožňuje nasadit kód, jde prakticky o vestavěný RCE kanál.
- Obrana stojí na oddělení uploadu obsahu od spustitelného prostředí a na tom, že deployment funkce nejsou zaměňované za běžný upload.

## Co si odnést do praxe

- Při auditu uploadu vždy ověřujte tři věci: typ kontroly, umístění uloženého souboru a komponentu, která ho bude později interpretovat.
- Polyglot obrázek je jen jedna varianta problému. Stejně nebezpečné jsou i legitimní deploy a plugin mechanizmy, pokud se k nim dostane útočník.
- Když administrace umožňuje nahrát kód, nejde už o pohodlnou funkci pro správce. Je to privilegovaný execution endpoint a podle toho se musí chránit.
