---
layout: post
author: Tomáš Matějíček
title: "Síťová topologie jako leak: WPAD, Squid, FXP a interní mapování"
date: 2020-10-29
tags: network recon ipv6 wpad squid fxp infrastructure
---
## Úvod a kontext

Některé útoky nezačínají zranitelností v aplikaci, ale tím, že prostředí samo prozradí, jak je poskládané. Proxy konfigurace, WPAD, interní IPv6 adresa nebo FXP odpověď z FTP serveru nevypadají jako plnohodnotný foothold. Často ale dají útočníkovi něco stejně cenného: mapu, kudy pokračovat do neveřejných služeb.

To je důležité odlišit od dvou blízkých témat. Nejde tu primárně o DNS/WHOIS jako veřejný informační kanál a nejde ani o port forwarding jako techniku, jak už nalezenou službu zpřístupnit. Tady je hlavní otázka jiná: jaké infrastrukturní detaily samotné prostředí neúmyslně prozradí a jak z nich vznikne další cíl.

## Co znamená „topologie jako leak“

Jde o situaci, kdy prostředí neprozradí přímo heslo nebo shell, ale prozradí:

- že existuje další interní rozsah,
- že je ve hře IPv6, které obrana nehlídá stejně jako IPv4,
- že klienti používají proxy nebo WPAD a kam přesně ukazují,
- že služba otevře datové spojení ze skryté interní adresy,
- že konkrétní hostname nebo management vrstva žije jinde, než vypadá z veřejného skenu.

Taková informace sama o sobě ještě není kompromitace. Ale velmi často je to právě ten bridge krok, bez kterého by další zranitelná služba zůstala neviditelná.

## Příklad z praxe: APT a IPv6 jako druhý obraz infrastruktury

APT je výborná ukázka toho, že IPv4 sken může vrátit úplně jiný obraz prostředí než reálná infrastruktura. Přes IPv4 byly zvenku skoro vidět jen web a RPC. Teprve zjištění IPv6 adresy otevřelo úplně jinou realitu:

- DNS,
- Kerberos,
- LDAP,
- SMB,
- WinRM,
- tedy plnou AD attack surface.

Tady je důležité pochopit, co vlastně uniklo. Nešlo o jedno „tajemství“. Unikla existence celé druhé síťové roviny, kterou obrana očividně nesledovala stejně přísně jako IPv4.

Z bezpečnostního hlediska to znamená, že otázka nemá znít jen `je služba vystavená`, ale i `na které adrese a v které síťové vrstvě ji vlastně vidíme`.

## Příklad z praxe: Tentacle a proxy jako mapa interních rozsahů

Tentacle ukazuje jiný vzor. Veřejně dostupný Squid nebyl konečný cíl, ale vstup do dalšího mapování. Přes proxy šlo osahat interní jména a stáhnout `wpad.dat`, který přímo prozradil další rozsah:

```javascript
if (isInNet(dnsResolve(host), "10.241.251.0", "255.255.255.0"))
    return "DIRECT";
```

To je prakticky síťová dokumentace. Útočník najednou neháda, kde hledat další hosty. Prostředí mu samo řekne, že vedle už známého segmentu existuje ještě další interní síť. Právě tam pak ležel OpenSMTPD host, který vedl k dalšímu footholdu.

Tentacle je užitečný právě tím, že topologický leak nepřichází přes chybnou DNS odpověď, ale přes pomocný provozní soubor určený pro klienty.

## Příklad z praxe: Zetta a FXP jako leak interní adresy

Zetta stojí na Pure-FTPd a jeho FXP/EPRT chování. Tady není pointa v tom, že by FTP samo dalo shell. Pointa je v tom, že datové spojení otevřené serverem prozradilo interní IPv6 adresu:

```text
dead:beef::250:56ff:feb9:f660
```

To je velmi silný topologický leak:

- útočník zjistí, že host je dosažitelný i v interní IPv6 vrstvě,
- získá konkrétní adresu,
- a následně na této vrstvě najde další službu, která z veřejné strany vidět nebyla.

Na Zettě to byla interní rsync služba. Bez úniku IPv6 adresy by další řetězec vůbec nezačal.

## Jaké typy leaků se opakují nejčastěji

### 1. Alternativní adresní rovina

APT i Zetta ukazují, že nejde jen o „nějakou IPv6 adresu“. Jde o druhou realitu infrastruktury, kde bývají viditelné jiné služby než na IPv4.

### 2. Proxy a klientská síťová konfigurace

Tentacle ukazuje, že:

