---
layout: post
author: Tomáš Matějíček
title: "NoSQL injection v loginu a extrakce přes `$regex` / `$ne`"
date: 2020-12-16
tags: nosql mongodb web auth login injection
---
## Úvod a kontext

NoSQL injection se často podává jako exotická varianta SQLi. V praxi je ale užitečnější dívat se na ni přes to, co přesně backend udělá s uživatelským vstupem. Pokud aplikace převede parametry login formuláře přímo na operátory databázového dotazu, útočník pak neovládá jen hodnotu typu "uživatelské jméno". Ovládá logiku dotazu.

Na [Mangu](/mango) je to vidět velmi čistě. Login formulář nad MongoDB backendem nejdřív dovolí jednoduchý bypass přes `$ne`, ale skutečně cenné je něco jiného: stejný endpoint jde použít k postupné extrakci reálných přihlašovacích údajů přes `$regex`. To je důležitý rozdíl. Nejde jen o jednorázové "jsem přihlášený". Jde o systematické získání dvojic `uživatel:heslo`, které pak fungují i mimo web.

## Co se u NoSQL injection skutečně kazí

Relativně bezpečný login předá databázi prostá data:

```json
{
  "username": "alice",
  "password": "Secret123!"
}
```

Zranitelný login udělá něco jiného. Uvěří, že vstup z HTTP requestu už je hotový dotaz nebo jeho část. V tu chvíli mohou parametry začít nést operátory typu:

- `$ne`,
- `$regex`,
- případně další operátory měnící logiku filtru.

To je zásadní rozdíl proti běžnému "špatně escapovanému stringu". U NoSQL injection útočník často nemanipuluje se syntaxí jednoho řetězce. Manipuluje s celou strukturou dotazu.

## Dva různé cíle: bypass a extrakce

### 1. Login bypass

Nejjednodušší varianta je donutit dotaz, aby přijal skoro cokoliv. Typický příklad je:

```text
username[$ne]=x&password[$ne]=x
```

Pokud backend tento vstup bez filtrace pošle do Mongo-like dotazu, význam je zhruba:

- `username není x`,
- `password není x`.

To je pro mnoho záznamů pravda zároveň, takže login vrátí první odpovídající účet. Tohle je dobrý potvrzovací test, ale často ještě není nejhodnotnější.

### 2. Extrakce dat

Mnohem zajímavější je okamžik, kdy se stejný formulář stane orákulem pro čtení skutečných hodnot. Jakmile aplikace dovolí prefixový match přes `$regex`, jde znak po znaku ověřovat:

- existuje uživatel začínající na `a`,
- existuje heslo začínající na `t`,
- existuje heslo začínající na `t9`,
- a tak dál.

Tím se z login formuláře stává extrakční kanál, ne jen bypass.

## Mango: když `vendor/composer` napoví správný směr

Na [Mangu](/mango) není nejcennější první payload. Je cenné už to, že veřejný `vendor/composer/installed.json` zúží okruh technologií na MongoDB:

```text
alcaeus/mongo-php-adapter
mongodb/mongodb
```

Právě tohle je dobrá metodická lekce. Bez této indicie by login formulář vypadal jako kandidát na SQLi, slabé heslo nebo klasický auth bug. Tady ale dává smysl zkusit rovnou strukturované parametry:

```text
?login=login&username[$ne]=toto&password[$ne]=toto
```

To potvrdí, že backend interpretuje MongoDB operátory přímo z requestu. V tu chvíli už nemá největší hodnotu samotný login bypass. Větší hodnotu má systematická extrakce.

## Jak funguje extrakce přes `$regex`

Jakmile je potvrzené, že backend akceptuje `$regex`, může útok postupovat po znacích. Praktická logika bývá jednoduchá:

1. zkusit první znak,
2. pokud aplikace vrátí úspěch nebo jiný rozlišitelný výsledek, znak sedí,
3. rozšířit prefix o další znak,
4. pokračovat až do celé hodnoty.

Na Mango právě tímto způsobem vznikly dvě reálné dvojice:

```text
mango / h3mXK8RhU~f{]f5H
admin / t9KcS3>!0B#2
```

To je důležitý rozdíl proti běžné brute force. Útočník netestuje slovník hesel proti lockoutu. Používá aplikaci jako booleovské orákulum, které samo odpovídá, jestli daný prefix sedí.

## Proč je `$regex` nebezpečnější než se zdá

