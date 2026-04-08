---
layout: post
author: Tomáš Matějíček
title: "Podepsaný aplikační stav na klientovi a server-side trust"
date: 2020-10-30
tags: web auth viewstate java trust serialization
---
## Úvod a kontext

Po článcích o JWT je užitečné udělat krok stranou. Problém důvěry v klientem vracená data nekončí u tokenů. Řada frameworků a aplikačních vzorů ukládá část stavu na klienta a server pak při dalším požadavku ověřuje, že se s tímto stavem nemanipulovalo. Jakmile ale unikne tajemství používané pro podpis nebo šifrování, celý model důvěry se rozpadá stejně rychle jako u kompromitovaného tokenu.

Tady je důležité držet správnou perspektivu. Nejde o to, že by „podepsaný stav na klientovi“ byl automaticky chyba. Jde o to, že server vrací klientovi vlastní interní stav a později mu znovu věří. Jakmile útočník získá klíč, kterým se tato důvěryhodnost potvrzuje, může si klientskou reprezentaci stavu přepsat tak, aby server vykonal něco úplně jiného, než zamýšlel.

## Co je podepsaný aplikační stav

Jde o situaci, kdy aplikace:

- serializuje část interního stavu do požadavku nebo formuláře,
- pošle ji klientovi,
- a při dalším requestu ji vezme zpět jako vstup,
- přičemž důvěryhodnost odvozuje z podpisu, MAC nebo šifrování.

Typicky může jít o:

- view state u server-side webových frameworků,
- serializovaný objekt v hidden poli formuláře,
- stav komponenty nebo workflow vracený mezi requesty,
- jakýkoliv aplikační blob, který si server ukládá „u klienta“ a později ho znovu načítá.

To je architektonicky velmi podobné JWT. Rozdíl je v tom, že se obvykle neřeší identita a autorizace, ale interní aplikační stav. Bezpečnostní otázka je ale stejná: kdo rozhoduje o obsahu a kdo rozhoduje o jeho důvěryhodnosti.

## Kdy je to ještě legitimní a kdy už nebezpečné

Podepsaný klientský stav může být legitimní optimalizace, pokud:

- nese jen nezávadný kontext,
- server ho po načtení používá opatrně,
- a klíč pro jeho ochranu je opravdu tajný.

Nebezpečné to začíná být ve chvíli, kdy:

- stav přímo ovlivňuje, jaký objekt se má deserializovat nebo vykonat,
- podpisový secret leží v záloze, repozitáři nebo konfiguračním exportu,
- server předpokládá, že „validní podpis = bezpečný obsah“,
- serializovaný stav obsahuje strukturu, která může při načtení měnit chování aplikace.

To je důležitý rozdíl. Kryptografie sama ještě nedělá z klientského vstupu bezpečný objekt. Jen říká, že ho vytvořil někdo se znalostí klíče. Pokud se tento klíč dostane ven, server začne útočníkovi věřit stejně, jako kdyby šlo o vlastní interní stav.

## Příklad z praxe: Arkham a JSF ViewState

Arkham ukazuje tento vzor velmi čistě. Samotný Tomcat na `:8080` nebyl hlavní problém. Rozhodující byla až záloha aplikace `appserver.zip`, z níž šlo vytáhnout konfiguraci s klíčem `org.apache.myfaces.SECRET`. Tím se otevřela cesta ke zneužití JSF ViewState.

To je přesně ta situace, kde je potřeba nepřeskočit příčinu a následek:

- problém nebyl „Tomcat je zranitelný“,
- problém nebyl ani „JSF je z principu špatně“,
- problém byl v tom, že aplikace vracela klientovi podepsaný stav a zároveň umožnila útočníkovi získat secret, kterým se tato důvěryhodnost potvrzovala.

Ve chvíli, kdy byl secret známý, šlo připravit nový ViewState payload a server ho přijal jako legitimní. To vedlo až k RCE.

Arkham je cenný právě tím, že ukazuje celý řetězec:

1. nejdřív unikne aplikační záloha,
2. ze zálohy vypadne signing secret,
3. secret se použije k výrobě nového klientského stavu,
4. server tento stav znovu načte a vykoná s důvěrou, kterou by jinak měl jen k vlastní interní reprezentaci.

## Proč je to důležité i mimo JSF

Není potřeba tvrdit, že všechny frameworky fungují stejně. To by byla nepřesnost. Důležité je spíš pochopit přenositelný model:

- server někdy přesune stav na klienta,
- později ho od klienta znovu přijme,
- a bezpečnost stojí na tom, že útočník nezná klíč, kterým se ověřuje důvěryhodnost tohoto stavu.

Jakmile je klíč kompromitovaný, aplikace ztrácí schopnost rozlišit mezi:

