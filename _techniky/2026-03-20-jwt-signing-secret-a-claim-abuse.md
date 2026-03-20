---
layout: post
author: Tomáš Matějíček
title: "JWT signing secret a claim abuse"
date: 2026-03-20
tags: jwt web auth
---
## Úvod a kontext

JWT se často popisuje jako "bezstavová autentizace", ale z bezpečnostního pohledu je důležitější jiná vlastnost: server se rozhoduje podle dat, která přišla od klienta, a věří jim proto, že token nese platný podpis. Jakmile unikne signing secret u symetricky podepisovaných tokenů, nejde jen o technický detail. Útočník získá možnost vytvářet nové tokeny se stejnou důvěryhodností jako aplikace sama.

To ale ještě neznamená, že každý únik secretu má stejný dopad. Rozhodující je, jaké claimy aplikace používá jako autoritativní zdroj pravdy. Pokud token nese jen identifikátor session a server všechna oprávnění znovu dopočítává na backendu, bývá dopad menší. Pokud ale token přímo rozhoduje o roli, tenantovi, administrátorském režimu nebo o tom, ke kterým objektům smí uživatel sahat, uniklý secret se rychle mění v plný aplikační kompromis.

## Co uniklý signing secret skutečně znamená

U algoritmů jako `HS256` nebo `HS512` používá aplikace pro podpis i ověření stejný tajný klíč. To je výhodné na implementaci, ale bezpečnostně to znamená jednoduchou věc: kdokoliv zná secret, umí vytvořit token, který server vyhodnotí jako legitimní.

To má několik praktických důsledků:

- lze měnit existující claimy bez toho, aby to server poznal,
- lze vytvářet úplně nové tokeny pro neexistující nebo privilegovanou identitu,
- lze prodlužovat expiraci nebo měnit kontext, ve kterém aplikace uživatele vyhodnocuje.

Je důležité odlišit dvě otázky:

1. `Umím token podepsat?`
2. `Jak moc se aplikace podle obsahu tokenu skutečně rozhoduje?`

První otázka řeší kryptografickou důvěryhodnost. Druhá řeší reálný dopad.

## Kde signing secret typicky uniká

V praxi secret málokdy unikne přímo z běžící aplikace "z ničeho". Častější jsou provozní a vývojové artefakty:

- starý commit v repozitáři,
- `.env` v přiloženém zdrojáku nebo záloze,
- testovací konfigurace ponechaná v image nebo archivu,
- ukázkové API soubory, které vedle endpointů obsahují i skutečné tajemství,
- dokumentace nebo helper skripty, které počítají s lokálním vývojem a tajemství mají natvrdo.

Právě to je důvod, proč se únik secretu často objeví až po druhém kroku. Nejdřív padne `.git`, záloha nebo testovací build a teprve z něj se vytáhne klíč, který pak otevře celý aplikační trust model.

## Kde se z technického detailu stává plný kompromis

Samotná schopnost podepsat token ještě neurčuje dopad. O něm rozhodují claimy a to, jak s nimi backend zachází.

Typicky rizikové jsou hlavně tyto vzory:

- claim `role`, `is_admin` nebo `admin_cap` přímo zapíná privilegované endpointy,
- claim `sub`, `name` nebo `user_id` se používá bez další validace jako identita aktuálního uživatele,
- claim `tenant`, `account_id` nebo podobný kontext určuje, do jakých dat aplikace sahá,
- claim `email_verified`, `mfa`, `support_mode` nebo `internal` vypíná další bezpečnostní kontroly,
- claim s dlouhou expirací slouží zároveň jako permanentní přihlášení i jako autorizační nosič.

Typický problém vypadá zhruba takto:

```json
{
  "sub": "42",
  "name": "alice",
  "role": "user",
  "admin_cap": false,
  "exp": 1773830400
}
```

Pokud server po ověření podpisu slepě uvěří tomu, že `role=user` nebo `admin_cap=false` skutečně vystihuje oprávnění daného účtu, stačí útočníkovi přepsat payload třeba takto:

```json
{
  "sub": "1",
  "name": "admin",
  "role": "admin",
  "admin_cap": true,
  "exp": 1893456000
}
```

V té chvíli už nejde o obejití loginu, ale o převzetí aplikační identity.

## Jak vypadá claim abuse v praxi

Nejběžnější chyby nebývají v samotném JWT middleware, ale v aplikační logice kolem něj.

### 1. Záměna autentizace a autorizace

Token dokazuje, že někdo zná validní signing secret. Neříká ale sám o sobě, že payload obsahuje správná oprávnění. Pokud backend bere `role=admin` jako dostatečný důvod k přístupu do administrace, autorizace se přesunula do nedůvěryhodného vstupu.

### 2. Důvěra v claim, který měl být jen cache

Vývojáři si často do tokenu uloží roli, jméno nebo seznam oprávnění proto, aby nemuseli sahat do databáze při každém requestu. Jakmile ale token začne být jediným zdrojem pravdy, z optimalizace se stane bezpečnostní závislost.

