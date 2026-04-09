---
layout: post
author: Tomáš Matějíček
title: "Reverzní inženýrství klienta nebo vlastní binárky jako součást běžného průniku"
date: 2020-11-05
tags: reverse-engineering client binaries protocol credentials
---
## Úvod a kontext

Reverzní inženýrství klienta se často bere jako disciplína pro pwn nebo malware analytiky. V běžném průniku si pod tím mnoho lidí představí něco zbytečně těžkého, pomalého a specializovaného. V praxi ale bývá situace mnohem prostší. Jakmile cíl vydá:

- vlastní klientskou aplikaci,
- interní binárku na share,
- helper DLL,
- desktopový nástroj,
- nebo archiv s implementací služby,

nejde už jen o „nějaký program“. Jde o dokumentaci, kterou si cíl napsal sám pro sebe.

[Fatty](/fatty), [Sharp](/sharp), [BigHead](/bighead) a [Nest](/nest) ukazují přesně tento vzorec. Nikde z nich nevzniká okamžitě shell jen tím, že otevřeš binárku v dekompilátoru. Hodnota je jinde:

- pochopíš protokol,
- vytáhneš endpointy,
- odhalíš hardcoded credentialy,
- uvidíš TODO poznámky vývojářů,
- nebo najdeš způsob, jak si exploit připravit offline.

To je důvod, proč má smysl o reverzním inženýrství klienta mluvit jako o běžné offensive dovednosti, ne jako o exotické specializaci.

## O čem tenhle článek je a o čem není

Tenhle text není návod na disassemblování assembleru po instrukcích. Není ani katalog nástrojů, kde se bude soutěžit mezi Ghidrou, dnSpy a JADX.

Je o rozhodovací logice:

1. kdy má smysl binárku vůbec otevřít,
2. co z ní hledat jako první,
3. jak z výsledku udělat další technický krok,
4. a proč je klientská aplikace často přesnější zdroj pravdy než veřejný banner nebo webový formulář.

Jinými slovy: neřešíme reversing jako cíl. Řešíme ho jako prostředek k pochopení cizího workflow.

## Proč má klientská binárka tak vysokou hodnotu

Vlastní klient nebo interní binárka často obsahuje to, co se vývojáři neobtěžovali abstrahovat nebo chránit:

- pevně zadané hostname a porty,
- názvy endpointů,
- klientské certifikáty,
- hesla ke keystore,
- debug účty,
- serializační třídy,
- komentáře a TODO,
- nebo přesnou podobu validační logiky.

To je zásadní rozdíl proti běžné webové enumeraci. Web ti často ukáže jen vstupní body. Klient ti ukáže i očekávaný formát, důvěru a vývojářské předpoklady.

Proto bývá správná otázka ne:

- `lze tu binárku spustit`,

ale:

- `co mi ta binárka řekne o systému, který obsluhuje`.

## Fatty: klient jako mapa protokolu, TLS i serverových chyb

Fatty začíná anonymním FTP s `fatty-client.jar` a několika poznámkami. To už samo o sobě naznačuje, že nejrychlejší cesta nepovede přes web, ale přes pochopení klienta.

Hodnota reverzní analýzy se tu rozpadá do několika vrstev.

### 1. Klient řekne, jak prostředí vůbec rozchodit

Bez čtení klienta by nebylo zřejmé:

- že aplikace očekává Java 8,
- že server běží na jiném portu než původně,
- a že komunikace stojí na starém TLS nastavení.

To je první důležitá lekce. Někdy není cílem v binárce hledat exploit. Cílem je vůbec vytvořit kompatibilní prostředí pro další analýzu.

### 2. Klient vydá embedded certifikát a jeho heslo

V `TrustedFatty.java` bylo vidět, že aplikace pracuje s `fatty.p12` a heslem:

```text
secureclarabibi123
```

Tím se z klienta stává technická opora pro MITM a čtení aplikačního protokolu. Bez toho by další kroky zůstaly slepé.

### 3. Klient pomůže dovést SQLi k vyššímu dopadu

MITM log a interní poznámky ukázaly, že přihlašování trpí SQL injection a že vývojáři už řešili time-based útoky. To samo ještě nebylo RCE. Skutečný posun přišel až ve chvíli, kdy serverová část a klientské třídy ukázaly nebezpečnou deserializaci v `changePw`.

Tady je dobře vidět, proč je reversing praktický. Nevede jen k tomu, že „vidím heslo“. Vede k pochopení celé cesty:

- jak mluvit s aplikací,
- jaké objekty očekává,
- a kde server slepě důvěřuje klientovi.

