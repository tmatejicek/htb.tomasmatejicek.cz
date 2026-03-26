---
layout: post
author: Tomáš Matějíček
title: "Metadata, logy, incidentní a forenzní artefakty jako zdroj přístupů"
date: 2020-11-23
tags: credentials logs metadata forensics post-exploitation
---
## Úvod a kontext

Článek o logách jako útokové ploše řeší situaci, kdy se z logu stane vstup do privilegovaného zpracování a následně i cesta k RCE nebo privescu. Tady jde o jiný problém: logy, metadata, pcapy, trace soubory, mailboxy nebo exporty konfigurací jako zdroj tajemství a provozního kontextu.

To je prakticky velmi častý pattern. Útočník nepotřebuje další exploit, pokud v záloze, logu nebo incidentním balíčku leží heslo, interní URL, jméno účtu, přesný název administrační stránky nebo stopa po chybném přihlášení. Provozní artefakt se v tu chvíli mění v credential source.

## Nejde o „neškodná provozní data“

Bezpečnostní problém obvykle nevzniká tím, že by log nebo metadata samy vykonávaly kód. Problém je v tom, že obsahují informace, které jinde chráníš jako tajemství:

- přihlašovací údaje zadané do špatného pole,
- URL resetů hesel a administrativních rozhraní,
- exporty síťových zařízení s dekódovatelnými hesly,
- incidentní pcap s celou přihlašovací relací,
- onboarding dokument s dočasným účtem,
- recent-file metadata ukazující na zajímavý soubor nebo skryté sdílení.

Rozdíl proti běžné konfigurační chybě je v tom, že správce tyto soubory často ani nepovažuje za citlivé. Přitom právě v nich bývá nejkratší cesta k dalšímu účtu.

## Typické kategorie artefaktů

### 1. Trace a aplikační logy

Remote ukazuje velmi čistý příklad. V záloze ležel nejen databázový soubor Umbraco, ale i `UmbracoTraceLog.intranet.txt`. Právě v něm se objevil neúspěšný login, kde bylo heslo omylem zadané do pole pro uživatelské jméno:

```text
Login attempt failed for username Umbracoadmin123!! ...
```

To je typ situace, kdy log neprozradí „jen provoz“, ale rovnou vysoce pravděpodobné aktuální heslo administrátora.

Podobně na Doctoru zůstal v Apache logu request, ve kterém se hodnota `Guitar123` propsala do URL. Bezpečnostní problém tady není v Apache jako takovém. Je v tom, že provozní log převzal citlivou hodnotu a uchoval ji v podobě čitelné dalšímu účtu.

### 2. Incidentní a forenzní balíčky

Na Scavengeru vedla cesta přes poštovní schránky k FTP přístupu a následně k souboru `ib01c01_incident.pcap`. Ten pak vydal administrační URL i přihlašovací údaje pro další službu.

To je velmi realistický scénář. Incidentní artefakt často obsahuje přesně to, co potřebuje člověk při analýze incidentu:

- zachycený provoz,
- URL, na která se šlo,
- použité účty,
- chybové hlášky,
- někdy i session nebo celé HTTP požadavky.

Bez dobré izolace se z takového balíčku stává koncentrovaný přehled cizích tajemství.

### 3. Exporty a zálohy zařízení

Heist nestál na webovém exploitu, ale na Cisco konfiguraci. Z ní šlo vytáhnout několik hesel a následně je otestovat proti doménovým účtům přes password spray. To je důležitá lekce: konfigurace síťového prvku není „jen konfigurace zařízení“. V praxi velmi často odhalí reuse hesla pro helpdesk, VPN nebo Windows účet.

Právě exporty zařízení bývají podceněné. Obsahují:

- dekódovatelná hesla,
- jména účtů,
- interní adresaci,
- názvy VPN skupin,
- někdy i tajemství k dalším systémům.

### 4. Metadata a vedlejší stopy v běžných nástrojích

Nest ukazuje jiný druh úniku. Nešlo o klasický log, ale o metadata z běžných nástrojů:

- onboarding dokument s účtem `TempUser / welcome2019`,
- `Notepad++` konfiguraci ukazující na `\\HTB-NEST\Secure$\IT\Carl\Temp.txt`,
- další XML konfigurace s uloženým šifrovaným heslem.

To je přesně ten typ stop, který se snadno přehlédne, protože nevypadá jako „tajemství“. Ve skutečnosti ale prozradí, kam se dívat dál, jak se jmenují účty a kde leží další relevantní soubory.

## Jak o těchto artefaktech přemýšlet po footholdu