Na první pohled by se mohlo zdát, že prefixová extrakce bude pomalá a málo praktická. Ve skutečnosti bývá velmi použitelná, pokud se sejdou tyto podmínky:

- odpověď loginu jasně odliší úspěch a neúspěch,
- aplikace neomezuje počet pokusů,
- neexistuje lockout nebo agresivní rate limit,
- a vstup jde posílat automatizovaně skriptem.

V tu chvíli už nejde o teoretickou injekci. Jde o plnohodnotný credential recovery kanál.

## Kde se podobná chyba bere

Prakticky se tenhle problém objevuje tam, kde aplikace:

- přímo mapuje parametry z requestu do dotazu,
- používá objekt nebo asociativní pole bez pevného allowlistu klíčů,
- nevynutí, že `username` a `password` musí být obyčejné stringy,
- nebo považuje "Mongo-like query syntax" za pohodlný interní detail, který se omylem propíše až k uživateli.

Jinými slovy: problém není v tom, že by MongoDB samo o sobě bylo nebezpečné. Problém je v tom, že aplikace dovolí uživateli přinést si vlastní operátory.

## Jak takový login bezpečnostně číst

Při review nebo testu dává smysl jít po těchto otázkách:

### Přijímá endpoint jen stringy

- nebo se do backendu propisují i pole, objekty a operátory,
- případně jejich ekvivalent po deserializaci requestu.

### Je login odpověď rozlišitelná

- vrací jiný status,
- jinou hlášku,
- jiný redirect,
- nebo jinou délku odpovědi.

Právě to často rozhodne, jestli půjde jen o bypass, nebo i o extrakci.

### Co se stane po získání hesla

Mango ukazuje důležitou druhou vrstvu. Extrahované heslo nemusí mít cenu jen pro web. Může fungovat i pro:

- SSH,
- `su`,
- nebo jiný management kanál.

Proto je užitečné vnímat NoSQL injection v loginu jako credential theft vektor, ne jen jako auth bypass.

## Obrana a hardening

### 1. Vynutit typy vstupu

`username` a `password` mají být obyčejné stringy. Pokud aplikace dovolí, aby se z nich stal objekt nebo pole s operátorem, problém začíná ještě před samotným dotazem.

### 2. Nepředávat uživatelský vstup přímo do query objektu

Bezpečný login nemá stavět query tak, že uživatel dodá jeho strukturu. Aplikace má sama vytvořit pevný dotaz a uživatelský vstup do něj vložit jen jako datovou hodnotu.

### 3. Sjednotit odpovědi loginu

Pokud login vrací snadno rozlišitelné odpovědi, stává se z něj orákulum pro extrakci. U citlivých endpointů je proto důležité minimalizovat rozdíly v chování při neúspěchu.

### 4. Omezit rychlost a množství pokusů

Rate limiting a další ochrany samy problém neřeší, ale výrazně zvyšují cenu prefixové extrakce.

### 5. Neopakovat hesla mezi webem a shellovým účtem

Mango připomíná, že skutečný dopad často neurčí injekce samotná, ale až reuse extrahovaných hesel napříč vrstvami.

## Shrnutí klíčových poznatků

- NoSQL injection v loginu není jen exotická alternativa SQLi. Je to chyba, kdy uživatelský vstup začne určovat logiku databázového dotazu.
- `$ne` typicky slouží jako rychlý bypass, ale `$regex` je často cennější, protože z login formuláře dělá extrakční kanál pro reálné přihlašovací údaje.
- Mango ukazuje celý praktický řetězec: technologická indicie ve `vendor/composer`, potvrzení operator injection a následná extrakce účtů `mango` a `admin`.
- Obrana stojí hlavně na pevném typování vstupu, explicitně sestaveném query objektu a na tom, že login endpoint nefunguje jako booleovské orákulum.

## Co si odnést do praxe

- Když u webu vidíš Mongo-like backend, nemysli jen na bypass loginu. Mysli i na to, zda endpoint neumí krok za krokem prozrazovat skutečné hodnoty.
- Největší problém často není "jsem přihlášený bez hesla", ale "umím si z loginu vytěžit skutečné heslo a použít ho jinde".
- Pokud aplikace pustí `$ne` nebo `$regex` až do databázového dotazu, je login formulář ve skutečnosti výslechový nástroj nad identitami, ne jen autentizační rozhraní.
