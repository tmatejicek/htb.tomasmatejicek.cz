---
layout: post
author: Tomáš Matějíček
title: "IPv6 jako opomíjená attack surface"
date: 2020-10-29
tags: ipv6 network infrastructure monitoring active-directory
---
## Úvod a kontext

IPv6 se v mnoha prostředích pořád bere jako něco vedlejšího. Síť „se přece dělá na IPv4“, monitoring běží nad IPv4, asset inventory zná IPv4 adresy a bezpečnostní pravidla se obvykle ověřují na IPv4 cestě. Jenže tím vzniká nebezpečný omyl:

- `nevidím službu na IPv4`

není totéž jako

- `služba není dostupná`.

APT a Zetta ukazují, že IPv6 nemusí být dekorace nebo síťařská kuriozita. Může to být druhá plnohodnotná attack surface s úplně jiným obrazem infrastruktury než ten, který vrací první veřejný sken. A právě to je potřeba chápat správně: IPv6 není zajímavé kvůli formátu adresy, ale kvůli tomu, že obrana často hlídá jinou síťovou realitu než útočník.

## Jaký je rozdíl proti „topologickému leaku“

Je užitečné tenhle článek držet odděleně od textu o [Síťové topologii jako leaku: WPAD, Squid, FXP a interní mapování](/techniky/sitova-topologie-jako-leak-wpad-squid-fxp-a-interni-mapovani).

Tam je hlavní otázka:

- jak prostředí prozradí další segment nebo skrytou cestu.

Tady je hlavní otázka:

- co se stane, když ten další obraz prostředí existuje přímo jako samostatná adresní rovina a obrana ho neřídí stejně jako IPv4.

Jinými slovy: topologický leak může na IPv6 teprve navést. Tenhle článek řeší, proč je samotná existence nehlídané IPv6 vrstvy tak silný problém.

## Proč IPv6 mění reálnou attack surface

Z bezpečnostního pohledu nejsou nejdůležitější syntaktické rozdíly v adrese. Důležité jsou provozní důsledky:

- služby mohou být dostupné jen na IPv6,
- firewall pravidla nemusí být rovnocenná mezi IPv4 a IPv6,
- monitoring a inventura často vidí jen IPv4,
- názvy v DNS mohou vracet AAAA záznamy, které nikdo aktivně neauditoval,
- a interní nebo management vrstva může na IPv6 žít samostatněji, než si tým uvědomuje.

Proto má smysl o IPv6 uvažovat jako o samostatné attack surface, ne jako o dodatku k existující IPv4 síti.

## APT: IPv4 skoro nic, IPv6 plná doména

[APT](/apt) je nejčistší případ. Přes IPv4 bylo zvenku vidět v zásadě jen minimum:

```text
80/tcp  open  http
135/tcp open  msrpc
```

Kdyby analýza skončila tady, působilo by prostředí poměrně chudě. Teprve zjištění IPv6 adresy a nový scan ukázaly úplně jiný obraz:

```text
53/tcp   open  domain
88/tcp   open  kerberos-sec
389/tcp  open  ldap
445/tcp  open  microsoft-ds
5985/tcp open  http
```

To je zásadní posun. Najednou už nejde o web a RPC. Jde o plnohodnotný Active Directory povrch:

- DNS,
- Kerberos,
- LDAP,
- SMB,
- WinRM.

Právě na IPv6 pak ležel i anonymně dostupný SMB share se `backup.zip`, který otevřel další celý kompromitační řetězec přes `ntds.dit`, `SYSTEM`, `henry.vinson_adm` a později `APT$`.

APT je proto výborná připomínka, že obrana může vidět „chudý“ IPv4 host, zatímco útočník po přechodu na IPv6 najde kompletní doménovou infrastrukturu.

## Zetta: interní IPv6 jako most k další službě

[Zetta](/zetta) ukazuje jiný vzor. Tady IPv6 nefunguje jako okamžitě veřejně zjevná druhá realita. Nejdřív bylo potřeba získat interní IPv6 adresu přes chování Pure-FTPd a FXP/EPRT.

Jakmile se ale adresa objevila:

```text
dead:beef::250:56ff:feb9:f660
```

otevřela se další služba, která na prvním veřejném pohledu vůbec nebyla vidět:

```text
8730/tcp open  rsync
```

Tady je důležité správně interpretovat roli IPv6. Nejde jen o to, že „unikla nějaká adresa“. Jde o to, že interní služba žila v adresní rovině, kterou první veřejná enumerace nepokrývala. IPv6 tu tedy funguje jako samostatná provozní vrstva s vlastním povrchem i s jinou sadou chyb.

## Kde do toho zapadá Tentacle