## Sharp: interní klient jako dokumentace remoting endpointu

Sharp je podobný, ale technologicky čistší případ. Anonymní share vydal:

- `Client.exe`,
- `Server.exe`,
- `RemotingLibrary.dll`,
- a `notes.txt`.

To samo o sobě říká, že binárky jsou součástí provozního modelu, ne vedlejší vývojový odpad.

### 1. Poznámka vývojářů zúží problém

`notes.txt` obsahovala stručné body:

```text
Todo: Migrate from .Net remoting to WCF
Add input validation
```

To je extrémně hodnotná informace. Neříká ještě, jak exploitovat službu, ale říká dvě podstatné věci:

- aplikace používá .NET remoting,
- a vývojáři sami vědí, že vstup není dostatečně hlídaný.

### 2. Dekompilace klienta doplní endpoint a credentialy

Z `Client.exe` pak šlo vytáhnout přesný endpoint:

```text
tcp://localhost:8888/SecretSharpDebugApplicationEndpoint
```

a debug účet:

```text
debug / SharpApplicationDebugUserPassword123!
```

To je zásadní moment. Bez klienta by port `8888` a `8889` byly jen zajímavé binární služby. S klientem z nich vznikne plnohodnotný model:

- jaký protokol tam běží,
- jaká autentizace se očekává,
- a jak přesně se ke službě připojit.

### 3. Reversing tu nahrazuje slepý brute-force

Sharp ukazuje velmi praktický princip: místo hrubé síly proti WinRM nebo náhodného fuzzingu vlastního portu dává větší smysl dekompilovat klienta a nechat aplikaci, aby sama řekla, jak ji oslovit.

To je přesně typ situace, kdy reverzní inženýrství není luxus navíc. Je to nejefektivnější forma enumerace.

## BigHead: vlastní binárka jako offline laboratoř pro exploit

[BigHead](/bighead) je odlišný tím, že nejde o distribuovaného klienta v úzkém smyslu, ale o veřejně dostupnou binárku vlastní služby v archivech `BHWS_Backup.zip`.

To je dobré připomenutí: pro stejný princip není důležitá role programu. Důležité je, že útočník získá kopii implementace.

### 1. Binárka potvrdí technologii a dovolí práci offline

Endpoint `/coffee` prozradil:

```text
BigheadWebSvr 1.0
```

Jakmile se podařilo získat archiv se samotnou implementací, šla analýza úplně mimo cílový host:

- najít offset,
- identifikovat použitelný `JMP ESP`,
- a připravit exploit bez hlučného testování na produkci.

To je velmi praktická lekce. Když máš binárku, nemusíš cílový server používat jako debugger.

### 2. Reversing tu není jen o RCE

Po prvním shellu vedla cesta dál přes lokální registry, heslo služby `nginx` a interní SSH na `:2020`. BigHead tedy dobře ukazuje, že offline analýza jedné binárky často otevře vícekrokový řetězec, ne jen jednorázový exploit.

## Nest: interní utility jako dokumentace šifrování i protokolu

[Nest](/nest) ukazuje praktičtější a méně "exploitový" případ. Získané zdrojáky `RU Scanner` a binárka `HqkLdap.exe` nesloužily k nalezení paměťové chyby, ale k pochopení:

- jak aplikace ukládá a dešifruje hesla,
- jaké kryptografické parametry používá,
- a jak vypadá interní workflow kolem HQK služby.

Právě to je důležitá připomínka, že reverzní inženýrství klienta často nevede k RCE přímo. Vede k tomu, že z binárky vytáhneš klíč, formát, endpoint nebo šifrovací rutinu, bez nichž by další krok nebyl možný.

## Co mají tyto případy společné

Na první pohled jde o odlišné technologie:

- Java klient,
- .NET remoting klient,
- vlastní Windows služba,

ale všechny dávají stejnou hodnotu:

- zkracují cestu od „mám podezřelý port nebo artefakt“ k přesnému technickému modelu,
- odhalují vývojářské předpoklady,
- a převádějí neurčitou hypotézu na konkrétní krok.

Právě to je sjednocující pointa. Reverzní inženýrství klienta není jen hledání paměťové chyby. Je to způsob, jak si z cizí aplikace vytáhnout:

- dokumentaci,
- konfiguraci,
- přístupové údaje,
- a implicitní důvěru mezi klientem a serverem.

## Kdy má smysl binárku otevřít jako první

Prakticky dává smysl jít po reverzní analýze tehdy, když:

### 1. Cíl sám distribuuje klienta nebo helper nástroj

