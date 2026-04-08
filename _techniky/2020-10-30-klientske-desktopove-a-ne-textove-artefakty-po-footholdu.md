---
layout: post
author: Tomáš Matějíček
title: "Klientské, desktopové a ne-textové artefakty po footholdu"
date: 2020-10-30
tags: windows credentials post-exploitation keepass teamviewer artifacts
---
## Úvod a kontext

Po prvním shellu se pozornost často zbytečně zužuje na klasické cíle: `config.php`, `.env`, `id_rsa`, `sudo -l` nebo seznam SUID binárek. Jenže v reálném prostředí bývá velká část citlivých údajů uložená jinde: v desktopových nástrojích, poštovních archivech, správcích hesel, serializovaných credential objektech nebo v různých „ne-textových“ formátech, které na první pohled nevypadají důležitě.

Právě tyto artefakty bývají po footholdu mimořádně cenné. Ne proto, že by samy obsahovaly exploit, ale protože propojují běžný uživatelský kontext s další identitou, vzdálenou správou nebo privilegovaným kanálem.

## Proč jsou tyto artefakty tak důležité

Mají tři společné vlastnosti:

- vznikají kvůli pohodlí uživatele nebo administrátora,
- bývají uložené lokálně na kompromitovaném hostu,
- často obsahují tajemství v podobě, kterou správce nepovažuje za „normální heslo v souboru“.

Typicky jde o:

- mailboxy a jejich přílohy,
- lokální databáze desktopových nástrojů,
- registry nebo konfigurační XML vzdálených support nástrojů,
- správce hesel a jejich key file,
- PowerShell serializované credential objekty,
- bytecode nebo binární artefakty s obfuskovaným heslem.

Z pohledu útočníka je důležité jedno: nejsou to nevinné uživatelské soubory, ale další vrstva credential storage.

## Typické kategorie artefaktů

### 1. Mail archivy a přílohy

Arkham ukazuje, že poštovní archiv není jen historický záznam komunikace. `alfred@arkham.local.ost` obsahoval přílohu, která po dekódování vydala další přihlašovací údaj a otevřela přechod do účtu `batman`.

Tady je důležité vnímat, proč to fungovalo. Mailbox není jen sbírka zpráv, ale i sklad:

- interních instrukcí,
- příloh s tajemstvími,
- screenshotů,
- dočasných hesel,
- exportů nebo „jen na chvíli“ poslaných přístupů.

Jakmile je na hostu lokální kopie pošty, je potřeba ji brát jako plnohodnotný zdroj privilegovaných informací.

### 2. Desktopové nástroje s reverzibilním uložením tajemství

Sharp je čistý příklad toho, jak nebezpečné jsou lokální databáze desktopových nástrojů. `PortableKanban.pk3` neobsahoval jen obsah boardu, ale reverzibilně uložená hesla. Ta pak otevřela další share a nakonec i cestu k debug endpointu.

Podobný princip se opakuje i u různých pomocných utility, klientů helpdesku nebo synchronizačních nástrojů:

- lokálně si ukládají přihlášení,
- často kvůli pohodlí uživatele drží klíč i data na stejném stroji,
- kompromitace hostu pak automaticky znamená kompromitaci další vrstvy prostředí.

### 3. Nástroje vzdálené podpory a unattended access

Remote ukazuje, jak silný je tento vzorec u TeamVieweru. Lokálně uložené `SecurityPasswordAES` po dešifrování vydalo heslo, které fungovalo pro `Administrator`.

To je typický případ, kdy support nástroj zvyšuje blast radius kompromitace:

- běží na serveru nebo admin workstation,
- chrání privilegovaný přístup,
- a přitom nechává lokálně tajemství, které lze po footholdu vyčíst.

Jakmile na stejném hostu leží i produkční služba, vzdálená správa přestává být oddělený kanál.

### 4. Správci hesel, key file a vedlejší formáty

BigHead je výborná připomínka, že tajemství nemusí ležet v jednom souboru. `root.txt:Zone.Identifier` vedl ke KeePass databázi, `KeePass.config.xml` prozradil vazbu na key file a teprve spojení databáze, klíče a hesla otevřelo finální tajemství.

Důležité je nepřehlédnout okolní soubory:

- konfigurační XML,
- key file v obrázku nebo v profilu uživatele,
- asociace databáze a klíče,
- metadata, která ukazují, jak má být tajemství použité.

Samotná KDBX databáze ještě nemusí stačit. Ale když na stejném hostu leží i key file a konfigurace klienta, celý model ochrany se rozpadá.

### 5. Serializované credential objekty a specializované formáty

Omni ukazuje jiný, ale stejně praktický vzor. Flagy ani přístupy neležely jako obyčejný text, ale jako PowerShell `CliXml` serializované credential objekty. Bez znalosti formátu by takový soubor vypadal jako nečitelný nebo nepoužitelný artefakt.

