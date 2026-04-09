---
layout: post
author: Tomáš Matějíček
title: "XXE a XML workflow"
date: 2020-11-11
tags: xxe xml web parser
---
## Úvod a kontext

XXE se často vysvětluje jako chyba XML parseru, který dovolí načítat externí entity. To je technicky správně, ale pro praxi trochu málo. V běžném provozu totiž XML málokdy vstupuje do systému jako "libovolný XML dokument". Obvykle je schované ve formuláři, trackeru, importu, dokumentovém workflow nebo převodníku, který vývojář nepovažuje za exponovanou attack surface.

Právě proto dává smysl mluvit o XML workflow, ne jen o parseru. Útočník totiž obvykle neútočí na XML jako takové. Útočí na podnikový nebo aplikační proces, který XML přijímá, rozbaluje, převádí nebo jinak zpracovává s větší důvěrou, než si zaslouží. U širších dokumentových řetězců na to navazuje i článek [Document workflow jako RCE nebo file-read primitivum](/techniky/document-workflow-jako-rce-nebo-file-read-primitivum).

## Proč je XML pořád relevantní

Mnoho týmů má pocit, že XXE patří spíš do starších technologií. Jenže XML se stále objevuje v celé řadě míst:

- interní trackery a importní endpointy,
- kancelářské formáty typu `docx`, které jsou ve skutečnosti ZIP s XML soubory,
- převodníky dokumentů,
- integrační rozhraní mezi aplikacemi,
- konfigurační a exportní formáty.

To znamená jednoduchou věc: i když veřejné API působí moderně, uvnitř může pořád běžet parser, který umí:

- číst lokální soubory,
- navazovat HTTP požadavky,
- nebo interpretovat strukturu dokumentu dřív, než se dostane ke slovu jakákoli obchodní logika.

## Kde se XXE v praxi schovává

XXE nevzniká jen tam, kde aplikace explicitně přijme `application/xml`. Často se objeví ve workflow, která na první pohled nevypadají jako XML:

- bugreport nebo tracker formulář, který si backend převádí do XML,
- upload kancelářského dokumentu, který se interně rozbalí a přepíše,
- validační nebo konverzní služba nad dokumenty,
- import/export funkcionalita ve správě.

Právě to je důležité i pro audit. Nestačí se ptát, jestli aplikace "má XML endpoint". Je potřeba se ptát, kde všude nějaká komponenta XML nakonec opravdu parsuje.

## Co XXE ve skutečnosti dovolí

XXE se vyplatí chápat podle dopadu, ne podle syntaxe payloadu.

### 1. File read

Nejběžnější dopad je čtení lokálních souborů. To je stále nejpraktičtější varianta, protože vede ke:

- konfiguračním tajemstvím,
- aplikačním klíčům,
- zdrojovým kódům,
- nebo dalším indiciím k laterálnímu pohybu.

Typický didaktický příklad vypadá takto:

```xml
<?xml version="1.0"?>
<!DOCTYPE bugreport [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<bugreport>
  <title>test</title>
  <reward>&xxe;</reward>
</bugreport>
```

Samotné `/etc/passwd` ale nebývá cílem. Je to jen potvrzení, že parser čte lokální soubory a že dává smysl sáhnout po `db.php`, `.env` nebo jiném provozním artefaktu.

### 2. SSRF do dalších služeb

Pokud parser dovolí externí entity přes `http://`, nevzniká jen file read, ale i SSRF. To bývá důležité hlavně ve chvíli, kdy:

- cílový host vidí interní síť,
- parser běží na jiném serveru než veřejná aplikace,
- nebo dokumentový backend důvěřuje localhost-only službám.

XXE tak nemusí končit u jednoho souboru. Může se stát síťovým pivotem.

### 3. Potvrzení chování dokumentového backendu

V některých řetězcích XXE sama nevydá finální shell, ale potvrdí, že převodník nebo importní backend skutečně:

- rozbaluje dokument,
- expanduje entity,
- a pracuje s nedůvěryhodným obsahem příliš otevřeně.

To je cenné hlavně u složitějších workflow, kde je XXE první důkaz, že dokumentová pipeline stojí na slabých předpokladech a zaslouží si hlubší test.

## Jak tenhle vzorec vypadal v konkrétních případech

Na [BountyHunteru](/bountyhunter) šlo o tracker endpoint, který přijímal base64-encoded XML a přes `DOMDocument` ho načítal s povolenými entitami. Praktický dopad nebyl v tom, že by parser "vrátil `/etc/passwd`". Rozhodující bylo až čtení lokálního `db.php`, z něhož vypadlo heslo použitelné pro SSH účet `development`. XXE tedy nevedla přímo k RCE, ale otevřela nejkratší cestu k tajemství, které už shell zajistilo.

