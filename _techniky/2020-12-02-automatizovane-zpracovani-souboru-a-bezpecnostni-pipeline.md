---
layout: post
author: Tomáš Matějíček
title: "Automatizované zpracování souborů a bezpečnostní pipeline"
date: 2020-12-02
tags: files pipeline automation ops security processing
---
## Úvod a kontext

Řada organizací dnes soubory jen neukládá. Hned po přijetí je přesouvá mezi složkami, rozbaluje, kontroluje, analyzuje, převádí nebo z nich vytěžuje další data. Taková pipeline obvykle vznikne z dobrého důvodu: antimalware analýza, PDF export, centrální zpracování logů, interní schvalování nebo monitoring. Bezpečnostní problém ale často neleží v jedné konkrétní zranitelnosti parseru. Leží v orchestrace kolem něj.

RE, Patents a Haystack ukazují tři různé varianty stejného vzorce:

- někde se nedůvěryhodný dokument automaticky otevře v LibreOffice,
- jinde se soubor předá konverznímu backendu a dalším lokálním službám,
- a jinde log processing pipeline přestane být monitoringem a začne vykonávat příkazy.

Smysl článku proto není v tom, jak funguje jeden konkrétní parser. Smysl je v tom, jak bezpečnostně číst celý řetězec: kdo soubor přijme, kdo ho přesune, kdo ho přejmenuje, kdo ho analyzuje, kdo výsledku věří a s jakými právy to všechno běží.

## Proč je pipeline nebezpečnější než jeden parser

Když se mluví o chybách při zpracování souborů, pozornost obvykle padne na:

- XML parser,
- archivátor,
- Office aplikaci,
- PDF renderer,
- nebo antivirový engine.

To je ale jen část obrazu. Praktický dopad často vznikne až spojením několika kroků:

1. útočník dodá soubor do vstupního prostoru,
2. systém ho automaticky přesune nebo přejmenuje,
3. další komponenta ho rozbalí nebo otevře,
4. výstup zpracování se dostane do citlivější vrstvy,
5. a někdo předpokládá, že už jde o "interní" nebo "ověřený" artefakt.

Právě mezi těmito kroky mizí původní nedůvěra. To je bezpečnostní jádro celé třídy problémů.

## Pět opakujících se slabin v pipeline

### 1. Vstupní prostor je dostupnější, než by měl být

Sdílená složka, upload adresář nebo lokální konzole bývají považované za "jen vstup". Ve skutečnosti je to začátek privilegovaného workflow.

### 2. Pipeline mění význam souboru za běhu

Přejmenování `.ods` na `.zip`, opětovné zabalení do `.rar`, převod HTML do PDF nebo vygenerování JS konfigurace mění nejen formát. Mění i to, který nástroj další krok použije a jaké má bezpečnostní vlastnosti.

### 3. Ověření je slabší než zbytek workflow

Yara pravidla, MIME kontrola nebo základní filtr názvů často působí jako ochrana. Pokud ale po jejich obejití pipeline soubor otevře v plné aplikaci nebo pošle do dalšího backendu, je ochrana jen kosmetická.

### 4. Lokální pomocné soubory se začnou chovat jako příkazový kanál

Log, dočasný soubor nebo generovaný artefakt někdy přestane být "jen výstup". Další služba ho začne interpretovat jako instrukci.

### 5. Pomocná komponenta má zbytečně silný kontext

Konverzní worker, log processor nebo lokální admin konzole často běží s širším přístupem k souborům, localhostu nebo interním tajemstvím, než bylo potřeba pro samotné zpracování.

## RE: malware pipeline, která sama otevře útočníkův dokument

RE je výborný hlavní příklad, protože celý řetězec je explicitně vidět ve skriptu `process_samples.ps1`. Pipeline dělá přesně to, co organizace potřebovala pro analýzu vzorků:

- vezme soubor z `malware_dropbox`,
- přesune ho do pracovního adresáře,
- přejmenuje `.ods` na `.zip`,
- archiv rozbalí,
- zkontroluje pomocí Yara,
- a pokud projde, otevře ho v LibreOffice.

To je učebnicový příklad toho, jak bezpečnostní problém nevzniká jen v parseru. Vzniká v orchestrace:

- vstupní share je dosažitelný,
- soubor během cesty mění formát a význam,
- Yara validace je slabší než následné otevření v plné aplikaci,
- a LibreOffice běží v běžném uživatelském kontextu, který už umí spustit shell.

Právě proto mohl `.ods` s makrem po nahrání do `malware_dropbox` otevřít shell jako `luke`. Kdyby se někdo soustředil jen na jednu část řetězce, třeba jen na Yara nebo jen na Office, přehlédne to hlavní: problém je celá pipeline důvěry.

## Patents: konverze dokumentu není jeden proces, ale řetězec backendů

Patents ukazuje podobný problém z webové strany. Veřejné artefakty jako `installed.json` a release notes prozradí, že uploadovaný dokument nekončí v jedné PHP funkci. Putuje přes `convert.php` do backendu `gears/pdf` a vedle webu navíc běží další služba na `8888/tcp`.

To je užitečné právě proto, že útoková plocha se rozpadá na více částí:

- web přijme vstup,
- konvertor pracuje s archivní a XML strukturou dokumentu,
- další lokální komponenta zajišťuje převod nebo jinou funkci,
- a až výsledkem je shell nebo další pivot.

Patents tak dobře připomíná, že při review podobných systémů nestačí hledat chybu v `upload.php`. Je potřeba dohledat celou pipeline:

- jaké lokální knihovny a utility jsou zapojené,
- co dělají s dočasnými soubory,
- kdo kam předává výsledek,
- a které části workflow jsou ještě veřejné a které už běží jen na localhostu.