### 3. Míchání identity a obchodního kontextu

Claim `tenant_id`, `project_id` nebo `account_id` bývá považovaný za harmless routing informaci. Ve skutečnosti často rozhoduje o tom, čí data backend vrátí. U uniklého secretu pak nejde jen o "jsem admin", ale i o horizontální průnik mezi zákazníky.

### 4. Příliš dlouhý život tokenu

Když se signing secret někdy dostal ven, dopad se dramaticky zvyšuje u tokenů s dlouhou expirací nebo bez revokačního modelu. Útočník si pak může vytvářet vlastní "věčně platné" identity i dlouho po opravě původního leaku, pokud nedojde i k rotaci klíče.

## Co z úniku secretu naopak neplyne automaticky

Je užitečné nepřeceňovat dopad mechanicky. Uniklý secret neznamená vždy totéž jako plný účet administrátora.

Dopad bývá omezenější, pokud:

- token nese jen referenci na server-side session,
- autorizace se vždy znovu ověřuje proti backendu,
- citlivé operace vyžadují další server-side kontrolu,
- token je krátkodobý a aplikace umí revokaci nebo rotaci klíčů,
- část claimů je informativní a neřídí přístup.

To ale není argument pro podcenění problému. Je to jen připomínka, že skutečný audit musí vždy projít konkrétní aplikační rozhodování, ne jen konstatovat "unikl JWT secret".

## Na co se zaměřit při analýze aplikace

Při testu nebo review je dobré projít několik konkrétních otázek:

### Kde se secret bere

- je v `.env`, v repozitáři, v image, v helper skriptu,
- je jeden pro všechny prostředí, nebo oddělený,
- měnil se někdy, nebo žije od prvního commitu.

### Jaké claimy backend skutečně používá

- kontroluje se role jen z tokenu,
- používá se `sub` nebo `user_id` přímo do databázového dotazu,
- rozhoduje token o tenantovi, billing účtu nebo interním režimu.

### Co se stane po změně claimů

- otevře se nová administrativní cesta,
- vrátí se cizí data,
- změní se chování background jobů, exportů nebo logovacích endpointů,
- zkrátí se cesta k dalšímu RCE nebo file read primitivu.

### Jak aplikace reaguje na rotaci

- invaliduje staré tokeny,
- umí bezpečně vynutit nové přihlášení,
- odděluje refresh token, access token a interní servisní tokeny.

## Obrana a hardening

Obrana nezačíná u JWT knihovny, ale u architektury důvěry.

### 1. Nepouštět tajemství do repozitáře ani artefaktů

To je základ, ale nestačí jen secret smazat z aktuálního commitu. Jakmile někdy unikl, je potřeba ho považovat za kompromitovaný a rotovat.

### 2. Omezit, co token smí rozhodovat

JWT je vhodné použít k identifikaci uživatele a k přenosu minimálního kontextu. Čím víc autorizační logiky je zakódované přímo v claimu, tím větší dopad má únik klíče.

### 3. Rozdělit autentizaci a autorizaci

Privilegované endpointy nemají stát jen na claimu `admin=true`. Bezpečnější je ověřit identitu z tokenu a oprávnění dopočítat na backendu podle aktuálního stavu účtu nebo role.

### 4. Zkrátit životnost a mít plán rotace

Krátké expirace, oddělené refresh tokeny a kontrola aktivních session významně snižují dopad kompromitovaného secretu. Stejně důležitý je i proces nouzové rotace.

### 5. Auditovat claimy jako vstup

Payload tokenu je z pohledu aplikace pořád vstup od klienta. Podpis z něj nedělá důvěryhodný zdroj pro libovolné obchodní rozhodnutí.

## Shrnutí klíčových poznatků

- Uniklý JWT signing secret neznamená jen "umím se přihlásit", ale hlavně "umím vytvořit payload, kterému server věří".
- Skutečný dopad neurčuje kryptografie sama, ale to, jak silně aplikace spoléhá na claimy při autorizaci.
- Nejčastější příčina problému nebývá v JWT knihovně, ale v tom, že tajemství unikne přes repozitář, zálohu nebo vývojový artefakt.
- Bezpečný návrh odděluje identitu z tokenu od finálního rozhodnutí o oprávnění.

## Co si odnést do praxe

- JWT není problém samo o sobě. Problém vzniká ve chvíli, kdy se z podepsaného payloadu stane jediný zdroj pravdy o oprávněních.
- Signing secret je potřeba chránit stejně jako produkční heslo nebo privátní klíč. Pokud se jednou objeví v repozitáři nebo záloze, musí se rotovat.
- Při auditu aplikace nestačí ověřit, že token má validní podpis. Je potřeba projít i to, které claimy backend používá pro autorizaci a jak by se změnilo chování po jejich podvržení.
