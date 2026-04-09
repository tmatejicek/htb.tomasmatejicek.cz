---
layout: post
author: Tomáš Matějíček
title: "OAuth a zneužití autorizačního toku"
date: 2020-12-24
tags: oauth web auth session oidc
---
## Úvod a kontext

OAuth se často vysvětluje zkratkou "přihlášení přes třetí stranu", ale to je jen část reality. Ve skutečnosti jde o delegaci: klientská aplikace získá omezený přístup k nějakému zdroji nebo identitě na základě autorizačního toku mezi několika stranami. Bezpečnostní problém proto málokdy vzniká tím, že by "OAuth byl rozbitý". Vzniká spíš tím, že aplikace špatně váže dohromady redirect, session, vydaný kód, lokální účet a výsledná oprávnění.

Na [Oouchu](/oouch) je to vidět velmi čistě. Consumer aplikace, autorizační server a interní API vytvářejí řetězec, kde útočník nemusí lámat kryptografii ani obcházet login formulář. Stačí pochopit, kdo komu v daném toku věří a kde se privilegovaný výsledek autorizačního procesu přelévá do jiného kontextu. [Schooled](/schooled) je naopak užitečný kontrast: tam nejde o čistý OAuth bug, ale o session theft a role abuse. Právě tím pomáhá oddělit, co je ještě OAuth problém a co už jen podobně vypadající chyba ve vazbě mezi identitou a lokální rolí.

## Co OAuth řeší a co už ne

Prakticky dává smysl držet v hlavě čtyři role:

- resource owner, tedy uživatel nebo systém, jehož jménem se něco schvaluje,
- client, tedy aplikace, která chce token nebo autorizační kód,
- authorization server, který rozhoduje o vydání kódu nebo tokenu,
- resource server, který token nakonec přijímá a podle něj vrací data nebo provádí akci.

Bezpečnostní chyba obvykle nevzniká při samotném vystavení tokenu. Vzniká ve chvíli, kdy některá strana udělá jeden z těchto skoků:

- vydaný `code` není pevně svázaný s očekávaným clientem, redirect URI nebo session,
- aplikace zamění "uživatel je autentizovaný" za "uživatel má správnou roli",
- interní API věří tokenu v silnějším kontextu, než pro jaký byl určený,
- registrace klienta nebo callbacků není dostatečně omezená,
- `client_credentials` token se použije k přístupu, který má mít jen lidská identita nebo silně omezená služba.

Jinými slovy: OAuth řeší delegaci a přenos oprávnění mezi komponentami. Neřeší automaticky, jestli aplikace správně interpretuje identitu, roli a hranice mezi jednotlivými toky.

## Kde se autorizační tok nejčastěji láme

### 1. Slabá vazba mezi requestem a callbackem

Autorizační kód musí být svázaný minimálně s:

- konkrétním clientem,
- konkrétním redirect URI,
- konkrétní session nebo requestem, který ho vyžádal.

Jakmile aplikace přijme `code` jen proto, že "přišel z autorizačního serveru", je to málo. Útočník pak může docílit toho, že se kód vystavený v privilegovaném kontextu zpracuje jinde, než měl.

Typický tvar toku vypadá nevinně:

```text
GET /oauth/authorize?client_id=consumer&redirect_uri=https://consumer.htb/callback&state=abc123
```

```text
GET /callback?code=SplxlOBeZQQYbYS6WxSbIA&state=abc123
```

Bezpečnostní otázka ale není jen "přišel platný code". Správná otázka zní: přišel tenhle `code` správnému klientovi, pro správnou session a na správný callback?

### 2. Redirect nebo callback v privilegovaném browser kontextu

OAuth často počítá s tím, že uživatel projde autorizací ve vlastním browseru. Pokud ale tok navštíví admin browser, review robot nebo server-side komponenta volaná přes SSRF, privilegovaný kontext se může nechtěně přelít do toku, který ovládá útočník. Přesně tenhle typ privilegovaného browser kontextu rozebírám i v článku [Stored XSS a admin browser a headless review jako útoková plocha](/techniky/stored-xss-a-admin-browser-a-headless-review-jako-utokova-plocha).