To je důležitá lekce obecně. Po footholdu je potřeba myslet i na formáty, které:

- nejsou plaintext,
- nejsou klasická databáze hesel,
- ale přesto představují přímo použitelný credential container.

### 6. Bytecode, binárky a obfuskované lokální skripty

Wall připomíná, že heslo nemusí být v konfiguraci ani ve vaultu. Může být schované v bytecode souboru, který někdo považuje za „jen interní skript“. Dekompilovaný `backup` vydal heslo pro `shelby`, i když na disku neležel čitelný plaintext.

Obfuskované heslo není bezpečnostní kontrola. Je to jen jiná reprezentace stejného tajemství.

## Jak tyto artefakty po footholdu hledat

Užitečné je přemýšlet po vrstvách, ne po příponách.

### Co na tom hostu používá člověk, ne služba

Sem patří hlavně:

- poštovní klient,
- TeamViewer a podobné support nástroje,
- KeePass a jiné vaulty,
- desktopové utility typu PortableKanban,
- editor konfigurace a pomocné skripty.

Jakmile víš, že host nepoužívá jen serverový software, ale i klientské nebo admin desktop nástroje, výrazně roste šance na lokální tajemství.

### Kde nástroj ukládá tajemství a kde metadata o nich

Nestačí hledat jen samotnou databázi. Je potřeba hledat i:

- key file,
- registry klíče,
- XML konfiguraci,
- serializované objekty,
- historii posledních otevřených souborů,
- skryté proudy a vedlejší artefakty.

Velmi často platí, že teprve kombinace několika souborů dá použitelné tajemství.

### Jaký další kanál ten artefakt otevírá

Nejcennější jsou ty artefakty, které nevedou jen k „dalšímu souboru“, ale k nové identitě nebo správní vrstvě:

- SSH,
- WinRM,
- RDP přes [xfreerdp](/nastroje/xfreerdp),
- vzdálený support,
- jiný lokální účet,
- admin rozhraní,
- privilegovaný skript nebo vault.

To je praktické kritérium pro prioritu. Nejde o to nasbírat co nejvíc exotických formátů, ale najít ty, které otevírají další důvěryhodný kanál.

## Obrana a hardening

### 1. Neprovozovat admin desktop nástroje na stejných hostech jako kompromitovatelnou službu

Pokud na serveru běží web a zároveň TeamViewer, browser s přihlášeným adminem nebo lokální vault, první foothold má mnohem větší dopad.

### 2. Počítat s tím, že lokální kompromitace mění model ochrany

Reverzibilně uložené heslo, key file ve stejné složce nebo serializovaný credential objekt nejsou bezpečné jen proto, že nejsou v plaintextu. Jakmile je útočník na hostu, často má k dispozici všechny části potřebné k jejich použití.

### 3. Oddělit tajemství od klientských profilů

Správci hesel, support nástroje a desktop utility by neměly ukládat dlouhodobé privilegované přístupy v podobě, která je po kompromitaci jedné stanice okamžitě zneužitelná.

### 4. Auditovat i ne-textové formáty

Bezpečnostní review nesmí končit u `grep password`. Je potřeba vědět, kde prostředí používá:

- KDBX a key file,
- `CliXml`,
- mailbox archivy,
- reverzibilní lokální databáze klientů,
- registry klíče vzdálené správy,
- bytecode nebo interní helper binárky.

### 5. Chránit administrativní workflow, ne jen soubory

Mnohé z těchto artefaktů dávají smysl jen proto, že otevírají další správní kanál. Obrana tedy není jen o zašifrování souboru, ale o tom, aby tentýž host zároveň nedržel službu, klientský nástroj i privilegovanou identitu.

## Shrnutí klíčových poznatků

- Po footholdu neleží další tajemství jen v konfiguračních souborech. Velmi často jsou schovaná v klientských, desktopových a ne-textových artefaktech.
- Největší hodnotu mají artefakty, které propojí kompromitovaný host s dalším přístupovým kanálem: SSH, WinRM, TeamViewer, vault nebo jiný účet.
- U těchto formátů je potřeba hledat celý kontext, ne jen jeden soubor. Databáze, key file, XML konfigurace a registry klíč často dávají smysl až dohromady.
- Obfuskované nebo serializované tajemství není bezpečnější tajemství. Je to jen méně nápadné tajemství.

## Co si odnést do praxe

- Po prvním shellu se dívej i po tom, co na hostu používá člověk: pošta, support nástroj, vault, desktop utility, PowerShell credential objekty.
- Hledej vedlejší soubory, které vysvětlují, jak se artefakt používá. Právě ty často rozhodnou, jestli je nalezené tajemství skutečně zneužitelné.
- Pokud jeden host kombinuje serverovou službu, klientské nástroje a privilegované identity, kompromitace jedné vrstvy se velmi rychle přelije do všech ostatních.
