---
layout: post
author: Tomáš Matějíček
title: "Lokálně dostupné služby po footholdu: localhost není boundary"
date: 2020-11-17
tags: localhost post-exploitation pivot services
---
## Úvod a kontext

Mnoho systémů se spoléhá na jednoduchou domněnku: tahle služba je bezpečná, protože poslouchá jen na `127.0.0.1`. To může fungovat proti anonymnímu návštěvníkovi z internetu. Jakmile ale útočník získá první shell, situace se zásadně mění. Z `localhostu` se stává plnohodnotná útočná plocha a interní porty najednou představují další vrstvu systému, kterou je potřeba prozkoumat stejně disciplinovaně jako veřejný web.

Tohle téma se částečně dotýká i SSRF, ale není to totéž. Samostatný článek o [SSRF, reverse proxy a localhost trust assumptions](/techniky/ssrf-reverse-proxy-a-localhost-trust-assumptions) řeší, jak se na interní službu dostat ještě před footholdem. Tento text naopak řeší situaci po prvním shellu: co hledat na `127.0.0.1`, proč to bývá důležité a jak z lokálních služeb nevytvářet tichou druhou administrativu.

## Proč localhost po footholdu přestává chránit

`127.0.0.1` není autorizační model. Je to jen síťové umístění. Jakmile je útočník uvnitř hostu nebo kontejneru, otevírá se mu přístup k věcem, které byly dosud schované za lokálním bindem:

- interní API,
- GraphQL nebo administrační endpointy,
- Redis, InfluxDB a další datové služby,
- chaty a helper démoni,
- Kibana, konzole a diagnostická rozhraní,
- binární protokoly, které nikdo nečekal na veřejném portu.

To je důvod, proč se po footholdu nevyplatí bezhlavě skenovat celou síť, ale velmi cíleně číst, co běží lokálně a jaký bezpečnostní význam tyto služby mají.

## Co typicky na localhostu běží

Lokální služby po footholdu mívají několik opakujících se podob.

### 1. Interní administrační API

Aplikace často skrývá citlivější endpointy za lokální bind:

- interní GraphQL,
- admin backend,
- management port,
- debug nebo orchestration API.

Předpoklad bývá jednoduchý: na localhost se přece veřejný uživatel nedostane. Po shellu je tento předpoklad pryč.

### 2. Datové služby a cache

Redis, InfluxDB, databáze, message brokery nebo jiné pomocné služby často neběží veřejně, ale lokálně. To je rozumné pouze tehdy, pokud se zároveň počítá s tím, že první kompromitace hostu tyto služby okamžitě zpřístupní.

### 3. Pomocné interní workflow

Na localhostu neběží jen standardní produkty. Často tam bývá i:

- vlastní binární protokol,
- lokální daemon pro správu,
- interní chat,
- pomocný validátor,
- log processing workflow,
- nebo jiný kus provozní automatiky.

Právě tyto méně nápadné služby bývají po footholdu nejzajímavější, protože jsou navržené s implicitní důvěrou k lokálnímu volajícímu.

## Jak tenhle vzorec vypadal v konkrétních případech

Na [ServMonu](/servmon) byl po prvním SSH přístupu klíčový až NSClient++ na `127.0.0.1:8443`. Zvenku sice působil jako interní admin rozhraní, ale po port forwardu a přečtení hesla z `nsclient.ini` se změnil v normálně dosažitelnou službu, která uměla nahrát a spustit vlastní skript. Veřejný web a FTP tedy otevřely jen foothold; skutečný privesc ležel až na localhostu.

Na [Devzatu](/devzat) se po vstupu pod účtem `patrick` ukázala interní InfluxDB na `127.0.0.1:8086`, z níž šlo vytáhnout hesla dalších uživatelů. Až další lokální služba na `8443` otevřela přístup k chatu a jeho funkci `/file`, která vydala rootův klíč. Tady je velmi dobře vidět, že lokální služby po footholdu tvořily celý druhý útokový řetězec.

Na [Luanne](/luanne) zase webové RCE samo o sobě nestačilo. Rozhodující byla až localhost-only služba na `127.0.0.1:3001`, která po cracknutí `.htpasswd` vydala soukromý klíč `r.michaels` a otevřela stabilní SSH foothold.

Na [OpenAdminu](/openadmin) se stejný pattern projevil na `internal.openadmin.htb` svázaném s `127.0.0.1:52846`. Veřejná RCE v ONA otevřela jen první shell; skutečný posun přinesl až interní vhost, který po port forwardu vydal klíč `joanna`.

Na [Horizontallu](/horizontall) zase první shell vedl jen do Strapi účtu `strapi`, ale teprve `127.0.0.1:8000` odkryl interní Laravel s vlastním RCE. Bez lokální enumerace a SSH port-forwardu by root část vůbec nepřišla na řadu.

Na [Oouchu](/oouch) se localhost a interní síť proměnily v druhou polovinu celého řetězce. Po SSH jako `qtc` už nebyl hlavním tématem veřejný consumer, ale interní kontejnery, `uwsgi` socket a D-Bus workflow dosažitelné až po prvním shellu.

Na [Rope](/rope) stál další případ na tom, že uživatelský shell dovolil přes SSH port-forward osahat lokální službu na `13907`. Nebyla veřejná a sama působila jako interní pomocník. Teprve ruční práce s protokolem a následné zhroucení privilegovaného wrapperu do debuggeru z ní udělaly cestu k rootu.

