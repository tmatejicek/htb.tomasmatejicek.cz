---
layout: post
author: Tomáš Matějíček
title: "SSRF, reverse proxy a localhost trust assumptions"
date: 2020-11-27
tags: ssrf web proxy localhost
---
## Úvod a kontext

Jedna z nejnebezpečnějších iluzí v webové bezpečnosti zní: "tohle je bezpečné, protože je to jen na localhostu". V praxi se tato hranice rozpadá velmi rychle. Jakmile aplikace umí stahovat URL, renderovat vzdálený obsah, posílat webhooky nebo přeposílat požadavky přes proxy, `127.0.0.1` přestává být chráněná zóna a stává se další částí veřejné attack surface.

Právě proto dává smysl spojit do jednoho článku SSRF a důvěru v reverse proxy nebo lokální původ požadavku. V obou případech jde o stejný bezpečnostní omyl: systém věří, že interní síťová poloha nebo lokální zdroj požadavku samy o sobě znamenají bezpečí.

## Proč localhost není bezpečnostní hranice

`localhost` je jen síťová vlastnost, ne autorizační model. Interní služba bývá chráněná tím, že na ni "nejde zvenku". To ale přestává platit ve chvíli, kdy:

- veřejná aplikace sama umí sahat na cizí URL,
- reverse proxy špatně rozlišuje veřejnou a interní cestu,
- backend důvěřuje hlavičkám nebo path normalizaci,
- po prvním footholdu běží na hostu další lokální služby,
- upload nebo helper endpoint funguje jako aplikační proxy.

Interní dostupnost je tedy často jen slabé síťové omezení. Jakmile se najde průchod, z interní služby se stane plnohodnotný útokový vektor.

## Kde SSRF typicky vzniká

SSRF nevzniká jen v "URL fetcheru". Obvykle se schovává v legitimních funkcích, které dávají aplikaci schopnost sama navazovat spojení.

Typické vstupy:

- import vzdáleného repozitáře nebo feedu,
- náhled odkazu nebo obrázku,
- upload "z URL",
- PDF nebo report renderer,
- webhook testery,
- zdravotní endpointy a interní validátory,
- video, audio nebo dokumentové konvertory.

Zranitelný kód často vypadá velmi neškodně:

```python
url = request.json["url"]
content = requests.get(url, timeout=5)
return content.text
```

Bez dalších omezení ale aplikace právě dostala roli síťového zástupce, který umí mluvit i se službami, kam se externí klient nedostane.

## Co přes SSRF hledat

Když už aplikace umí spojení navazovat, otázka nezní jen "lze číst interní stránku", ale hlavně "jaké služby backend sám považuje za důvěryhodné".

Prakticky bývají nejzajímavější:

- admin rozhraní dostupná jen z `localhost`,
- interní vhosty za reverse proxy,
- Redis, registry, metadata endpointy nebo jiné helper služby,
- rozhraní, která povolují více metod než veřejná vrstva,
- diagnostické endpointy a interní API,
- služby, které samy vracejí další tajemství nebo přístupy.

To je důležité i metodicky: SSRF nebývá finální exploit. Často je to prostředek, jak se dostat k dalšímu credentialu, lokální konfiguraci nebo zápisovému API.

## Jak se dopad SSRF mění podle možností útočníka

Ne každé SSRF je stejně silné. Rozhoduje, jak moc kontroly nad požadavkem útočník získá.

### Pouhé načtení obsahu

Nejslabší varianta umí jen stáhnout a vrátit obsah odpovědi. I to ale může stačit na:

- čtení interních stránek,
- objevování neveřejných hostname,
- získání tajemství z interní konfigurace,
- ověření, které služby na localhostu existují.

### Kontrola cíle a schématu

Jakmile lze volit i schéma nebo cíl není omezený na `http://`, dopad roste. Aplikace se může stát mostem do:

- `https://`,
- `ftp://`,
- interních hostname,
- služeb, které nejsou zamýšlené pro veřejný přístup.

### Kontrola cesty, parametrů a metod

Ještě silnější varianta dovolí:

- interní `POST` nebo `PUT`,
- změnu query parametrů,
- použití autentizovaného backendového workflow,
- spuštění interní akce, ne jen čtení stránky.

Právě tady se z SSRF stává skutečný pivot, ne jen recon.

### Kombinace s jinou chybou

V praxi se SSRF často páruje s:

- nedotaženým blacklistem hostname,
- důvěrou v `localhost`,
- helper endpointem, který vrácený obsah dále zpracuje,
- interní službou bez vlastní autentizace,
- reverse-proxy logikou, která jinak skrytou cestu zpřístupní.

