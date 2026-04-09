---
layout: post
author: Tomáš Matějíček
title: "Dokumentová metadata jako vstup do domény"
date: 2020-12-05
tags: active-directory metadata documents osint kerberos
---
## Úvod a kontext

Metadata dokumentů se často berou jako drobný OSINT detail. Typický závěr bývá: "prozradí autora dokumentu, nic víc." V prostředí Active Directory to ale může být mnohem silnější. Jakmile veřejné dokumenty vydají:

- jména zaměstnanců,
- naming convention účtů,
- historické dokumenty,
- nebo provozní instrukce typu výchozího hesla,

nejde už o kosmetický leak. Jde o první krok k validním uživatelům, password sprayingu, AS-REP roastu nebo dalším identitním útokům.

[Intelligence](/intelligence) je velmi čistý příklad právě proto, že celý řetězec začíná nevinně: web server publikuje PDF soubory. Teprve při systematickém čtení metadat se ukáže, že dokumenty vydávají jmenný prostor domény i reálně použitelné credential clue.

## Co všechno metadata typicky prozradí

Při bezpečnostním pohledu na dokumenty je užitečné nemyslet jen na pole `Author`. Prakticky hodnotné bývají hlavně:

- `Creator`,
- `Author`,
- názvy souborů a jejich datové vzory,
- názvy tiskových front a exportních šablon,
- datum vytvoření a úpravy,
- interní názvy oddělení nebo projektů,
- a někdy i samotný text dokumentu, pokud obsahuje onboarding instrukce nebo dočasná hesla.

Samotná metadata málokdy dají shell. Jejich hodnota spočívá v tom, že zpřesní identitní model cíle. A u AD prostředí je to často přesně to, co útočník potřebuje nejvíc.

## Intelligence: od `/documents/` k validním uživatelům

Na [Intelligence](/intelligence) hraje hlavní roli kombinace tří detailů:

1. veřejně dostupný adresář `/documents/`,
2. předvídatelné názvy souborů podle data,
3. metadata PDF souborů.

To je důležitý moment. Web nemusí mít index listing všech dokumentů. Jakmile jsou názvy předvídatelné, lze systematicky generovat URL a celý archiv si stáhnout:

```text
/documents/2020-01-01-upload.pdf
/documents/2020-12-15-upload.pdf
```

Potom přichází samotná těžba metadat:

```text
exiftool *.pdf
Creator : William.Lee
Creator : Jose.Williams
```

Tím se z "veřejných PDF" stává seznam kandidátních uživatelů. V AD prostředí to má okamžitou hodnotu, protože jména lze ověřit proti doméně:

```text
kerbrute userenum --dc intelligence.htb -d intelligence.htb users.txt
=> VALID USERNAME: William.Lee@intelligence.htb
=> VALID USERNAME: Jose.Williams@intelligence.htb
```

To je první důležitá lekce: metadata nejsou cenná sama o sobě. Jsou cenná proto, že snižují cenu další identitní enumerace.

## Dokument jako zdroj naming convention

Na Intelligence není nejdůležitější jedno jméno. Nejdůležitější je vzor. Jakmile dokumenty vrací jména typu `William.Lee` a `Jose.Williams`, lze odhadnout:

- formát uživatelských jmen,
- pravděpodobný tvar e-mailových adres,
- a hranice mezi interním jménem osoby a doménovým účtem.

To je důležitější než jednorázový leak, protože právě naming convention umožní:

- systematicky generovat další kandidáty,
- rozšiřovat user list,
- nebo cílit velmi úzký password spray místo hlučného zkoušení naslepo.

## Historické dokumenty jako nosič slabších tajemství

Na Intelligence celý příběh nekončí u `Creator` metadat. Jeden z historických dokumentů, `2020-06-04-upload.pdf`, obsahuje výchozí heslo `NewIntelligenceCorpUser9876`. To je přesně ten typ provozního detailu, který by v interním onboarding dokumentu mohl dávat krátkodobě smysl, ale veřejně je katastrofální.

Tady je důležité správně chápat roli dokumentu:

- metadata dala jmenný prostor,
- dokument sám dal credential clue,
- a dohromady z toho vznikla použitelná kombinace proti celé sadě validních uživatelů.

Výsledek:

Prakticky k tomu, kdy je `CrackMapExec` nejlepší jen jako rychlé potvrzení dopadu, viz i [CrackMapExec](/nastroje/crackmapexec).