Na [Patents](/patents) stál jiný případ na dokumentovém převodníku a veřejně dostupných artefaktech, z nichž šlo odvodit, že backend pracuje s kancelářským dokumentem jako se ZIP strukturou obsahující XML. Škodlivý `docx` s externí entitou tam posloužil jako potvrzení, že převodník opravdu expanduje entity a že dokumentové workflow stojí na nebezpečném parseru. Samotný shell pak přišel jinou cestou nad stejným převodníkem, ale XXE byla klíčová pro správné čtení architektury celé funkce.

## Co při analýze hledat

Při review nebo testu se vyplatí projít několik konkrétních otázek.

### Kde se XML do systému dostává

- přijímá ho veřejný formulář,
- převádí si ho aplikace interně sama,
- rozbaluje se z dokumentu,
- nebo ho zpracovává pomocná konverzní služba.

### Jaký parser a jaké volby se používají

- jsou zakázané externí entity,
- je zakázané načítání DTD,
- používá se bezpečný režim parseru,
- nebo je parser spuštěný s historickým defaultem.

### Jaká práva má proces, který XML čte

- vidí lokální konfigurační soubory,
- má přístup k interní síti,
- běží v odděleném sandboxu,
- nebo sdílí filesystem a síť s hlavní aplikací.

### Co parser dělá s výsledkem

- vrací přímo obsah do odpovědi,
- ukládá ho do databáze,
- přeposílá ho dalšímu kroku pipeline,
- nebo jen potichu potvrzuje, že dokumentový backend pracuje s nedůvěryhodným obsahem.

## Proč "zakázat externí entity" není celý příběh

Tohle doporučení je správné, ale samo o sobě nestačí jako mentální model.

První problém je, že XML bývá ukryté ve workflow, které si tým jako XML ani nepojmenuje.  
Druhý problém je, že i správně opravený parser nemusí vyřešit celou dokumentovou pipeline, pokud:

- převodník rozbaluje ZIP strukturu nekontrolovaně,
- backend běží s příliš širokými právy,
- nebo importní workflow dál pracuje s nedůvěryhodným obsahem jiným nebezpečným způsobem.

Jinými slovy: XXE je často první symptom, že hranice kolem dokumentového nebo importního workflow je navržená příliš volně.

## Obrana a bezpečnější návrh

### 1. Zakázat externí entity a DTD tam, kde nejsou nezbytné

To je základ. Parser nemá číst externí entity jen proto, že to "umí". Pokud aplikace nepotřebuje DTD, nemá být zapnuté.

### 2. XML parser chápat jako bezpečnostní hranici

Proces, který zpracovává XML nebo dokumenty obsahující XML, nemá běžet s přístupem k citlivému filesystemu a k interní síti. I když dojde k chybě parseru, dopad má zůstat omezený.

### 3. Auditovat i skrytá XML workflow

Bezpečnostní review musí zahrnout:

- importy,
- trackery,
- převodníky dokumentů,
- kancelářské formáty,
- a pomocné backendy, které si XML vytvářejí nebo rozbalují "bokem".

### 4. Nevracet parser output slepě do odpovědi

Pokud aplikace přímo vypisuje zpracovaný XML obsah do response nebo do dalšího kroku workflow, stává se z parseru mnohem pohodlnější file-read kanál.

### 5. Tajemství uložená vedle aplikace považovat za okamžitě ohrožená

Jakmile je možný XML-based file read, jsou `.env`, `db.php`, `config.yml` a podobné soubory prakticky první cíle. Obrana musí počítat s tím, že parser bug rychle eskaluje na credential leak.

## Shrnutí klíčových poznatků

- XXE není jen parser bug, ale chyba v celém XML workflow, které nedůvěryhodný vstup zpracovává příliš široce.
- V praxi se XML často schovává ve formulářích, trackerech, importech a dokumentových převodnících.
- Nejčastější dopady jsou file read, SSRF a potvrzení, že dokumentový backend stojí na nebezpečných předpokladech.
- Obrana musí řešit jak parser, tak práva a izolaci procesu, který XML čte.

## Co si odnést do praxe

- Když narazíte na XML, neptejte se jen "jde z něj vyčíst `/etc/passwd`". Ptejte se, jaké workflow XML do systému přináší a co parser smí dělat.
- XXE je často nejcennější ve chvíli, kdy vydá provozní tajemství nebo potvrdí chování jinak špatně viditelného dokumentového backendu.
- Bezpečný návrh nestojí jen na parser flagu. Stojí i na tom, že importní a konverzní služby nejsou připojené ke všemu, co by parser mohl zkusit číst nebo kontaktovat.