## Reverse proxy jako zdroj falešného pocitu bezpečí

Podobný problém vzniká i bez klasického SSRF. Aplikace nebo proxy vrstva může věřit, že něco je bezpečné "jen proto", že:

- cesta je dostupná pouze přes interní hostname,
- backend požadavek vidí jako lokální,
- veřejný server filtruje jen konkrétní tvar URL,
- autorizace stojí na síťové poloze místo na skutečné autentizaci.

To vede k typickým chybám:

### 1. "Only localhost is allowed"

Tahle věta sama o sobě neřeší nic, pokud existuje veřejný endpoint, který umí interní URL načítat nebo přeposílat.

### 2. Proxy chrání cestu, ale ne její význam

Veřejná vrstva může skrývat manager, admin panel nebo interní API, ale backend pořád věří požadavku, který se k němu dostane správným tvarem cesty nebo přes jinou routu.

### 3. Backend důvěřuje zdroji místo identity

Jestliže administrativní endpoint stojí jen na tom, že požadavek "přišel z localhostu", je to spíš síťové pohodlí než bezpečnostní kontrola.

## Co při auditu nebo testu ověřovat

Při hledání SSRF a localhost trust problémů je užitečné projít několik konkrétních otázek:

### Které endpointy navazují spojení

- importy,
- preview,
- webhooky,
- upload z URL,
- helper služby v administraci.

### Jak se validuje cíl

- allowlist vs. blacklist,
- normalizace hostname,
- práce s redirecty,
- podpora různých schémat,
- rozlišení mezi veřejným a interním jménem.

### Co všechno běží interně

- další porty na stejném hostu,
- interní vhosty,
- admin panely,
- cache a message služby,
- monitorovací nebo diagnostické endpointy.

### Kde se autorizace opírá jen o síťovou polohu

- admin endpoint bez loginu, ale jen na localhostu,
- interní API za proxy,
- trust v `X-Forwarded-*` hlavičky,
- rozhodování podle source IP místo identity.

## Jak to navrhovat a bránit

Bezpečná obrana stojí na tom, že interní dostupnost sama o sobě nestačí.

### 1. Přestat věřit blacklistům

Pokud už aplikace musí stahovat vzdálený obsah, bezpečnější je:

- přesná allowlist pravidla,
- pevně definované hostname,
- omezené schéma,
- zákaz redirectů mimo očekávanou sadu cílů.

### 2. Interní služby mají mít vlastní autentizaci

Admin panel, metadata endpoint nebo helper API nemá být chráněný jen tím, že je "schovaný" na localhostu. Jakmile na něj veřejná aplikace umí sáhnout, tohle omezení padá.

### 3. Rozdělit síťové role

Aplikace, která potřebuje stahovat cizí URL, by neměla mít volný přístup ke všemu ostatnímu na stejné hostitelské síti. Egress pravidla a oddělení interních služeb dramaticky snižují dopad SSRF.

### 4. Canonicalizovat a znovu vyhodnocovat cíl

Kontrola cíle musí proběhnout po normalizaci adresy, ne před ní. Jinak se obrana často rozbije na detailech typu jiný tvar hostname, redirect nebo odlišná interpretace cesty mezi proxy a backendem.

### 5. Nezakládat autorizaci na source IP

Síťový původ požadavku může být pomocná informace, ale ne jediný důvod pro přístup do administrace nebo interního API.

## Shrnutí klíčových poznatků

- `localhost` není bezpečnostní hranice. Je to jen místo v síti, ke kterému se dá nepřímo dostat přes veřejnou aplikaci, proxy nebo první foothold.
- SSRF bývá nejnebezpečnější ve chvíli, kdy nevede k jedné interní stránce, ale k celé interní vrstvě služeb a pomocných rozhraní.
- Reverse proxy a interní hostname často vytvářejí falešný pocit bezpečí, pokud backend nemá vlastní autentizaci a důvěřuje síťové poloze.
- Obrana musí řešit jak zdrojové endpointy navazující spojení, tak cílové interní služby, které na síťovou izolaci spoléhají.

## Co si odnést do praxe

- Při auditu se vyplatí hledat všechny funkce, které sahají na URL nebo přeposílají požadavky dál. Často právě ony spojují veřejný web s interní infrastrukturou.
- Interní admin rozhraní, cache, registry nebo helper API nemají být bezpečné jen díky tomu, že neběží na veřejném portu.
- Pokud aplikace umí mluvit do vnitřní sítě, je potřeba ji z pohledu obrany chápat jako potenciální proxy. A podle toho navrhnout validaci, síťová omezení i autorizaci na cílové straně.
