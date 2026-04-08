---
layout: post
author: Tomáš Matějíček
title: "BloodHound"
date: 2020-11-26
tags: nastroje windows active-directory ldap graph acl
---
## Úvod a kontext

`BloodHound` je jeden z mála nástrojů, který v Active Directory opravdu mění způsob přemýšlení. Není to skener na hesla ani exploit framework. Je to grafová analýza vztahů mezi účty, skupinami, ACL, session a delegovanými oprávněními. Praktická hodnota proto neleží v tom, že "něco najde", ale v tom, že z množství nenápadných vazeb složí reálnou cestu k vyšším oprávněním.

Na [Forestu](/forest) byl přesně tohle rozhodující moment. Účet `svc-alfresco` sám o sobě nevypadal jako doménový admin, ale [BloodHound](/nastroje/bloodhound) ukázal cestu přes `Exchange Windows Permissions`, LDAP relay a DCSync. To je dobrý příklad toho, proč BloodHound stojí spíš na straně rozhodování než na straně samotného exploitu.

## Co tenhle nástroj v praxi řeší

V AD prostředí je velmi snadné uvíznout v jednoduché otázce: "Je ten účet admin, nebo není?" BloodHound pomáhá položit lepší otázku:

- jaké vztahy ten účet skutečně má,
- které skupiny nebo ACL umí zneužít,
- jaké privilege paths vedou k cíli,
- a kde je rozdíl mezi formální rolí a reálnou mocí v doméně.

To je prakticky zásadní, protože v reálných řetězcích často nerozhoduje přímé členství v `Domain Admins`, ale nenápadná kombinace několika kroků.

## Nejčastější scénáře využití v tomto projektu

### Hledání cesty z běžného footholdu k vyšší identitě

Na [Forestu](/forest) je dobře vidět základní pattern. První foothold vznikl přes AS-REP roast a WinRM, ale root/admin část už nestála na lokálním exploitu. Rozhodující bylo nasbírat vztahy v doméně pomocí `SharpHound` a otevřít je v BloodHoundu:

```powershell
Import-Module ./SharpHound.ps1
Invoke-BloodHound -CollectionMethod All -Verbose
```

Teprve graf ukázal, že účet vede k `Exchange Windows Permissions`, a až z toho dávalo smysl navázat PrivExchange a relay. BloodHound tu tedy neřeší poslední krok. Řeší otázku, zda ten krok vůbec stojí za zkoušení.

### Čtení ACL a delegovaných práv jako reálné útočné plochy

Praktická síla BloodHoundu je hlavně v tom, že zvýrazní vztahy, které se v běžné ruční enumeraci snadno ztratí:

- `GenericAll`,
- `WriteDacl`,
- členství ve skupinách s nepřímým dopadem,
- možnost relaye nebo DCSync přes delegovanou cestu,
- nebo session a lokální admin vztahy mezi hosty.

To je přesně kontext, který podrobněji rozebírám i v článku [Delegovaná práva v AD: DCSync, gMSA, relay, deleted objects](/techniky/delegovana-prava-v-ad-dcsync-gmsa-relay-deleted-objects). BloodHound není náhrada za tenhle typ přemýšlení. Je to nástroj, který ho výrazně urychlí.

### Ověření, jestli má smysl jít po konkrétním účtu

Po footholdu na AD hostu je lákavé zkoušet náhodné laterální pohyby nebo agresivní dumping. BloodHound pomáhá tenhle chaos omezit. Když ukáže, že daný účet nikam nevede, je lepší hledat další identitu. Když naopak ukáže krátkou cestu přes konkrétní skupinu nebo ACL, dává smysl soustředit energii tam.

Právě proto má BloodHound vysokou hodnotu po prvním shellu, ne před ním. Před footholdem ještě často chybí sběr dat. Po footholdu už naopak pomůže rozhodnout, co dál.

## Kdy je BloodHound nejlepší volba

BloodHound dává největší smysl tehdy, když:

- už existuje aspoň jeden doménový foothold,
- potřebujete zmapovat privilege paths místo náhodného zkoušení,
- v doméně čekáte delegovaná práva, Exchange vztahy nebo ACL zkratky,
- nebo chcete zkontrolovat, zda účet vede k DCSync, session abuse nebo jiné identitní eskalaci.

Naopak to není ideální nástroj pro úplně první nulovou enumeraci. Tu řeší spíš [Kerbrute](/nastroje/kerbrute), [ldapsearch](/nastroje/ldapsearch), [windapsearch](/nastroje/windapsearch) nebo [Impacket pro AD enumeraci a první identity](/nastroje/impacket-pro-ad-enumeraci-a-prvni-identity).

## Nejčastější chyby v praxi

### Zaměnění mapy za exploit

BloodHound nic sám neeskaluje. Jen ukáže cestu. Na [Forestu](/forest) pořád bylo potřeba udělat relay a DCSync. Bez dalších kroků zůstane i nejhezčí graf jen hypotézou.

### Slepá víra v "nejkratší cestu"

To, že BloodHound ukáže krátkou cestu, ještě neznamená, že je provozně nejspolehlivější. Někdy vede přes hlučný relay, jindy přes session, která už mezitím zmizela. Proto je potřeba graf číst jako rozhodovací pomůcku, ne jako absolutní pravdu.

### Příliš široký sběr bez ohledu na dopad

`CollectionMethod All` je praktický, ale ne vždy nutný. V citlivém prostředí může dávat větší smysl cílenější sběr podle toho, co právě potřebujete ověřit.

## Související články v projektu

- Rozbory: [Forest](/forest)
- Techniky: [Delegovaná práva v AD: DCSync, gMSA, relay, deleted objects](/techniky/delegovana-prava-v-ad-dcsync-gmsa-relay-deleted-objects), [NTLM coercion a vynucená autentizace](/techniky/ntlm-coercion-a-vynucena-autentizace), [Kerberos útoky z nulového přístupu](/techniky/kerberos-utoky-z-nuloveho-pristupu-as-rep-roast-a-prace-s-kandidatnimi-uzivateli)
- Nástroje: [Kerbrute](/nastroje/kerbrute), [ldapsearch](/nastroje/ldapsearch), [windapsearch](/nastroje/windapsearch)

## Co si odnést do praxe

- BloodHound je nejcennější jako mapa privilege paths, ne jako nástroj na samotný exploit.
- Největší hodnotu má po prvním doménovém footholdu, kdy pomůže rozhodnout, která identitní cesta stojí za další práci.
- V AD review je užitečný hlavně proto, že ukáže nepřímou moc účtů a skupin, která z pouhého seznamu členství nemusí být vidět.