To je přesně ten moment, kdy se z běžného callbacku stává útoková plocha. Ne proto, že by byl špatně podepsaný token, ale proto, že autorizační server vydal výsledek ve špatném kontextu.

### 3. Lokální role se odvozují příliš přímo

Token nebo callback obvykle obsahuje identitu, scope nebo jinou formu potvrzení autorizace. Pokud klientská aplikace bez další lokální kontroly z toho rovnou odvodí:

- že uživatel je administrátor,
- že smí spravovat jiného uživatele,
- že může registrovat další klienty,
- nebo že smí sahat na interní API,

pak se z autorizačního toku stává přímý zdroj privilegovaného rozhodnutí.

### 4. Servisní tokeny žijí ve stejném trust modelu jako uživatelé

Zvlášť nebezpečné je, když aplikace vydává tokeny typu `client_credentials`, ale následně je používá k operacím, které jsou ve skutečnosti silnější než běžná servisní delegace. V tu chvíli už nejde o OAuth jako pohodlné API. Jde o interní privilegovaný kanál.

## Oouch: když se OAuth tok propojí s SSRF a interním API

Oouch je dobrý příklad právě proto, že se na něm potkají skoro všechny výše popsané chyby. Zvenku je vidět consumer aplikace, ale anonymní FTP hned na začátku prozradí architekturu: Flask consumer a oddělený Django authorization server. To je první důležitý krok, protože od té chvíle dává smysl vnímat `/oauth` ne jako detail přihlášení, ale jako samostatnou hranici důvěry.

První kritický moment vznikne ve chvíli, kdy kontaktní mechanizmus dovolí server-side návštěvu zadané URL. Tím se z něj stane SSRF, přesně v duchu článku [SSRF, reverse proxy a localhost trust assumptions](/techniky/ssrf-reverse-proxy-a-localhost-trust-assumptions). V kombinaci s OAuth tokem to znamená, že server nebo privilegovaná obsluha navštíví autorizační URL v kontextu, který útočník sám nemá. Výsledkem není rovnou shell, ale mnohem důležitější artefakt: autorizační `code` vydaný v cizím, silnějším kontextu.

Další chyba neleží v samotném kódu. Leží v tom, co s ním systém dovolí dělat dál. Přístupové údaje `develop:supermegasecureklarabubu123!` otevřou registraci vlastní OAuth aplikace, následně lze přes stejný trust chain získat administrátorskou `sessionid` na authorization serveru a vytvořit klienta s grantem `client_credentials`.

To je rozhodující zlom. `client_credentials` se zde nechovají jako úzce omezený servisní token. Chovají se jako klíč k internímu API. Praktickou práci s takovým přesným HTTP requestem rozebírám i v článku [wget a curl](/nastroje/curl):

```bash
curl -X POST 'http://authorization.oouch.htb:8000/oauth/token/' \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data "grant_type=client_credentials&client_id=...&client_secret=..."
```

Takto získaný token už stačí na interní endpoint:

```text
/api/get_ssh/?access_token=...
```

který vrátí SSH klíč pro účet `qtc`. To je výborný příklad špatně navržené hranice. OAuth tok zde nechrání jen přístup k běžnému API. Chráněná operace přímo vydává dlouhodobý systémový přístup.

Na Oouch je proto důležité pojmenovat problém přesně:

- SSRF otevřelo cestu do privilegovaného autorizačního kontextu,
- špatně svázaný OAuth flow dovolil získat hodnotný artefakt,
- registrace klienta a servisní tokeny nebyly omezené dost tvrdě,
- interní API přisoudilo servisnímu tokenu příliš silný význam.

To není "jedna OAuth chyba". Je to řetězec trust assumptions kolem OAuth.

## Schooled jako důležitý kontrast

Schooled je užitečné zmínit právě proto, že pomáhá scope článku ohraničit. Na Moodle 3.9 došlo ke krádeži `MoodleSession` cookie a následnému role abuse, který vedl k převzetí účtu `carter_lianne`. Výsledek zvenku vypadá podobně jako OAuth compromise: útočník se ocitne v cizím, privilegovaném webovém kontextu a z něj pokračuje k dalším oprávněním.

Technicky je to ale jiná chyba.