## Haystack: když monitoring stack začne vykonávat obsah souboru

Haystack posouvá téma z dokumentů do log processingu, ale bezpečnostní logika je stejná. Po získání účtu `security` je dostupná lokální Kibana vrstva na `127.0.0.1:5601`, která umí načíst vlastní `/tmp/tm.js`. To je první posun: pomocná lokální konzole přestane být neškodná, jakmile se k ní dostane útočník po footholdu.

Ještě důležitější je však navazující Logstash workflow. Soubory `logstash_tm.log` a `logstash_tm2.log` nejsou v tomto modelu jen logy. Pipeline je čte a řádky `Ejecutar comando:` interpretuje jako instrukce, které běží s vysokými právy.

To je extrémně poučné, protože tu vůbec nejde o parser dokumentu. Jde o stejný trust bug v jiné podobě:

- nějaký soubor vzniká nebo je měněn méně důvěryhodnou vrstvou,
- další komponenta ho slepě interpretuje,
- a monitoring nebo provozní automatizace se tím mění v root kanál.

Haystack je proto důležitá protiváha k RE a Patents. Ukazuje, že bezpečnostní pipeline nemusí pracovat s Office dokumenty. Stačí, aby pracovala s artefaktem, který přestane být braný jako nedůvěryhodný.

## Jak podobné pipeline hledat

Při analýze hostu nebo aplikace se vyplatí jít po konkrétních otázkách.

### Kde je vstupní bod

- upload adresář,
- SMB share,
- bucket,
- databázová tabulka,
- lokální admin konzole,
- nebo adresář s logy a dočasnými soubory.

### Co je další krok po přijetí souboru

- přesun,
- přejmenování,
- rozbalení,
- převod do jiného formátu,
- otevření v desktopové aplikaci,
- import do databáze,
- nebo publikace do jiné složky.

### Která komponenta běží s největší důvěrou

- LibreOffice worker,
- PDF renderer,
- log processor,
- lokální Kibana API,
- vlastní Java nebo PowerShell helper,
- antivirová nebo analýzová služba.

### Kde se ztrácí informace o původu vstupu

Typicky ve chvíli, kdy někdo řekne:

- "to už prošlo kontrolou",
- "to už je jen interní soubor",
- "to už je jen výsledek konverze",
- "to už je jen log pro další zpracování".

Právě tohle je kritický okamžik, kdy se pipeline stává zranitelnou.

## Rozdíl proti článku o document workflow

Téma se částečně překrývá s dokumentovým workflow, ale důraz je jiný.

- Článek [Document workflow jako RCE nebo file-read primitivum](/techniky/document-workflow-jako-rce-nebo-file-read-primitivum) řeší, co dělá nebezpečným samotný dokumentový vstup: XML, ZIP, HTML renderer, Office dokument.
- Tento článek řeší hlavně to, co se děje kolem: přesuny, přejmenování, validace, pomocné logy, lokální konzole a řetězení několika komponent.

Jinými slovy: tam je středem dokument. Tady je středem automatizace.

## Obrana a hardening

### 1. Držet každý krok pipeline jako samostatnou trust boundary

Soubor, který prošel Yara nebo kontrolou formátu, není automaticky bezpečný pro další krok. Každá další komponenta musí pracovat s tím, že stále jde o nedůvěryhodný vstup.

### 2. Omezit schopnosti workerů a pomocných služeb

Konverzní a analýzové procesy nemají mít přístup k citlivým cestám, k localhost-only admin API ani k interním tajemstvím, která nesouvisí s jejich úkolem.

### 3. Nenechat logy a dočasné soubory fungovat jako instrukční kanál

Log processing nebo helper skript nesmí interpretovat volný text v souboru jako příkaz, akci nebo cestu bez velmi tvrdé validace a oddělení důvěryhodných zdrojů.

### 4. Auditovat orchestrace, ne jen jednotlivé nástroje

Bezpečnostní review nemá končit větou "LibreOffice je aktualizovaný" nebo "Logstash má poslední verzi". Důležitější je, kdo mu dává vstup, v jakém formátu a co se stane po zpracování.

### 5. Dělat explicitní model toku souboru

U každého kritického workflow má smysl umět stručně odpovědět:

1. odkud soubor přišel,
2. kdo ho poprvé otevřel,
3. kdo ho převedl nebo přejmenoval,
4. kdo věří výsledku,
5. a s jakými právy to celé běží.

## Shrnutí klíčových poznatků

- Bezpečnostní problém u file processingu často neleží v jednom parseru, ale v celé orchestrace kolem souboru.
- RE ukazuje pipeline, která sama otevře nedůvěryhodný dokument v LibreOffice.
- Patents připomíná, že konverze dokumentu bývá vícevrstvý backend s vlastními lokálními službami.
- Haystack ukazuje, že stejný trust bug existuje i u log processingu a pomocných lokálních konzolí.
- Obrana stojí na tom, že každý krok pipeline zůstane oddělenou trust boundary a že se z výsledků zpracování nestane automaticky "interní a bezpečný" artefakt.

## Co si odnést do praxe

- Při bezpečnostním review souborových workflow nekonči u upload formuláře. Zajímej se o celý řetězec až po poslední worker a poslední dočasný soubor.
- Každé automatické otevření, přejmenování nebo další zpracování souboru je nový rozhodovací bod, kde se může ztratit původní nedůvěra.
- Monitoring, antimalware a konverzní pipeline nejsou automaticky bezpečné proto, že vznikly "na obranu". Jakmile interpretují vstup příliš silně, mění se ve stejnou útokovou plochu jako běžná aplikace.
