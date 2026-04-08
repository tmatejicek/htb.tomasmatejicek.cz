---
layout: post
author: Tomáš Matějíček
title: "Live credential extraction z procesní paměti"
date: 2020-12-03
tags: credentials memory processes browser post-exploitation
---
## Úvod a kontext

Po prvním shellu většina lidí automaticky sahá po souborech: `.env`, `config.php`, `id_rsa`, `history`, vault databázi nebo registry. Jenže část nejcennějších tajemství na disku vůbec neleží. Leží v živém procesu:

- v browseru s přihlášeným administrátorem,
- v klientovi vzdálené podpory,
- v debug utilitě,
- v paměti aplikace, která si právě drží session nebo heslo,
- nebo v procesech, které na serveru nemají co dělat.

Heist je čistý případ právě proto, že root nevzniká z konfiguračního souboru ani z exploitu služby. Vzniká z toho, že na serveru běží Firefox s aktivním přihlášením a jeho paměť obsahuje administrátorské tajemství.

Tenhle článek je o tom, jak přemýšlet o běžících procesech jako o credential store. Ne o dumpování paměti pro efekt, ale o situacích, kdy aktivní proces nese víc hodnoty než celý filesystem.

## O čem tenhle článek je a o čem není

Tenhle text není LSASS průvodce ani forenzní příručka pro každou platformu. Je o užším a praktičtějším problému:

- kdy po footholdu dává smysl podívat se na živé procesy,
- jak poznat, že proces pravděpodobně drží citlivá data,
- a kdy je procesní paměť cennější než statické soubory na disku.

Je také důležité oddělit tohle téma od článku o klientských a desktopových artefaktech. Tam je těžiště v souborech typu KeePass, TeamViewer, OST nebo CliXml. Tady je těžiště v aktivním runtime stavu.

## Proč jsou živé procesy tak cenné

Procesní paměť má několik vlastností, které z ní dělají vysoce hodnotný cíl:

- obsahuje právě používané tajemství, ne jen historický artefakt,
- bývá v ní plaintext nebo snadno rozpoznatelný řetězec,
- drží session, které na disku vůbec nejsou,
- a často existuje jen po dobu aktivního přihlášení nebo běhu aplikace.

To je důležitá obranná i útočná hranice. Když uživatel nebo admin udělá chybu a přihlásí se z nevhodného hostu, nemusí po sobě zanechat heslo v souboru. Stačí, že ho drží proces v paměti právě teď.

## Heist: Firefox na serveru jako trezor s hesly

Na Heist byl první stabilní přístup otevřen přes WinRM účtem `Chase`. V tu chvíli ještě root nevyplýval z žádné lokální zranitelnosti. Rozhodující byly až běžící procesy:

```text
Get-Process Fire*
```

Na serveru běželo několik procesů Firefoxu. To je samo o sobě silná indicie. Browser na serveru není normální provozní stav, obzvlášť pokud jde o server, na kterém se zároveň řeší helpdesk a administrace.

Tím vznikla správná pracovní hypotéza:

- někdo se na serveru interaktivně přihlašoval,
- browser může držet session nebo uložené heslo,
- a paměť procesu může být cennější než konfigurace na disku.

## Proč tady dává smysl dump procesu

Procesní dump není univerzální první krok po footholdu. Dává smysl tehdy, když existuje konkrétní důvod věřit, že proces drží tajemství:

- browser s aktivní webovou session,
- support nástroj,
- mail klient,
- nebo vlastní GUI utility používané administrátorem.

Na Heist byla situace přesně taková. Firefox neběžel na workstation uživatele, ale přímo na serveru. To znamená, že jakékoli uložené nebo právě použité přihlašovací údaje budou mít vyšší hodnotu než běžný desktopový šum.

Použitý postup byl přímočarý:

1. identifikovat konkrétní PID,
2. udělat dump procesu,
3. a v dumpu hledat přihlašovací artefakty.

## Heist: od dumpu ke konkrétnímu heslu

Na Heist se pro dump použil [ProcDump](/nastroje/procdump) a následně se nad výstupem hledaly řetězce související s loginem:

```text
Administrator / 4dD!5}x/re8]FBuZ
```

To je klíčová praktická lekce. Hodnota dumpu nespočívá v tom, že vytvoříš obrovský binární soubor. Hodnota spočívá v tom, že už víš, co v něm hledáš:

- login formuláře,
- názvy uživatelů,
- session tokeny,
- nebo řetězce, které mají vysokou pravděpodobnost vztahu k autentizaci.

Heist je v tomhle směru učebnicový: aktivní browser proces obsahoval přesně to tajemství, které pak otevřelo administratorský WinRM.

## Sharp jako hraniční připomínka runtime kontextu

Sharp není čistý procesní dump případ jako Heist. Je ale užitečný jako připomínka, že běžící služba a její runtime kontext často řeknou víc než statické soubory.

