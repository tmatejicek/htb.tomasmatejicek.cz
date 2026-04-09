---
layout: post
author: Tomáš Matějíček
title: "Document workflow jako RCE nebo file-read primitivum"
date: 2020-11-11
tags: documents file-read rce xxe pdf office
---
## Úvod a kontext

Dokument v aplikaci často nevystupuje jako obyčejný soubor ke stažení. Mnohem častěji spouští celé workflow: import, rozbalení, validaci, konverzi, rendrování do PDF, náhled nebo další publikaci. Právě v této chvíli se z "pasivního" vstupu stává aktivní útoková plocha. Útočník už nepracuje jen s obsahem dokumentu. Pracuje s tím, co všechno s ním backend udělá.

To je užitečné odlišit od užšího článku o XXE. XML entity jsou jen jedna podmnožina problému. Stejně nebezpečné může být:

- že se `docx` rozbalí jako ZIP a parser věří jeho vnitřní struktuře,
- že HTML nebo rich text končí v server-side PDF rendereru,
- že kancelářský dokument někdo automaticky otevře v LibreOffice,
- nebo že konverzní služba přistupuje k lokálním souborům a interním URL, protože je považuje za "bezpečné".

Smysl článku proto není v seznamu parser bugů. Smysl je v modelu důvěry: co přesně dokument v systému spustí a k čemu všemu dostane během zpracování přístup.

## Co dělá z dokumentu útokovou plochu

Z bezpečnostního pohledu je dokument nebezpečný tehdy, když backend udělá alespoň jednu z těchto věcí:

- interpretuje jeho interní strukturu pomocí složitého parseru,
- rozbaluje ho jako archiv a věří souborům uvnitř,
- rendruje jeho obsah do jiného formátu,
- otevírá ho v plnohodnotné desktopové nebo serverové aplikaci,
- nebo výsledek zpracování znovu publikuje jinam s vyššími právy, než měl útočník na vstupu.

Právě proto je dobré myslet na dokument jako na kontejner pro další instrukce. Někdy doslova, jako u XML nebo maker. Jindy nepřímo, třeba přes HTML, lokální cesty, odkazy na externí zdroje nebo manipulaci s archivní strukturou.

## Čtyři opakující se vzory

### 1. XML import jako file-read

Nejjednodušší varianta je ta, kdy dokumentový nebo tracker workflow přijímá XML a parser bez dostatečných omezení expanduje externí entity. Výsledek nebývá hned shell. Často jde nejprve o čtení lokálních souborů, zdrojáků nebo konfigurací.