Pokud je aplikace veřejně na FTP, share nebo portálu, je velmi pravděpodobné, že v sobě nese technické detaily, které server jinak nesděluje.

### 2. Ve hře je vlastní nebo málo obvyklý protokol

Neznámý port, custom service nebo remoting rozhraní jsou typické situace, kdy banner řekne málo, ale klient řekne téměř vše.

### 3. Vývojové artefakty vypadají „příliš interně“

`notes.txt`, DLL knihovny, `Server.exe`, test utility nebo staré zálohy obvykle neunikly náhodou bez následků. Často v nich leží přesně to, co obrana nepředpokládala, že někdo uvidí.

### 4. Hrubá enumerace nikam nevede

Když web i síť zvenku vypadají chudě, ale binárka je k dispozici, reversing může být výrazně rychlejší než další brute-force nebo fuzzing naslepo.

## Jak z toho udělat praktický workflow

Při podobné analýze je užitečné nehledat všechno najednou, ale držet se jednoduché posloupnosti:

### 1. Ověřit, co binárka dělá a s čím komunikuje

- hostname,
- porty,
- URI,
- názvy tříd,
- protokoly,
- knihovny.

### 2. Hledat credentialy a trust materiál

- hesla,
- certifikáty,
- keystore,
- debug účty,
- tokeny,
- connection stringy.

### 3. Hledat vývojářské poznámky a bezpečnostní dluh

- TODO,
- debug logika,
- fallback účty,
- „temporary“ hacky,
- nebo komentáře typu `Add input validation`.

### 4. Převést nález na další krok, ne se v binárce utopit

Smyslem není analyzovat celý program. Smyslem je dostat odpověď na konkrétní otázku:

- jak službu oslovit,
- jaký vstup čeká,
- čemu důvěřuje,
- a kde je nejpravděpodobnější slabina.

## Rozdíl proti článku o klientských artefaktech

Je užitečné tenhle článek oddělit od textu o desktopových a ne-textových artefaktech po footholdu.

Tam jde hlavně o to, co po kompromitaci hostu hledat v:

- KeePass,
- browser dumpu,
- CliXml,
- OST,
- a podobných souborech.

Tady jde o něco jiného:

- aktivně analyzovaný klient nebo binárku,
- pochopení protokolu a architektury,
- a převod vývojářské implementace na další útočný krok.

## Obrana a hardening

### 1. Počítat s tím, že distribuovaný klient je veřejná dokumentace

Jakmile klient opustí firmu a dostane se k uživateli nebo na share, musí se s ním počítat jako s analyzovatelným artefaktem.

### 2. Nehardcodovat credentialy ani trust materiál

Certifikáty, hesla ke keystore, debug účty a endpointy s citlivou rolí nemají být zadrátované přímo do klienta.

### 3. Neopírat bezpečnost serveru o to, že klient je „správný“

Server nesmí důvěřovat serializovaným objektům, parametrům nebo workflow jen proto, že je „posílá náš klient“. Právě Fatty a Sharp ukazují, jak rychle se tahle důvěra rozpadne.

### 4. Hlásit a odstraňovat technický dluh, ne ho jen zapisovat do poznámek

TODO typu `Add input validation` nebo `Migrate from .NET remoting` je pro útočníka varovný maják. Pokud je problém známý, musí se skutečně odstranit.

### 5. Nezveřejňovat staré buildy, zálohy a interní binárky bez kontroly

Archiv služby nebo klienta často stačí k tomu, aby si útočník exploit nebo protokol připravil offline. BigHead je přesně ten případ.

## Shrnutí klíčových poznatků

- Reverzní inženýrství klienta není jen specializovaná disciplína pro reverzéry. V běžném průniku je to často nejrychlejší cesta k pochopení vlastního protokolu, důvěry a credentialů.
- Fatty ukazuje, jak klient vydá TLS materiál, strukturu protokolu a návaznost na serverovou deserializaci.
- Sharp ukazuje, že interní klientská binárka může přímo prozradit debug endpoint a hardcoded účet.
- BigHead ukazuje hodnotu offline kopie vlastní služby: exploit i další kroky lze připravit mimo cílový host.

## Co si odnést do praxe

- Když cíl distribuuje vlastního klienta, považuj ho za technickou dokumentaci, ne za černou skříňku.
- Nehledej v binárce všechno. Hledej to, co ti nejrychleji odpoví na otázku, jak služba funguje a čemu věří.
- Pokud bezpečnost serveru stojí na předpokladu, že nikdo nebude analyzovat klienta, nejde o bezpečnostní model. Jde jen o odklad kompromitace.