- Squid,
- WPAD,
- proxy názvy,
- pravidla `DIRECT` a `PROXY`

nejsou jen pomocná klientská logika. Jsou to často velmi přesné instrukce o vnitřní topologii.

### 3. Datové protokoly s bočním únikem adresy

FXP/EPRT u FTP je hezký příklad toho, že i protokol, který nevydá shell, může prozradit, z jaké adresy server skutečně komunikuje. To je cenné hlavně tehdy, když se veřejná a interní síťová vrstva liší.

## Na co se při analýze zaměřit

U podobných prostředí má smysl jít po několika konkrétních otázkách:

### Existuje jiná síťová vrstva než ta, kterou vidím při prvním skenu

- IPv6,
- další segment za proxy,
- management síť,
- interní hostname ukazující mimo veřejný rozsah.

### Prozrazuje prostředí vlastní síťovou dokumentaci

- `wpad.dat`,
- proxy konfigurace,
- helper skripty,
- interní názvy v certifikátech nebo konfiguračních souborech,
- odpovědi protokolů, které vracejí zdrojovou adresu.

### Dá se z leaku odvodit konkrétní další krok

Tohle je nejdůležitější filtr. Ne každý infrastrukturní detail je relevantní. Praktickou hodnotu má ten, který vede k:

- novému skenu,
- novému hostname,
- nové adrese,
- nebo jiné síťové vrstvě, kde běží další služba.

## Rozdíl proti jiným článkům

Je užitečné to držet odděleně od dalších témat v backlogu.

### Není to článek o WHOIS/DNS/AXFR

Tam je hlavní téma veřejný jmenný prostor a datový únik z registrační nebo DNS vrstvy. Tady je hlavní téma síťová topologie a boční kanály, které ji prozradí.

### Není to článek o port forwardingu

Port forwarding řeší, jak už nalezenou interní službu zpřístupnit. Tento článek řeší, jak se vůbec dozvíš, že nějaká interní služba nebo segment existuje.

### Není to článek jen o IPv6

IPv6 je jen jedna z variant. Stejně důležitý může být WPAD, Squid nebo FXP.

## Obrana a hardening

### 1. Inventarizovat všechny adresní roviny

Pokud bezpečnostní monitoring a asset inventory vidí jen IPv4, prostředí ve skutečnosti nezná. APT i Zetta ukazují, že právě na „druhé“ síťové vrstvě může ležet nejcitlivější část infrastruktury.

### 2. Revidovat proxy a WPAD jako citlivý konfigurační kanál

Soubor určený klientům nemá fungovat jako dokumentace vnitřních segmentů. To platí jak pro WPAD, tak pro další helper konfigurace.

### 3. Omezit protokoly, které prozrazují interní adresaci

FXP/EPRT, špatně navržené proxy vrstvy nebo podobné mechanismy mohou prozradit víc, než by správce čekal. Je potřeba je hodnotit nejen podle funkčnosti, ale i podle toho, co vyzradí o topologii.

### 4. Hodnotit každý leak podle navazující služby

Samotná adresa nebo rozsah nemusí vypadat nebezpečně. Důležité je, co na něm běží a jak moc se liší od veřejně dostupného obrazu infrastruktury.

### 5. Neoddělovat síťový design od bezpečnostního review

Topologické leaky často vznikají v provozních vrstvách, ne v aplikacích. Bezpečnostní review proto musí zahrnovat i proxy logiku, klientské síťové konfigurace a alternativní transportní roviny.

## Shrnutí klíčových poznatků

- Síťová topologie je sama o sobě citlivá informace, pokud z ní útočník odvodí další skrytou službu nebo adresní rovinu.
- APT ukazuje, že jiná síťová vrstva může odhalit úplně jinou attack surface než první veřejný sken.
- Tentacle ukazuje, že proxy a WPAD mohou fungovat jako neúmyslná síťová dokumentace.
- Zetta ukazuje, že i FTP může přes FXP/EPRT prozradit interní adresu a tím otevřít cestu k dalším službám.

## Co si odnést do praxe

- Když narazíš na proxy, WPAD, FXP nebo zvláštní adresaci, neber to jen jako síťařský detail. Ptej se, jakou další vrstvu prostředí ti to právě odhalilo.
- Obrana nemůže chránit jen to, co vidí na první veřejné adrese. Musí chránit i to, co je dostupné přes proxy, alternativní adresní rodinu nebo vedlejší transportní logiku.
- Topologický leak není jen recon. Ve chvíli, kdy navede útočníka na nový segment nebo službu, stává se samostatnou součástí kompromitačního řetězce.