Typický vstup vypadá takto:

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE bugreport [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<bugreport>
  <title>test</title>
  <reward>&xxe;</reward>
</bugreport>
```

To je důležitý první krok, protože file-read v dokumentovém workflow často otevře zdroják, heslo do databáze nebo další interní endpoint. Není to "jen leak". Je to velmi často mezikrok k plnému footholdu.

### 2. Archivní formát, který se jen tváří jako dokument

`docx`, `xlsx`, `ods` a řada dalších formátů nejsou monolitický binární soubor. Jsou to kontejnery, obvykle ZIP archivy s vnitřní strukturou XML, konfigurací a pomocných souborů. Jakmile backend bez velké opatrnosti dokument rozbaluje a znovu skládá, otevírá mnohem širší plochu než u prostého "upload -> store".

### 3. Renderer, který vidí víc, než by měl

HTML-to-PDF a podobné převody bývají podceňované. Přitom právě renderer často rozhoduje o tom, zda lze:

- číst `file:///` cesty,
- načítat interní URL,
- dělat SSRF do localhostu,
- nebo převést server-side export na čtecí kanál pro tajemství a klíče.

### 4. Dokument se opravdu otevře

Jakmile workflow dokument předá LibreOffice, WinRARu, preview handleru nebo jiné plnohodnotné aplikaci, už nejde jen o parsování dat. Jde o vykonání celé aplikační logiky nad nedůvěryhodným vstupem. To je kvalitativně jiný problém než prosté "zkontrolujeme MIME type a uložíme soubor".

## BountyHunter: XML tracker, který se změnil ve čtení souborů

[BountyHunter](/bountyhunter) je dobrý minimální příklad. Webový tracker nepůsobí jako dokumentové workflow v kancelářském smyslu, ale logika je stejná: aplikace přijme strukturovaný vstup, pošle ho do XML parseru a důvěřuje výsledku. Endpoint `tracker_diRbPr00f314.php` načítá XML s povolenými externími entitami:

```php
libxml_disable_entity_loader(false);
$dom = new DOMDocument();
$dom->loadXML($xml, LIBXML_NOENT | LIBXML_DTDLOAD);
```

To je přesně okamžik, kdy se z ticket formuláře stane file-read primitivum. Útočník si nejprve přečte samotný tracker skript, potom `db.php` a z něj vytáhne heslo `m19RoAU0hP41A1sTsq6K`. Dokumentový vstup tedy nevede rovnou k RCE. Vede k přečtení souborů, které pak otevřou SSH účet `development`.

Na BountyHunter je dobře vidět důležitá lekce: i "malé" importní workflow může být ve skutečnosti velmi silná bezpečnostní hranice, protože přemosťuje nedůvěryhodný vstup k lokálnímu filesystemu.

## Patents: když `docx` znamená XML, ZIP a převodník v jednom

[Patents](/patents) je bohatší a proto výukově cennější případ. Veřejné `release notes`, `.DS_Store` a `installed.json` prozradí, že web nepřijímá dokumenty jen k uložení, ale předává je do `convert.php` a backendu `gears/pdf`. Důležité je už tohle zjištění: útoková plocha neleží v samotném formuláři, ale v konverzní vrstvě.

Release notes navíc přímo naznačí dvě věci:

- zapnuté entity parsing,
- experimentální include funkcionalitu při převodu dokumentů.

To okamžitě mění způsob přemýšlení. Nejde už o obyčejný upload. Jde o parser a konvertor, který:

- rozbalí kancelářský dokument,
- pracuje s jeho vnitřním XML,
- a zřejmě z něj generuje nový výstup.

První rozumný test je proto XXE uvnitř `word/document.xml` zabaleného zpět do `docx`. Tím se potvrdí, že převodník skutečně expanduje entity a umí číst lokální soubory. To je ale pořád jen první půlka příběhu. Jakmile je jasné, že backend důvěřuje ZIP struktuře dokumentu a bez izolace ho převádí, vzniká prostor i pro silnější útok proti samotnému `convert.php`, který už vede k prvnímu shellu.

Patents tak velmi dobře ukazuje, že dokumentové workflow bývá vícevrstvé:

- dokument je archiv,
- uvnitř je XML,
- to všechno zpracuje konvertor,
- a výsledek se vrací do webové vrstvy.

Každá z těchto vrstev může být sama o sobě primitivem pro file-read nebo RCE.

## Bucket: HTML-to-PDF jako server-side čtečka souborů

Na [Bucketu](/bucket) je vidět jiná, ale stejně praktická varianta. `bucket-app` si bere HTML uložené v DynamoDB tabulce `alerts`, zapisuje ho do souboru a následně ho předává Java rendereru `pd4ml_demo.jar`:

```php
file_put_contents('files/'.$name,$item["data"]);
passthru("java -Xmx512m -Djava.awt.headless=true -cp pd4ml_demo.jar Pd4Cmd file:///var/www/bucket-app/files/$name 800 A4 -out files/result.pdf");
```

To je učebnicový moment, kdy se server-side export mění v bezpečnostní hranici. Renderer nepracuje s "bezpečným textem". Pracuje s HTML, které může obsahovat další odkazy a vnořené zdroje. Jakmile mu aplikace dovolí vidět lokální filesystem, jednoduchý vstup:

```html
<html>
  <body>
    <iframe src="/root/.ssh/id_rsa"></iframe>
  </body>
</html>
```

změní PDF export v čtečku citlivých lokálních souborů. A protože výsledkem je PDF, které si lze stáhnout, file-read se okamžitě mění v praktickou krádež `root` SSH klíče.

Bucket připomíná, že dokumentové workflow neznamená jen Word a XML. Stejně nebezpečný může být i renderer, který dostane HTML a běží s příliš širokým přístupem k lokálním cestám.

## RE: dokument jako vstup do celé analýzové pipeline

[RE](/re) jde ještě o krok dál. `malware_dropbox` není klasický upload formulář, ale vstup do interní pipeline, která:

- soubor přesune do pracovního adresáře,
- přejmenuje `.ods` na `.zip`,
- archiv rozbalí,
- zkontroluje pomocí Yara,
- a pokud projde, otevře ho v LibreOffice.

Tím se krásně ukazuje hranice mezi dokumentovým workflow a širší bezpečnostní pipeline. Z pohledu tohoto článku je důležitý hlavně moment, kdy se kancelářský dokument opravdu otevře v LibreOffice. V tu chvíli už nejde o "parsování obsahu". Jde o to, že nedůvěryhodný dokument vstupuje do plnohodnotné aplikace, která umí makra, externí zdroje a další vedlejší efekty.

Právě proto mohl `.ods` s makrem po nahrání do `malware_dropbox` otevřít shell jako `luke`. Dokument zde neprošel jedním parserem. Prošel celým workflow, které mu nakonec dalo interaktivní execution context.

## Jak podobná workflow hledat a číst

Při auditu aplikace nebo hostu má smysl klást si několik velmi konkrétních otázek.

### Co přesně se se souborem děje po uploadu nebo importu

- Uloží se jen na disk?
- Rozbalí se jako archiv?
- Předá se do XML parseru?
- Vyrenderuje se do PDF nebo náhledu?
- Otevře se v LibreOffice, Wordu, WinRARu nebo jiné aplikaci?

### Jaký přístup má zpracovávající komponenta

- vidí lokální filesystem,
- umí číst `file://` nebo UNC cesty,
- umí sahat na localhost,
- běží pod privilegovaným uživatelem,
- nebo má přístup k interním tajemstvím a konfiguračním souborům.

### Kdo dostane výsledek zpracování

- běžný uživatel jako download,
- interní admin rozhraní,
- jiný proces v pipeline,
- nebo další servisní komponenta.

Právě poslední bod je důležitý. File-read nebo render obvykle není cíl sám o sobě. Cíl je, kam se výstup dál dostane a co z něj útočník umí vyčíst.

## Obrana a hardening

### 1. Převodníky a preview služby izolovat

Konverzní backend nemá běžet se zbytečným přístupem k lokálním tajemstvím, domovským adresářům, SSH klíčům ani interním servisním socketům. Ideální je samostatný sandbox nebo kontejner s minimálním přístupem.

### 2. XML a archivní formáty brát jako aktivní vstup

`docx` není "jen dokument". Je to ZIP s XML. Stejně tak HTML export není "jen text". Je to vstup do rendereru. Obrana proto musí hlídat parser i kontext, v němž běží.

### 3. Zakázat lokální a interní načítání, pokud není nutné

Renderer nebo preview komponenta by neměla:

- načítat `file:///`,
- chodit na localhost,
- sahat na interní URL,
- nebo bezdůvodně otevírat externí zdroje obsažené v dokumentu.

### 4. Neotevírat nedůvěryhodné dokumenty v plné desktopové aplikaci bez izolace

Jakmile workflow končí v LibreOffice nebo jiné bohaté aplikaci, je potřeba to považovat za execution boundary a podle toho volit sandboxing, omezení maker a oddělení od zbytku prostředí.

### 5. Auditovat celý řetězec, ne jen upload endpoint

Skutečný problém často neleží v `upload.php`. Leží v `convert.php`, preview workeru, pomocném JAR souboru nebo v tom, co se stane až po uložení do databáze a následném renderu.

## Shrnutí klíčových poznatků

- Dokumentový workflow je bezpečnostní hranice ve chvíli, kdy backend dokument rozbaluje, parsuje, rendruje nebo automaticky otevírá.
- BountyHunter ukazuje XML import jako file-read primitivum, Patents vícevrstvý problém kolem `docx` a konverze, Bucket server-side PDF renderer a RE automatické otevření kancelářského souboru.
- Největší riziko nevzniká jen z parseru samotného, ale z toho, k čemu má zpracovávající komponenta během workflow přístup.
- Obrana stojí hlavně na izolaci převodníků, zákazu zbytečných lokálních zdrojů a na tom, že dokument není braný jako pasivní soubor, ale jako aktivní vstup do dalších komponent.

## Co si odnést do praxe

- Při auditu se neptej jen "lze soubor nahrát?". Ptej se hlavně "co se s ním stane potom".
- Každý PDF export, Office preview nebo XML import je potenciální file-read nebo RCE hranice, pokud běží bez izolace a s přístupem k místům, která útočník sám nevidí.
- Jakmile se dokument automaticky otevírá nebo převádí server-side, je potřeba jeho zpracování navrhovat stejně opatrně jako spouštění cizího kódu.
