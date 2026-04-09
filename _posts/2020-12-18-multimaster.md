---
layout: post
author: Tomáš Matějíček
title: "Multimaster"
date: 2020-12-18
tags: windows exploit privesc enumeration hackthebox
---
## Úvod a kontext

Multimaster je doménový stroj, kde se foothold neudělá jedním trikem, ale čistým řetězením více slabin. Začíná SQL injection ve veřejné webové aplikaci, pokračuje přes zneužití debug funkce ve VS Code, potom přes reuse hesla mezi účty a nakonec končí delegovanými právy v Active Directory.

To je na tomhle boxu nejcennější. Každý krok sám o sobě vypadá omezeně, ale dohromady vytvoří plnohodnotný laterální pohyb: `web -> cyork -> sbauer -> jorden -> SYSTEM`. Prakticky tak propojuje články [Password reuse a rozpad hranic mezi aplikací, SSH, WinRM a admin nástroji](/techniky/password-reuse-a-rozpad-hranic-mezi-aplikaci-ssh-winrm-a-admin-nastroji) a [Delegovaná práva v AD: DCSync, gMSA, relay, deleted objects](/techniky/delegovana-prava-v-ad-dcsync-gmsa-relay-deleted-objects).

## Počáteční průzkum

### Web jako první realistický vstup

Veřejně dostupná aplikace byla na Multimaster hlavním vstupním bodem do prostředí. Důležité nebylo jen potvrdit, že web běží, ale pochopit, že pracuje s databází a že jeho chyby mohou dát příkazový kontext přímo na serveru.

Na tomto stroji proto dává smysl soustředit se od začátku na to, jak webová vrstva sahá do backendu a jaké lokální procesy běží na hostu po získání prvního shellu.

## Analýza zjištění

### SQL injection jako foothold, ne jako cíl

První posun přinesla SQL injection. Na Multimaster ale nesloužila hlavně k dumpu databáze, nýbrž k získání prvního příkazového kontextu na samotném serveru. To je důležitý rozdíl: z databázové chyby se velmi rychle stal systémový foothold.

Lokální enumerace potom ukázala, že na hostu běží VS Code a že jsou aktivní i související procesy. To otevřelo další pivot: zneužití debug funkce k přechodu na účet `cyork`.

### Heslo v DLL a přechod na `sbauer`

Jakmile je k dispozici shell jako `cyork`, další fáze už není o nové zranitelnosti, ale o hledání tajemství v souborech a binárkách. Na Multimaster se rozhodující stopa objevila v DLL, která obsahovala heslo znovu použité pro účet `sbauer`. Je to přesně ten typ mezikroku, který shrnuji i v článku [Reverzní inženýrství klienta nebo vlastní binárky jako součást běžného průniku](/techniky/reverzni-inzenyrstvi-klienta-nebo-vlastni-binarky-jako-soucast-bezneho-pruniku).

Právě tento moment je na stroji velmi realistický. Mnoho prostředí je dnes odolnější vůči přímému exploitu, ale stále padá na uložená tajemství v klientských binárkách, konfiguracích a pomocných knihovnách.

## Získání přístupu

### Stabilní přístup jako `sbauer`

Účet `sbauer` je na Multimaster skutečný zlom. Do té doby jde hlavně o foothold a lokální pivot. Až `sbauer` dává dostatečný doménový kontext pro další práci s Active Directory a jejími delegovanými právy.

Tím se celý řetězec přesune z lokální Windows enumerace do AD logiky. Další krok už není hledání dalšího hesla na disku, ale pochopení toho, kdo nad kým drží oprávnění.

## Eskalace oprávnění

### `GenericWrite` nad `jorden` a cesta k `SYSTEM`

Rozhodující zjištění bylo, že `sbauer` má právo `GenericWrite` nad účtem `jorden`. To je v AD velmi silné oprávnění, protože dovoluje měnit vybrané atributy cílového účtu. V praxi to znamená, že lze vypnout Kerberos pre-auth, získat AS-REP odpověď a hash potom lámat offline. Praktickou roli takových cest dobře ukáže [BloodHound](/nastroje/bloodhound) a širší význam podobných delegací rozebírám i v článku [Delegovaná práva v AD: DCSync, gMSA, relay, deleted objects](/techniky/delegovana-prava-v-ad-dcsync-gmsa-relay-deleted-objects).

Po získání přístupu jako `jorden` už se situace znovu mění. `jorden` je členem `Server Operators`, takže není potřeba další exploit. Stačí zneužít oprávnění k úpravě nebo spuštění služby tak, aby běžela pod `SYSTEM` a provedla požadovaný příkaz.

Multimaster tak hezky ukazuje dvě různé privilegované roviny Windows prostředí: doménová delegovaná práva a lokální servisní oprávnění. Konečný `SYSTEM` shell vznikne teprve jejich kombinací.

## Shrnutí klíčových poznatků

- SQL injection na Multimaster nebyla cílem sama o sobě. Sloužila jen jako první most k lokálnímu kódu na serveru.
- VS Code debug funkce a heslo uložené v DLL ukázaly, že po footholdu často rozhodují vývojářské artefakty a tajemství zanechaná na hostu.
- Finální eskalace nebyla kernelová. Stála na `GenericWrite` nad účtem `jorden`, AS-REP roastu a následném zneužití členství ve `Server Operators`.

## Co si odnést do praxe

- Webové aplikace s přímým napojením na databázi musí být chráněné nejen proti úniku dat, ale i proti eskalaci do systémového kontextu. SQL injection často nekončí u databázových řádků.
- Vývojářské nástroje jako VS Code a jejich debug rozhraní nemají co běžet na produkčním serveru bez přísných omezení. Na kompromitovaném hostu se z nich snadno stává pivot do dalších účtů.
- Delegovaná oprávnění v AD je potřeba auditovat stejně pečlivě jako členství v privilegovaných skupinách. `GenericWrite` nad uživatelem nebo členství ve `Server Operators` bývá z obranného pohledu téměř stejně nebezpečné jako lokální admin.