- stavem, který vytvořila sama,
- a stavem, který si vytvořil útočník.

To je důvod, proč je toto téma širší než jedna implementace ViewState. Stejný problém se může objevit v různých vlastních serializačních mechanikách, hidden polích, klientských „session blobech“ nebo frameworkových state kontejnerech.

## Jaké dopady z toho obvykle plynou

Nejčastěji jde o některou z těchto variant:

### 1. Manipulace workflow

Útočník změní stav tak, aby server přeskočil část procesu, přijal jiný objekt, otevřel jinou větev formuláře nebo sáhl na jiný zdroj.

### 2. Podvržení identity nebo kontextu

Pokud je v klientském stavu uložený uživatel, role, objekt nebo tenant, uniklý signing secret mění tento stav v další formu claim abuse.

### 3. Deserializační nebo gadget-based RCE

Nejnebezpečnější varianta nastává tehdy, když server po validaci podpisu načítá serializovanou strukturu, která může vést k vykonání kódu nebo k nebezpečnému side effectu.

Právě tady je důležité nepodlehnout zkratce „podepsané = bezpečné“. Podepsané jen znamená, že server věří původu. Ne že je obsah sám o sobě bezpečný.

## Na co se při analýze zaměřit

Při review podobných aplikací má smysl projít několik konkrétních otázek:

### Kde se bere klíč nebo secret

- je v repo, záloze nebo image,
- je stejný mezi prostředími,
- žije dlouhodobě bez rotace,
- leží v exportu aplikace nebo v konfiguračním artefaktu.

### Jaký stav klient skutečně vrací

- jde jen o jednoduchý identifikátor,
- nebo o serializovanou strukturu,
- obsahuje stav formuláře, workflow nebo komponenty,
- rozhoduje o tom, co server bude dál dělat.

### Co server se stavem po ověření udělá

- jen ho použije jako cache,
- nebo z něj rekonstruuje objekt,
- nebo podle něj dokonce spustí citlivou logiku.

### Jak těsně je stav svázaný s interním modelem aplikace

Čím víc klientský blob připomíná interní datovou strukturu frameworku nebo aplikace, tím větší je riziko, že se z něj po kompromitaci klíče stane přímý vstup do nebezpečné serverové logiky.

## Obrana a hardening

### 1. Nenechávat signing secret v přenositelných artefaktech

Arkham připomíná úplně základní věc: jakmile je secret v záloze nebo exportu aplikace, není to už tajemství. Ochrana klientského stavu pak přestává existovat.

### 2. Minimalizovat, co se na klienta vůbec ukládá

Čím bohatší a strukturálně složitější je klientský stav, tím větší je dopad kompromitace klíče. Ukládat na klienta jen minimum je často nejúčinnější obrana.

### 3. Nepovažovat validní podpis za důkaz bezpečného obsahu

Podpis řeší integritu vůči někomu, kdo nezná klíč. Neřeší bezpečnost samotné struktury, kterou aplikace načítá a zpracovává.

### 4. Oddělit integrity check od nebezpečné deserializace

Pokud aplikace musí klientský stav přijmout, je vhodné:

- držet ho co nejjednodušší,
- nepoužívat ho jako přímý vstup do složitých objektů,
- a po validaci pořád uplatňovat bezpečnostní kontrolu nad obsahem.

### 5. Mít plán na rotaci a invalidaci

Pokud se signing secret jednou objeví v exportu, historii nebo backupu, musí se považovat za kompromitovaný a celý mechanismus klientského stavu je potřeba přegenerovat nebo invalidovat.

## Shrnutí klíčových poznatků

- Podepsaný aplikační stav na klientovi je bezpečnostně stejná kategorie problému jako JWT: server znovu věří datům, která se k němu vrací od klienta.
- Jakmile unikne signing secret, klientský stav se mění z interní optimalizace na útočníkem ovladatelný vstup.
- Arkham ukazuje, že rozhodující nebývá samotný framework, ale kombinace dvou faktorů: klientem vracený stav a kompromitovaný secret z aplikační zálohy.
- Bezpečný návrh stojí na tom, že klientský stav je malý, přísně omezený a po validaci se s ním stále zachází jako s potenciálně rizikovým vstupem.

## Co si odnést do praxe

- Když vidíš v aplikaci podepsaný nebo serializovaný stav vracený klientovi, ptej se stejně jako u JWT: co se stane, když někdo získá signing secret.
- Záloha aplikace s konfigurací a secrety není jen „informační leak“. Často přímo ruší důvěryhodnost bezpečnostních mechanismů uvnitř frameworku.
- Validní podpis neznamená bezpečný objekt. Pokud aplikace po ověření podpisu bezmyšlenkovitě načítá složitý stav, problém je v trust modelu, ne jen v kryptografii.