Na [Haystacku](/haystack) zase běžela Kibana konzole jen na `127.0.0.1:5601`. Zvenku se k ní nešlo dostat, ale po shellu se ukázalo, že umí načíst lokální soubor a že vedle ní běží log processing workflow s vysokými právy. Opět tedy nešlo o "jednu lokální službu", ale o celý lokální ekosystém, který po footholdu najednou ztratil ochranu.

I [Ready](/ready) ukazuje stejný vzorec v kompromitovaném kontejneru. Jakmile útočník získal shell uvnitř GitLab kontejneru, lokální Redis už nebyl interní služba chráněná bindem na loopback. Stal se přirozenou součástí útočné plochy daného runtime prostředí.

## Jak k localhostu po footholdu přistupovat

Po prvním shellu má smysl uvažovat o localhostu jako o nové mapě systému, ne jako o jednom triku.

### 1. Nejdřív zjistit, co poslouchá

Bez přehledu o lokálních portech není možné odhadnout další směr. Důležitější než agresivní scan bývají:

```bash
ss -lntp
netstat -ano
lsof -i -P -n
```

Cílem není "najít všechno". Cílem je pochopit:

- které služby jsou jen lokální,
- pod jakým účtem běží,
- a jak souvisejí s právě kompromitovanou aplikací.

### 2. Číst konfigurace a pomocné soubory

Lokální port sám o sobě ještě neřekne, co služba dělá. Hodnotu často přinese až:

- `docker-compose.yml`,
- systemd unit,
- skript volaný přes `sudo`,
- konfigurační soubor aplikace,
- logy a pomocné dokumenty,
- nebo výpis z webového repozitáře.

Právě tam se ukáže, zda je lokální služba jen cache, nebo třeba plnohodnotné administrační API.

### 3. Testovat lokální služby v jejich kontextu

Jakmile je jasné, co na localhostu běží, dává smysl zkoumat:

- zda služba opravdu vyžaduje autentizaci,
- jestli důvěřuje lokálnímu volajícímu příliš široce,
- jestli vydává tajemství nebo spouští další akce,
- a zda se přes ni nedá aktivovat jiný privilegovaný workflow.

Pořád nejde o bezhlavé fuzzování. Jde o čtení architektury hostu po kompromitaci.

## Co se na localhostu hledá nejčastěji

Prakticky se vyplatí cíleně myslet na tyto kategorie:

- interní webové administrace a GraphQL endpointy,
- lokální databáze a cache,
- monitoring a observability rozhraní,
- message queue a pomocné API pro joby,
- servisní porty vlastních binárek,
- interní chaty, konzole a file-read helpery,
- desktopové a management nástroje, které spoléhají na lokální trust.

Společný vzorec je pořád stejný: služba nebyla navržená tak, aby odolávala nedůvěryhodnému uživateli na stejném hostu.

## Obrana a bezpečnější návrh

### 1. Localhost nebrat jako jedinou ochranu

Interní služba má mít vlastní autentizaci a vlastní autorizaci. Bind na `127.0.0.1` může být doplňkové omezení, ne jediný důvod, proč je něco "bezpečné".

### 2. Oddělit role a práva procesů

Jestli veřejná aplikace běží pod účtem, který po compromise automaticky vidí citlivé lokální služby, je hranice mezi webem a interní vrstvou příliš tenká.

### 3. Minimalizovat hodnotu lokálních služeb

Lokální endpoint nemá vracet tajemství, číst libovolné soubory ani spouštět privilegované akce jen proto, že ho volá něco "ze stejného stroje".

### 4. Po compromise počítat s plným přístupem k loopbacku

Threat model hostu musí předpokládat, že první shell automaticky znamená přístup k `127.0.0.1`. Pokud některá interní služba tento scénář nepřežije, je třeba ji přepracovat nebo izolovat.

### 5. Monitorovat neobvyklé lokální vztahy mezi službami

Lokální přístupy mezi procesy, které se v běžném provozu nepotkávají, bývají silný signál útoku i chybného návrhu. Totéž platí pro podezřelé port-forwardy a náhlé volání interních konzolí z nečekaného účtu.

## Shrnutí klíčových poznatků

- Po prvním shellu se `127.0.0.1` mění v plnohodnotnou útočnou plochu.
- Na localhostu často běží interní API, datové služby, diagnostické rozhraní a pomocné workflow, které nebyly navržené proti nedůvěryhodnému lokálnímu volajícímu.
- Úspěšná post-exploitation práce stojí na pochopení vztahu mezi těmito službami, ne na slepém skenování všech portů.
- Obrana musí počítat s tím, že loopback po kompromitaci hostu přestává být bezpečnostní hranicí.

## Co si odnést do praxe

- Po footholdu má localhost stejnou prioritu jako veřejná attack surface. Často právě tam leží druhá polovina útokového řetězce.
- Při enumeraci lokálních služeb se ptejte, kdo je volá, čemu důvěřují a co se stane, když je začne používat kompromitovaný uživatel nebo proces.
- Interní služby mají být bezpečné i proti lokálnímu volajícímu. Pokud spoléhají jen na to, že "nejsou venku", jde jen o odložený problém, ne o skutečnou ochranu.
