---
layout: post
author: Tomáš Matějíček
title: "smbclient"
date: 2020-10-28
tags: nastroje smb windows shares enumeration
---
## Úvod a kontext

`smbclient` je v těchto rozborech jeden z nejpraktičtějších nástrojů pro okamžik, kdy už je k dispozici SMB přístup a je potřeba z něj vytěžit konkrétní hodnotu. Neřeší exploitaci SMB služby. Řeší mnohem běžnější a v praxi častější úlohu: projít sdílení, stáhnout soubory, ověřit oprávnění a číst provozní artefakty, které pak rozhodnou další krok.

Na [APT](/apt) pomůže získat `backup.zip`. Na [Nestu](/nest) otevře share `Users` a dovede k debug heslu pro HQK. Na [Anubis](/anubis) a [Intelligence](/intelligence) z něj vznikne pohodlný způsob, jak číst interní soubory a ověřit dopad získaného účtu. V tom je jeho praktická síla: když SMB není cíl, ale zdroj dat, `smbclient` bývá nejrychlejší cesta k dalšímu rozhodujícímu artefaktu.

## Co `smbclient` v praxi řeší

Jakmile existuje přístup k SMB, typické otázky bývají:

- jaké share jsou dostupné,
- co z nich lze číst,
- co dává smysl stáhnout lokálně,
- a zda se ve sdíleních neskrývá heslo, konfigurace, archiv nebo provozní stopa.

`smbclient` je praktický právě proto, že tyto otázky řeší přímo a bez velké režie. Místo složitějšího toolingu stačí rychle ověřit:

- seznam share,
- čitelnost konkrétního umístění,
- a hodnotu konkrétního souboru.

## Nejčastější scénáře využití v tomto projektu

### Anonymous nebo guest přístup ke sdílení

Na [APT](/apt) je `smbclient` první nástroj, který z anonymního SMB přístupu udělá něco opravdu cenného. Přes share `backup` je možné stáhnout `backup.zip`:

```bash
smbclient -N //dead:beef::b885:d62a:d679:573f/backup
```

Tím celý řetězec přechází ze síťového průzkumu k offline práci s doménovou zálohou. Prakticky je důležité, že SMB tu není "další otevřený port". Je to přímý zdroj identity materiálu. Širší dopad takového scénáře rozebírám i v článku [Doménové zálohy a offline těžba tajemství](/techniky/domenove-zalohy-a-offline-tezba-tajemstvi).

### První skutečný foothold přes čitelné interní soubory

Na [Nestu](/nest) a částečně i na [Intelligence](/intelligence) není hlavní hodnota v tom, že SMB "funguje". Důležité je, co lze číst.

Na Nestu po získání hesla `C.Smith` otevře `smbclient` share `Users` a z něj důležitý soubor:

```bash
smbclient -W '' //'10.10.10.178'/users -U'C.Smith'%'xRxRxPANCAK3SxRxRx'
allinfo "Debug Mode Password.txt"
more "Debug Mode Password.txt:Password"
```

Tohle je výborný praktický případ. `smbclient` tu není jen "klient na share". Je to nástroj, který přímo vede k debug heslu a dalšímu kroku v řetězci. Na Intelligence zase otevře čitelné share a potvrdí, že účet `Tiffany.Molina` není slepá credential clue, ale skutečně přináší interní přístup.

### Ověření, co účet umí v doménovém kontextu

Na windowsových a AD strojích bývá SMB často první rozumný test po získání identity. Pokud heslo nebo hash někde unikne, `smbclient` rychle odpoví:

- jestli účet funguje proti SMB,
- jaké share jsou vidět,
- a zda se v nich dá číst něco, co má praktickou hodnotu.

Na [Anubis](/anubis) je to důležité pro průzkum interní sítě přes proxy. Na [Resolute](/resolute) se podobnou logikou ověřuje účinek hesla proti `ipc$`, i když tam je použit jiný klient. Praktická lekce je stejná: SMB bývá první levná vrstva, na které lze validovat dopad identity.