U Sharp byla hlavní hodnota v kombinaci:

- klientské binárky,
- debug endpoint,
- a remoting služba, která aktivně přijímala vstup.

To není totéž jako vytáhnout heslo z dumpu Firefoxu. Ale je to stejný mentální model:

- nestačí hledat jen soubory na disku,
- je potřeba ptát se, který proces nebo služba právě drží nejcennější aktivní stav.

Heist je tedy hlavní case pro live credential extraction. Sharp doplňuje širší lekci, že runtime stav a živé procesy bývají pro post-exploitation podstatnější než pasivní filesystem.

## Jak poznat, že proces stojí za bližší analýzu

Po footholdu dávají největší smysl tyto indikátory:

### 1. Interaktivní aplikace na neobvyklém hostu

Browser, mail klient nebo support GUI na serveru je silná stopa. Znamená to, že někdo host používá i jako pracovní stanici nebo jump host.

### 2. Proces zjevně souvisí s autentizací nebo správou

- browser na admin portálu,
- vzdálený support nástroj,
- debug klient,
- password manager,
- nebo jiná utility, která z principu pracuje s credentialy.

### 3. Na disku nic relevantního neleží, ale přístup zjevně existuje

Pokud víš, že se někdo přihlašuje do citlivé služby, ale nenacházíš config ani uložené heslo na disku, aktivní proces je logická další vrstva.

### 4. Proces je právě aktivní a dlouhodobě běžící

Krátké utility mívají menší hodnotu. Naopak dlouho běžící browser, support agent nebo GUI aplikace často drží stav dost dlouho na to, aby se dal analyzovat.

## Co má v dumpu smysl hledat

Není účelné dump jen slepě „mít“. Praktický postup začíná tím, že máš hypotézu:

- jaká služba je v procesu otevřená,
- jaký účet by mohl být přihlášený,
- a jaké textové vzory s tím souvisí.

Typické cíle jsou:

- jména účtů,
- URL login stránek,
- názvy polí typu `user`, `login`, `password`,
- session tokeny,
- a rozpoznatelné tajné řetězce.

Heist ukazuje, že někdy to může být až nepříjemně přímočaré: v dumpu prostě leží jméno a heslo.

## Rozdíl proti artefaktům na disku

Je důležité držet hranici mezi dvěma blízkými tématy:

### Diskové artefakty

- KeePass databáze,
- TeamViewer config,
- mailbox,
- `.pypirc`,
- `CliXml`.

Tyto věci existují i po restartu a tvoří relativně stabilní credential storage.

### Live procesní stav

- browser s otevřenou session,
- GUI aplikace s právě načteným heslem,
- support proces s aktivním připojením,
- nebo jiná aplikace, která drží tajemství jen v RAM.

Tenhle článek řeší druhou skupinu. Je výbušnější, ale také dočasnější.

## Obrana a hardening

### 1. Nepoužívat servery jako admin desktop

Browser, poštovní klient nebo jiné interaktivní GUI na serveru dramaticky zvyšují dopad prvního footholdu. Heist je přesně důkaz proč.

### 2. Omezit ukládání a autofill citlivých údajů v browseru

Pokud se privilegovaný účet používá přes browser, má být minimalizováno:

- ukládání hesel,
- dlouhé session,
- a obecně vše, co zvyšuje hodnotu aktivního procesu při kompromitaci hostu.

### 3. Chránit proti procesním dumpům a detekovat je

Na obranné straně je důležité sledovat:

- dump utilit,
- podezřelé přístupy k paměti jiných procesů,
- a nestandardní nástroje pro inspection běžících aplikací.

### 4. Oddělit správní relace od kompromitovatelných hostů

Administrátor nemá otevírat citlivé panely z hostu, který zároveň slouží jako server pro méně důvěryhodnou službu.

### 5. Uvažovat o runtime stavu v threat modelu

Nestačí přemýšlet o souborech. Bezpečnostní model musí zahrnout i to, co drží aktivní procesy během práce administrátora nebo supportu.

## Shrnutí klíčových poznatků

- Procesní paměť je samostatná credential vrstva, která po footholdu může mít větší hodnotu než celý filesystem.
- Heist ukazuje učebnicový případ: browser běžící na serveru držel administratorské heslo přímo v aktivním procesu.
- Nejdůležitější není samotný dump, ale správná hypotéza, který proces drží autentizační stav a co v něm hledat.
- Sharp připomíná širší princip: runtime kontext služby nebo klienta bývá často cennější než statický obsah disku.

## Co si odnést do praxe

- Když na kompromitovaném hostu běží browser nebo jiná interaktivní admin aplikace, je to bezpečnostní nález sám o sobě.
- Live credential extraction není forenzní extravagance. V určitých situacích je to nejkratší cesta k další identitě.
- Obrana proti podobným scénářům nezačíná až u EDR. Začíná u toho, že admin session a neprivilegovaná serverová role nesmí sdílet stejný stroj.