- V Schooled nejde o autorizační server, redirect nebo výměnu `code` za token.
- Jde o převzetí existující session a o to, že aplikace této session přisuzuje příliš silnou lokální roli.

Právě to je hranice, kterou je užitečné držet i při review. Ne každý případ "ukradl jsem cizí session a mám víc práv" je OAuth problém. Ale stejné otázky o vazbě mezi autentizací a autorizací se tam vracejí.

## Jak podobné chyby poznat při analýze

Při review OAuth nebo OIDC integrace má smysl jít po velmi konkrétních otázkách.

### Jak pevně je svázaný callback

- Je `state` opravdu kontrolovaný a jednorázový?
- Je redirect URI pevně registrované, nebo ho lze ohnout?
- Je `code` svázaný s konkrétním clientem a session?

### Co přesně se děje po úspěšné autorizaci

- Vznikne jen lokální login, nebo i silnější role?
- Dají se po callbacku registrovat další klienti nebo měnit důvěryhodné integrace?
- Otevírá token interní API s vyšší důvěrou než běžná uživatelská session?

### Kdo může autorizaci navštívit místo běžného uživatele

- admin browser,
- review workflow,
- headless robot,
- server-side request přes SSRF,
- nebo interní callback handler volaný z jiného systému.

### Jak oddělené jsou servisní a uživatelské toky

- Používá `client_credentials` jen úzce vymezené machine-to-machine operace?
- Je scope opravdu minimální?
- Vrací servisní endpointy něco, co vytváří dlouhodobý shell nebo další přístup?

## Obrana a hardening

### 1. Callback musí být silně svázaný s requestem

`state`, jednorázovost kódu, pevné redirect URI a kontrola clienta nejsou "doporučené drobnosti". Jsou to základní vazby, bez kterých se z callbacku stane přenosový kanál mezi cizími kontexty.

### 2. OAuth výsledek nesmí automaticky určovat lokální roli

Úspěšná autentizace nebo delegace má vést k identifikaci uživatele nebo klienta. Teprve potom má aplikace rozhodnout, co tato identita smí dělat lokálně. Pokud role vznikne přímo z toho, že autorizace proběhla, je to příliš silné.

### 3. Registrace klientů a servisní granty musí být tvrdě omezené

Samoregistrace klienta, široké scope nebo volné použití `client_credentials` zvyšují hodnotu každého dílčího compromise kroku. Oouch dobře ukazuje, že právě z toho vznikne most do interního API.

### 4. Interní API nesmí vracet dlouhodobé přístupy jen na základě OAuth tokenu

Endpoint, který vrací SSH klíč, tajemství nebo jiný dlouhodobý credential, potřebuje výrazně tvrdší model oprávnění než běžné API. Samotný access token je pro takovou operaci slabá bariéra.

### 5. OAuth a OIDC je potřeba reviewovat jako trust graph

Nestačí auditovat jednu knihovnu. Je potřeba projít:

- kdo vystavuje identitu,
- kdo ji interpretuje,
- kdo ji převádí na lokální účet nebo roli,
- a kde se z ní stává přístup k interním systémům.

## Shrnutí klíčových poznatků

- OAuth bývá kompromitovaný hlavně tehdy, když aplikace špatně sváže callback, session, token a lokální oprávnění.
- Oouch ukazuje, že autorizační tok může být pivot do interního API a dlouhodobých přístupů, pokud se propojí s SSRF a slabou registrací klientů.
- Ne každý session compromise je OAuth chyba. Schooled je dobrý kontrast, protože podobný výsledek vznikl čistě z krádeže session a role abuse.
- Bezpečný návrh odděluje delegaci identity od finálního rozhodnutí, co smí uživatel nebo klient dělat uvnitř aplikace.

## Co si odnést do praxe

- Při auditu OAuth se neptej jen "je tu PKCE a `state`?". Ptej se hlavně "kam až vede důvěra v úspěšně dokončený tok".
- Největší riziko často neleží v login tlačítku, ale v návazných operacích: registraci klienta, interním API, servisních tokenech a mapování lokálních rolí.
- Jakmile lze autorizační tok navštívit nebo dokončit v cizím privilegovaném kontextu, z pohodlné federace se stává útoková plocha.
