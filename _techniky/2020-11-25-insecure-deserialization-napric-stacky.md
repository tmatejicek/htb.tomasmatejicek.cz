---
layout: post
author: Tomáš Matějíček
title: "Insecure deserialization napříč stacky"
date: 2020-11-25
tags: deserialization java dotnet serialization
---
## Úvod a kontext

Deserializace se často vysvětluje přes jednotlivé technologie: Java serializace, .NET `BinaryFormatter`, Json.NET, YAML parser nebo nějaký konkrétní gadget chain. To je užitečné pro exploit development, ale méně užitečné pro architekturu a code review. Ve skutečnosti totiž nejde o problém jednoho frameworku. Jde o opakující se chybu důvěry: aplikace převezme od klienta nebo jiné nedůvěryhodné vrstvy objekt, typ nebo celý objektový graf a uvěří, že jeho znovuvytvoření je bezpečné. Praktickou roli payload generatoru nad už potvrzenou chybou rozebírám i v článku [ysoserial](/nastroje/ysoserial).

Právě proto dává smysl dívat se na insecure deserialization napříč stacky. Na povrchu vypadá jinak v .NET webu, jinak v Java klientovi a jinak v interním remoting endpointu. Ale bezpečnostní jádro je stejné: útočník neovládá jen data, nýbrž i to, jaký objekt aplikace vytvoří a jaké vedlejší efekty při tom vzniknou.

## Co se při deserializaci skutečně pokazí

Bezpečná serializace pracuje s prostými daty:

- stringy,
- čísla,
- seznamy,
- slovníky,
- nebo předem známé datové typy bez vedlejších efektů.

Nebezpečná deserializace začíná ve chvíli, kdy vstup může rozhodovat i o:

- konkrétním typu objektu,
- třídě, která se má vytvořit,
- metodě nebo konstruktoru, který se při tom spustí,
- nebo objektovém řetězci, jehož vedlejší chování skončí vykonáním příkazu.

Tím se z formátu pro přenos dat stává mechanismus pro spouštění kódu nebo pro vyvolání chování, které vývojář nikdy nezamýšlel jako veřejný vstup.

## Proč je problém tak přenositelný mezi technologiemi

Jednotlivé stacky používají jinou syntaxi, ale velmi podobný mentální model:

- .NET může přijmout JSON s `$type` nebo binární objekt z remoting kanálu,
- Java může přes parser nebo serializační mechanismus vytvořit nečekaný objekt,
- YAML loader může ze zdánlivě datového dokumentu materializovat instanci třídy,
- vlastní klient/server protokol může slepě přijmout serializovaný objekt jen proto, že "to posílá náš klient".

Rozdíl tedy není v tom, jestli jde o JSON, YAML nebo binární stream. Rozdíl je jen v tom, jak daná platforma dovolí propojit data s typovým systémem a se side-effecty při vytváření objektů.

## Tři základní vzory, které se opakují

### 1. Útočník ovládá typ

Nejnebezpečnější varianta je ta, kde vstup přímo říká, jaký objekt má aplikace vytvořit. V .NET to může být například JSON s polem `$type`:

```json
{
  "$type": "System.Windows.Data.ObjectDataProvider, PresentationFramework",
  "MethodName": "Start"
}
```

V jiném stacku se stejná logika projeví jinak:

- YAML tagem typu `!!nějaká.třída`,
- typem objektu přeneseným v binárním streamu,
- nebo remoting zprávou, která už ze své podstaty přenáší serializovaný objekt, ne jen data.

Jakmile vstup určuje typ, je potřeba velmi přísně hlídat, které typy jsou vůbec přípustné. Bez toho si aplikace sama staví objekt z nedůvěryhodného materiálu.

### 2. Útočník neovládá jen data, ale i vedlejší efekty

Mnoho nebezpečných gadgetů není nebezpečných proto, že "obsahují shell". Jsou nebezpečné proto, že jejich vytvoření nebo následná práce s nimi:

- zavolá metodu,
- otevře proces,
- načte URL,
- nebo aktivuje jinou komponentu, která už dál udělá něco citlivého.

To je důležité i pro review. Není nutné hledat jen explicitní `exec()`. Stačí najít objektový graf, který po znovuvytvoření vede k nečekané akci.

### 3. Server věří, že vstup poslal "náš klient"

To je častý a podceňovaný motiv. U interních klientů, desktopových aplikací nebo remoting endpointů se často předpokládá:

- vstup vytváří náš binární klient,
- nikdo jiný protokol nezná,
- a proto není třeba input tvrdě omezovat.

Jenže veřejně dostupný klient lze stáhnout, dekompilovat a znovu implementovat. Jakmile server slepě přijímá serializovaný objekt jen proto, že očekává "našeho klienta", je to stejný problém jako u webového API bez validace vstupu.

## Jak se to projevilo v různých případech

Na [Jsonu](/json) se nebezpečná deserializace schovala do cookie `OAuth2`. Frontend ji chápal jako bearer token, backend ji ale zpracovával jako objekt a dovolil přes `$type` vytvořit `ObjectDataProvider`, který spustil proces na serveru. Tady je dobře vidět, že problém neleží v cookie sama o sobě. Leží v tom, že server přijal klientský JSON jako autoritativní popis objektu.

Na [Sharpu](/sharp) šlo o interní .NET remoting endpoint. Ten už ze své podstaty nepřenášel jen data, ale serializované objekty. Dekompilace klienta odhalila endpoint, debug credentials i to, že služba slepě přijímá deserializovaný vstup. Výsledek nebyl "jen bug v remotingu". Výsledek byl celý produkční debug kanál postavený na důvěře k objektům od klienta.