[Tentacle](/tentacle) není čistě IPv6 příklad v tom samém smyslu jako APT nebo Zetta. Je ale dobrý kontrast.

Na Tentacle se ukazuje stejný obranný omyl: tým si myslí, že vidí síť tak, jak je skutečně dostupná, ale ve skutečnosti existuje druhá rovina dosažitelnosti přes proxy, WPAD a interní segmenty. U IPv6 je tenhle problém ještě zrádnější, protože druhá rovina není jen „za proxy“. Je to samostatná adresní rodina, kterou část nástrojů a procesů vůbec nevidí.

Proto je Tentacle užitečný jako připomínka principu, i když samotný hlavní argument článku stojí především na APT a Zettě.

## Typické obranné slepé skvrny kolem IPv6

### 1. Asset inventory zná jen IPv4

Host je evidovaný podle jedné adresy, ale reálně odpovídá i na AAAA záznamu nebo interním IPv6 prostoru. Tím vzniká rozdíl mezi tím, co je „v evidenci“, a tím, co je skutečně dosažitelné.

### 2. Firewall a ACL nejsou rovnocenné

Politika může být přísná na IPv4 a podstatně volnější na IPv6, protože se nastavovala později, jen částečně nebo vůbec.

### 3. Monitoring nesleduje stejná data

Logy, flow, IDS/IPS nebo alerting mohou mít pro IPv6 menší pokrytí. Výsledek není akademický. Znamená to, že útok běžící přes IPv6 má jinou pravděpodobnost odhalení než tentýž útok na IPv4.

### 4. Správci předpokládají, že „IPv6 se nepoužívá“

To je možná nejhorší varianta. Nepoužívané IPv6 se často neznamená vypnuté IPv6. Znamená jen neřízenou vrstvu, kterou nikdo aktivně neauditoval.

## Jak podobné situace hledat systematicky

Při analýze infrastruktury dává smysl ptát se:

### Vidím na IPv6 stejné služby jako na IPv4

Pokud ne, je to samo o sobě důležitý nález. Rozdíl v povrchu znamená rozdíl v útokové ploše.

### Existují AAAA záznamy nebo interní IPv6 adresy, které nejsou v běžné dokumentaci

U APT i Zetty byl rozhodující právě moment, kdy se druhá adresa objevila a ukázalo se, že nese jiný význam než hlavní veřejná IPv4.

### Jsou stejná pravidla i pro monitoring, filtering a přístup

Bez této parity vzniká situace, kdy je jedna a tatáž služba bráněná různě podle toho, přes kterou adresní rodinu na ni přijdeš.

## Obrana a hardening

### 1. Inventarizovat hosty v obou adresních rodinách

Bezpečnostní evidence nesmí pracovat s představou „tento server má adresu X“, pokud ve skutečnosti odpovídá na více IPv4/IPv6 identitách s různým povrchem.

### 2. Udržovat rovnocennou politiku pro IPv4 i IPv6

Firewall, segmentace, management přístupy i monitorovací pravidla mají být záměrně srovnané. Není bezpečné předpokládat, že co je zavřené na IPv4, bude automaticky zavřené i na IPv6.

### 3. Aktivně auditovat AAAA záznamy a interní IPv6 dosah

IPv6 se nesmí řešit až ve chvíli, kdy ji někdo náhodou najde při incidentu nebo pen testu. Musí být součástí běžné enumerace i interního review.

### 4. Pokud IPv6 opravdu nechceš používat, vypni ji řízeně

„Nepoužíváme ji“ není bezpečnostní politika. Buď je IPv6 řízená, monitorovaná a spravovaná, nebo ji je potřeba vědomě vypnout tam, kde pro ni není legitimní use case.

## Shrnutí klíčových poznatků

- IPv6 není jen jiný formát adresy. V praxi často představuje druhou attack surface s jiným povrchem než IPv4.
- APT ukazuje nejtvrdší variantu: přes IPv4 skoro nic, přes IPv6 plná Active Directory infrastruktura a kritický SMB share.
- Zetta ukazuje, že i interní nebo méně viditelná IPv6 vrstva může nést další službu, která z veřejné IPv4 perspektivy neexistuje.
- Obranný problém nevzniká jen ze samotné IPv6, ale hlavně z toho, že ji inventura, filtering a monitoring často nehlídají stejně jako IPv4.

## Co si odnést do praxe

- Když „na serveru nic nevidíš“, ověř si, jestli to platí i pro IPv6. Bez téhle kontroly nevíš, jaká je skutečná attack surface.
- Rozdíl mezi IPv4 a IPv6 povrchem je bezpečnostní nález sám o sobě, ne jen síťařská zvláštnost.
- Obrana, která aktivně řídí jen IPv4, ve skutečnosti neřídí infrastrukturu jako celek.