## Co v `smbclientu` rozhoduje víc než samotný příkaz

### Správný kontext účtu

Na Windows prostředí je důležité vědět:

- zda jde o guest/null session,
- lokální účet,
- nebo doménovou identitu.

Stejný nástroj pak řeší velmi odlišné situace:

- anonymní přístup k backupu,
- validovaný přístup běžného uživatele,
- nebo čtení share přes doménový kontext.

### Správná otázka

Nejužitečnější není otevřít share "a projít všechno". Prakticky se vyplácí hledat:

- zálohy,
- konfigurační soubory,
- debug artefakty,
- textové poznámky,
- přístupové údaje,
- nebo názvy souborů, které odhalí další interní cestu.

Právě to odlišuje SMB enumeraci od běžného listování adresáři.

### Vědomí, že share je často jen mezikrok

Na [APT](/apt) share vydá `backup.zip`, ale skutečný dopad přináší až offline těžba. Na [Nestu](/nest) share vydá debug heslo, ale root přichází přes HQK. Na [Intelligence](/intelligence) share vydá skript `downdetector.ps1`, ale skutečný dopad se projeví až v další identitní a coercion fázi.

## Co `smbclient` neumí vyřešit za vás

`smbclient` sám:

- neobjeví nové heslo,
- nevyřeší zranitelnost služby,
- a neříká, které soubory jsou důležité.

Jeho hodnota závisí na tom, jestli umíte číst kontext:

- které share dávají smysl,
- jaké soubory hledat,
- a jak poznat, že z obyčejného textového souboru právě vznikl další foothold.

## Nejčastější chyby při použití

### Mechanické listování bez hypotézy

SMB share může obsahovat stovky souborů. Hodnota obvykle neleží v objemu, ale v několika konkrétních artefaktech:

- `backup.zip`,
- debug heslo,
- konfigurační XML,
- nebo interní skript.

### Zaměnění SMB přístupu za hotový průlom

Čitelný share ještě neznamená shell. Na blogu se opakovaně ukazuje, že SMB je často prostředník mezi první identitou a dalším silnějším krokem.

### Ignorování rozdílu mezi čtením a skutečným dopadem

Na [Nestu](/nest) je cenné debug heslo, ne samotný share. Na [APT](/apt) je cenný obsah archivu, ne fakt, že jde stáhnout soubor. Praktická hodnota tedy není "mám SMB", ale "co z SMB opravdu plyne".

## Nejčastější praktické scénáře

### Anonymous share nebo guest přístup

To je [APT](/apt) a částečně první fáze [Bankrobberu](/bankrobber).

### Po získání běžného účtu je potřeba rychle projít interní soubory

To je [Nest](/nest) a [Intelligence](/intelligence).

### Je třeba ověřit, zda credential clue otevírá aspoň SMB vrstvu

To je praktický mezikrok u více AD hostů, i když další shell pak přijde přes jiný kanál.

## Související články v projektu

- Rozbory: [APT](/apt), [Nest](/nest), [Intelligence](/intelligence), [Anubis](/anubis), [Fuse](/fuse)
- Techniky: [Doménové zálohy a offline těžba tajemství](/techniky/domenove-zalohy-a-offline-tezba-tajemstvi), [Metadata, logy, incidentní a forenzní artefakty jako zdroj přístupů](/techniky/metadata-logy-incidentni-a-forenzni-artefakty-jako-zdroj-pristupu), [Špatně vystavené datové, storage a cloud-like služby](/techniky/spatne-vystavene-datove-storage-a-cloud-like-sluzby)

## Co si odnést do praxe

- `smbclient` je nejpraktičtější ve chvíli, kdy už SMB přístup existuje a je potřeba rychle z něj vytěžit konkrétní artefakt.
- Nejde o nástroj na exploitaci SMB, ale o velmi účinný pracovní nástroj pro share, zálohy, konfigurace a provozní soubory.
- Největší hodnotu přinese tehdy, když k SMB nepřistupujete jako k cíli, ale jako ke zdroji dalšího rozhodujícího kroku.