Užitečnější než slepé hledání konkrétních přípon je položit si několik praktických otázek:

### Který soubor vznikl proto, aby usnadnil provozní práci

Sem typicky patří:

- trace logy,
- reset logy,
- mailboxy,
- pcapy,
- exporty konfigurací,
- onboarding a support dokumentace,
- recent-file seznamy a editor konfigurace.

Právě tyto soubory často obsahují zkratky, které si administrátor nebo support tým vytváří pro vlastní pohodlí.

### Který artefakt zachycuje cizí vstup

Rizikové jsou hlavně soubory, které uchovávají:

- autentizační pokusy,
- URL s parametry,
- požadavky zákazníků,
- dump komunikace,
- e-mailové přílohy z incidentu.

Jakmile se do nich dostane cizí vstup, je velká šance, že v něm zůstane i tajemství nebo provozní kontext.

### Který artefakt propojuje dvě vrstvy prostředí

Nejcennější bývají soubory, které propojí něco, co se jinak zdá oddělené:

- zařízení a doménové účty,
- helpdesk a WinRM,
- webovou aplikaci a interní admin URL,
- mailbox a FTP,
- zálohu CMS a lokální administraci hostu.

Právě tato vazba dělá z provozního artefaktu skutečný bridge krok.

## Na co se při analýze zaměřit

Po prvním footholdu má smysl systematicky projít hlavně:

- backup adresáře a exporty webů,
- `trace`, `access`, `error` a `history` soubory,
- mail spool, OST/PST archivy a incidentní mailboxy,
- pcapy, forenzní ZIPy a support balíčky,
- dokumenty pro onboarding nebo handover,
- konfigurace editorů a nástrojů s recent-file historií,
- exporty síťových zařízení a interních appliance.

Nejde o to sbírat vše. Jde o to rozlišit, který artefakt nese provozní tajemství nebo ukazuje na další důležitý krok.

## Obrana a hardening

### 1. Chránit artefakty stejně jako primární tajemství

Pokud backup, trace log nebo incidentní pcap obsahuje heslo, URL administrace nebo session detail, musí mít stejnou úroveň ochrany jako původní účet nebo služba.

### 2. Nenechávat tajemství v URL ani v logovaných polích

Doctor i Remote ukazují stejný problém z různých stran: citlivá hodnota se dostala do vrstvy, která se automaticky loguje. Jakmile se jednou zapíše do URL, trace logu nebo chybných login záznamů, obvykle se začne šířit dál.

### 3. Omezit šíři incidentních balíčků

Pcap, support ZIP nebo export z appliance by měl obsahovat jen to, co je skutečně potřeba. Čím víc provozních detailů se archivuje „pro jistotu“, tím větší je dopad pozdějšího úniku.

### 4. Oddělit přístupy k support a provozním artefaktům

To, že někdo potřebuje číst dokumentaci nebo vyřešit incident, ještě neznamená, že má mít přístup i k archivům obsahujícím cizí hesla, síťové konfigurace nebo plné trace logy.

### 5. Počítat s tím, že metadata jsou také data

Názvy souborů, recent-file seznamy, e-mailové přílohy, historii editoru nebo staré onboarding dokumenty je potřeba auditovat stejně jako hlavní konfigurační soubory. Útočník často nepotřebuje tajemství v plaintextu. Stačí mu stopa, která ho dovede k dalšímu účtu.

## Shrnutí klíčových poznatků

- Logy, metadata, pcapy a provozní exporty často neútočí samy, ale velmi často vydají přesně ta tajemství, která útok posunou dál.
- Největší riziko nevzniká u „hlavních“ konfigurací, ale u vedlejších artefaktů, které správci nepovažují za citlivé.
- Prakticky nejcennější jsou artefakty, které propojí dvě vrstvy prostředí: web a admin rozhraní, zařízení a doménu, mailbox a FTP, backup a lokální účet.
- Obrana není jen o mazání logů, ale hlavně o tom, aby se citlivé údaje do provozních artefaktů vůbec nedostávaly.

## Co si odnést do praxe

- Po footholdu nehledáš jen konfiguraci. Hledej i stopu po provozu: trace log, mailbox, pcap, support ZIP nebo onboarding dokument.
- Každý artefakt si zkus zařadit do jedné otázky: obsahuje tajemství, nebo ukazuje na místo, kde tajemství leží?
- Pokud se citlivé údaje zapisují do logů, exportů nebo incidentních balíčků, problém není v jednom souboru. Problém je v celé provozní disciplíně kolem nich.