```text
crackmapexec smb intelligence.htb -u users.txt -p NewIntelligenceCorpUser9876
=> intelligence.htb\Tiffany.Molina:NewIntelligenceCorpUser9876
```

Právě tím se metadata mění z pasivního leaku v první reálný vstup do domény.

## Proč to v AD prostředí funguje tak dobře

V jiném typu aplikace může `Author` znamenat jen jméno zaměstnance. V Active Directory ale téměř okamžitě vznikají další návazné kroky:

- ověření existence účtu,
- AS-REP roast nebo jiné kerberové testy,
- opatrný password spray,
- hledání výchozích hesel,
- nebo zacílení dalších veřejných dokumentů na konkrétní oddělení a uživatele.

To je důvod, proč metadata v doménovém prostředí bývají mnohem nebezpečnější než v běžném marketingovém webu. Nejsou jen vizitkou autora. Jsou stavebním materiálem pro identitní útok.

## Jak podobné leaky hledat systematicky

Při auditu veřejných dokumentů dává smysl jít po tomto sledu:

### 1. Najít samotný archiv dokumentů

- veřejný adresář,
- predictable názvy,
- sitemapa,
- odkazy v intranet-like stránkách,
- nebo generované exporty.

### 2. Vyčíst metadata napříč sadou souborů

Ne jednotlivě, ale hromadně. Hodnota roste až s množstvím:

- opakující se jména,
- oddělení,
- datové vzory,
- naming convention.

### 3. Zkusit z metadat sestavit user list

Tady už nejde o OSINT pro hezkou poznámku. Jde o pracovní podklad pro další bezpečnostní test.

### 4. Projít obsah historických a onboarding dokumentů

Zvlášť nebezpečné bývají:

- dočasná hesla,
- instrukce pro nové zaměstnance,
- interní URL,
- a pomocné postupy typu "první přihlášení použijte ...".

## Rozdíl proti čistému OSINT

Je užitečné držet hranici mezi běžným OSINT a tímto tématem.

- Běžný OSINT říká: "našel jsem jména lidí ve firmě".
- Tento článek řeší: "veřejné dokumenty dávají strukturovaná metadata a provozní kontext, který se dá přímo převést na validní AD uživatele a další přístupový pokus".

Jinými slovy: nejde o sběr personálních detailů. Jde o to, že dokument sám poskytne dost struktury na útok proti identitní vrstvě.

## Obrana a hardening

### 1. Redigovat metadata před publikací

Veřejný dokument nemá nést interní `Creator`, uživatelské jméno ani název stanice, pokud to není nutné. Metadata je potřeba čistit stejně jako samotný obsah.

### 2. Publikovat jen to, co je opravdu potřeba

Veřejně přístupný archiv historických dokumentů s předvídatelnými názvy je z bezpečnostního pohledu dar. Každý starý onboarding PDF nebo interní šablona zvyšují hodnotu celé sbírky.

### 3. Nepoužívat výchozí hesla v dokumentech

Jakmile se onboarding nebo instrukce dostanou mimo interní hranici, výchozí heslo se mění v přímý foothold. A i když už dávno "nemá platit", útočník to musí ověřit jen jednou.

### 4. Počítat s tím, že jmenný prostor firmy je veřejný

Obrana nemá stát na tom, že útočník nezná jména zaměstnanců. Má stát na tom, že ani s těmito jmény nezíská crackovatelný nebo znovupoužitelný přístup.

## Shrnutí klíčových poznatků

- Metadata dokumentů nejsou jen kosmetický OSINT detail. V AD prostředí často poskytují přímo stavební materiál pro útok na identitní vrstvu.
- Intelligence ukazuje celý řetězec: veřejné PDF, `Creator` metadata, validní uživatelé a nakonec i výchozí heslo v historickém dokumentu.
- Největší hodnota metadat neleží v jednom poli `Author`, ale v tom, že z větší sady dokumentů vznikne naming convention a kvalitní user list.
- Obrana stojí hlavně na redakci metadat, omezení veřejného archivu dokumentů a na tom, že interní hesla a onboarding instrukce nikdy neskončí v publikovaných materiálech.

## Co si odnést do praxe

- Když vidíš veřejný adresář s dokumenty, neřeš jen jejich obsah. Podívej se i na metadata, názvy souborů a historické verze.
- V doménovém prostředí může metadata leak znamenat začátek plnohodnotného identity řetězce, ne jen hezký seznam jmen pro poznámky.
- Dokument, který veřejně prozradí autora, naming convention a výchozí heslo, je z hlediska útoku často cennější než jeden izolovaný webový bug.