Na [Ophiuchi](/ophiuchi) v Java/Tomcat prostředí zase parser YAML nepůsobil jako klasická serializace. Praktický efekt byl ale stejný: uživatelský vstup vedl k materializaci nečekaného typu, který pak stáhl a aktivoval další kód. Formát je jiný, ale bezpečnostní chyba zůstává totožná: data určují typ a tím i chování.

Na [Fatty](/fatty) stál další případ na tom, že vlastní Java klient komunikoval se serverem přes serializované objekty. Nejdřív bylo potřeba klient rozchodit, dekompilovat a pochopit protokol. Teprve potom vyšlo najevo, že server-side funkce pro změnu hesla slepě zpracovává serializovaný objekt `ClientCredential`. To je velmi čistý příklad toho, že insecure deserialization není jen webový problém. Je to problém celého návrhu klient/server důvěry.

[Cereal](/cereal) přidává ještě jinou variantu téhož vzorce. Server po admin akci stáhl JSON s typem `Cereal.DownloadHelper`, deserializoval ho a vytvořil z něj server-side download workflow. Znovu tedy nešlo o to, že by klient "poslal data". Klient poslal instrukci, jaký objekt má server vytvořit a co s ním udělat.

## Jak takový problém poznat při review

Při code review a architekturní analýze se vyplatí hledat několik konkrétních signálů.

### Typ rozhoduje vstup

- `$type`, `TypeNameHandling`, polymorfní loader,
- YAML tagy vedoucí na konkrétní třídy,
- `BinaryFormatter`, remoting a jiné mechanismy, které obnovují objekty včetně jejich typu,
- vlastní protokol přenášející serializované objekty místo datových DTO.

### Server přijímá bohatý objekt tam, kde by stačila data

Pokud API ve skutečnosti nepotřebuje nic víc než:

- identifikátor,
- několik stringů,
- jednoduchou strukturu,

a přesto přijímá nebo rekonstruuje komplexní objekty, je to silný varovný signál.

### Důvěra stojí na původu klienta, ne na validaci

- "posílá to naše desktop appka",
- "je to interní endpoint",
- "nikdo nezná ten formát",
- "debug API je schované".

To všechno jsou slabé předpoklady. Jakmile je klient nebo binárka dostupná, je jen otázka času, kdy ji někdo rozebere.

## Co je důležitější než gadget chain

U deserializace je snadné sklouznout k seznamu gadgetů a payloadů. Pro obranu je ale důležitější rozlišit tři úrovně rizika:

1. přijímám jen prostá data,
2. přijímám polymorfní data, ale s pevným allowlistem typů,
3. přijímám volně specifikovaný objekt nebo celý objektový graf.

Teprve třetí úroveň bývá skutečně katastrofická. V tu chvíli aplikace už neparsuje vstup. Ona z něj rekonstruuje programové chování.

## Obrana a bezpečnější návrh

### 1. Přenášet data, ne objekty

Nejbezpečnější řešení je architektonické: veřejné a interní API má přijímat jednoduché datové struktury, ne serializované objekty s typovou informací.

### 2. Vypnout nebo omezit polymorfní typy

Pokud platforma umí deserializovat podle typu ze vstupu, je to potřeba chápat jako vysoce rizikovou funkci. Většina aplikací ji pro běžný provoz vůbec nepotřebuje.

### 3. Opustit nebezpečné legacy mechanismy

U .NET to znamená nepoužívat staré serializer/remoting mechanizmy pro nedůvěryhodná data. U jiných stacků to znamená totéž v jejich vlastním slovníku: nepoužívat loader nebo parser v režimu, který materializuje libovolné objekty.

### 4. Klientský původ nebrat jako bezpečnostní záruku

To, že zprávu vytváří "náš klient", není bezpečnostní kontrola. Jakmile lze klienta stáhnout nebo dekompilovat, musí být server připravený na ručně vytvářený vstup.

### 5. Oddělit deserializační hranici od citlivých akcí

Proces, který obnovuje objekty nebo bohaté dokumenty, nemá běžet v kontextu, kde může:

- spouštět procesy,
- sahat na citlivý filesystem,
- nebo přistupovat k interním tajemstvím bez další bariéry.

## Shrnutí klíčových poznatků

- Insecure deserialization není chyba jednoho frameworku, ale opakující se chyba důvěry k objektům a typům z nedůvěryhodného vstupu.
- Rozdíl mezi JSON, YAML, remotingem nebo vlastním binárním protokolem je menší, než se zdá. Ve všech případech jde o to, kdo rozhoduje o typu a vedlejších efektech rekonstruovaného objektu.
- Největší riziko vzniká tam, kde server přijímá volně specifikovaný objekt místo jednoduchých dat.
- Obrana stojí na tom, že se přenášejí data, omezují typy a opouštějí legacy mechanizmy, které automaticky obnovují objekty z klientského vstupu.

## Co si odnést do praxe

- Při review se neptejte jen "je tu `BinaryFormatter` nebo SnakeYAML?". Ptejte se, kde vstup určuje typ a jaké chování se z toho může zrodit.
- Deserializace je nebezpečná i tehdy, když nejde o veřejný web. Interní klient, debug endpoint nebo vlastní protokol mají stejný problém, pokud server slepě věří objektům od druhé strany.
- Pokud API nebo služba nepotřebuje přijímat objektový graf, nemá ho přijímat. Ve většině případů stačí prosté DTO a právě tím se odřízne celá třída rizika.
