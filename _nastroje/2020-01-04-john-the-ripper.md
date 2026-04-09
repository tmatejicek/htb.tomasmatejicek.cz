---
layout: post
author: Tomáš Matějíček
title: "John the Ripper"
date: 2020-01-04
tags: nastroje cracking passwords archives keys hashes
---
## Úvod a kontext

`John the Ripper` se v tomto projektu používá hlavně jako praktický nástroj na zpracování různorodých artefaktů, které nejsou rovnou v podobě hashů připravených pro [Hashcat](/nastroje/hashcat). Jeho největší hodnota není jen v samotném crackingu, ale i v převodnících typu `zip2john`, `ssh2john`, `keepass2john` nebo `gpg2john`, které z archivů, klíčů a dalších formátů udělají crackovatelný vstup.

Na [APT](/apt) zpracuje heslo k `backup.zip`, na [OpenAdminu](/openadmin) passphrase k privátnímu SSH klíči a na [Controlu](/control) či [Traverxec](/traverxec) pomůže s dalšími typy credential artefaktů. To je jeho praktická role v projektu: když narazíte na archiv, klíč nebo jiný uzavřený soubor, `John` často představuje nejkratší cestu od artefaktu k dalšímu použitelnému tajemství.

## Co `John the Ripper` v praxi řeší

`John` je vhodný hlavně tehdy, když problém nevypadá jako "mám hash a potřebuji mód". Spíš vypadá takto:

- mám ZIP archiv,
- mám šifrovaný privátní klíč,
- mám `.htpasswd`,
- mám KeePass databázi,
- nebo jiný formát, který je potřeba nejdřív převést.

Právě tady bývá v praxi jednodušší než jiné cracking workflow, protože umí:

- vytvořit crackovatelný vstup,
- a pak ho rovnou zkusit prolomit.

## Nejčastější scénáře využití v tomto projektu

### Archiv nebo záloha chráněná heslem

Na [APT](/apt) je první praktický use case velmi čistý. Ze SMB share se stáhne `backup.zip`, ale skutečnou hodnotu má až ve chvíli, kdy se podaří získat jeho heslo:

```bash
zip2john backup.zip > backup.john
john --wordlist=/usr/share/wordlists/rockyou.txt backup.john
```

Tady je důležité, že `John` neřeší jen "heslo k ZIPu". Řeší přechod k mnohem hodnotnějšímu artefaktu, kterým je obsah doménové zálohy. Praktický dopad je tedy výrazně větší než samotné cracknutí archivu. Širší kontext téhle situace je v článku [Doménové zálohy a offline těžba tajemství](/techniky/domenove-zalohy-a-offline-tezba-tajemstvi).

### Passphrase k privátnímu klíči

Na [OpenAdminu](/openadmin) už privátní klíč existuje, ale je chráněný passphrase. Tady je `John` výborný právě díky `ssh2john.py`:

```bash
/usr/share/john/ssh2john.py OpenAdmin_joanna_id_rsa > OpenAdmin_joanna_id_rsa.john
/usr/sbin/john OpenAdmin_joanna_id_rsa.john --wordlist=/usr/share/wordlists/rockyou.txt
```

To je prakticky jiný scénář než běžný hash cracking:

- nejde o password database,
- ale o klíčový materiál,
- který už sám o sobě otevírá SSH.

Právě proto je `John` v takových případech velmi užitečný. Převod a cracking tvoří jeden přirozený pracovní krok.

### `.htpasswd`, exporty a jiné menší credential artefakty

Na [Controlu](/control), [Traverxec](/traverxec), [SneakyMailer](/sneakymailer), [Schooled](/schooled) nebo [Cache](/cache) se `John` objevuje nad menšími, ale prakticky hodnotnými soubory:

- `.htpasswd`,
- exportované hashy,
- nebo další konfigurace obsahující crackovatelný materiál.

V takových případech má `John` velkou výhodu v tom, že je rychlý na nasazení. Když už soubor leží na disku, často stačí krátký převod a jeden přímý pokus se slovníkem.

### Specializované převodníky pro méně běžné formáty

Na [BigHeadu](/bighead) se objevuje `zip2john` i `keepass2john`, na [Boltu](/bolt) `gpg2john`. To je praktická připomínka, že `John` je v projektu důležitý hlavně jako ekosystém převodníků:

- archiv,
- klíč,
- KeePass databáze,
- GPG klíč,
- nebo jiný chráněný artefakt

se dá převést do formy, která už je crackovatelná standardním workflow.

## Kdy je `John` nejlepší volba

Největší smysl dává tehdy, když:

- už máte konkrétní soubor nebo artefakt,
- nejde o "čistý" hash vhodný rovnou pro jiné workflow,
- a praktická hodnota spočívá v jeho otevření nebo v získání passphrase.

Tady `John` obvykle přináší větší hodnotu než snaha složit vlastní konverzi formátu nebo hned přeskakovat na jiný cracking nástroj.

## Co `John` neumí vyřešit za vás

`John`:

- sám neříká, zda artefakt stojí za cracknutí,
- neřeší, jak jste se k němu dostali,
- a neznamená automaticky, že po cracknutí vznikne privilegovaný přístup.

Na [APT](/apt) otevírá celou doménovou zálohu. Na [OpenAdminu](/openadmin) otevírá klíč pro další SSH hop. Jinde může mít hodnota výrazně menší dopad. Praktický význam tedy vždy závisí na tom, co daný soubor skutečně chrání.

## Nejčastější chyby při použití

### Podcenění převodní části

Největší praktická síla `Johna` bývá právě v převodnících. Pokud někdo vnímá nástroj jen jako "cracker hashů", přehlédne přesně ty situace, kde je v projektu nejcennější:

- `zip2john`,
- `ssh2john`,
- `keepass2john`,
- `gpg2john`.

### Cracknutí artefaktu bez zamyšlení nad dopadem

Stejně jako jinde je důležité vědět, co otevřený soubor vlastně znamená:

- archiv se zálohou,
- privátní klíč,
- `.htpasswd`,
- nebo lokální konfigurační soubor.

Právě to určuje, jestli jde o kosmetický výsledek, nebo o zásadní posun v řetězci.

### Záměna `Johna` za univerzální náhradu všech cracking workflow

V některých případech je praktičtější jiný nástroj. Pokud už držíte čistý hash nebo roast materiál, bývá přímější sáhnout po [Hashcatu](/nastroje/hashcat). `John` má v tomto projektu největší hodnotu tam, kde nejdřív potřebujete převést artefakt do crackovatelné podoby.

## Nejčastější praktické scénáře

### Získali jste archiv a potřebujete se dostat k jeho obsahu

To je [APT](/apt) nebo [BigHead](/bighead).

### Získali jste privátní klíč, ale chybí passphrase

To je [OpenAdmin](/openadmin) a [Traverxec](/traverxec).

### Získali jste menší konfigurační nebo autentizační soubor a potřebujete rychle ověřit, jestli z něj jde dostat další heslo

To je [Control](/control), [Cache](/cache), [SneakyMailer](/sneakymailer) a další podobné případy.

## Související články v projektu

- Rozbory: [APT](/apt), [OpenAdmin](/openadmin), [Control](/control), [Traverxec](/traverxec), [BigHead](/bighead), [SneakyMailer](/sneakymailer)
- Techniky: [Doménové zálohy a offline těžba tajemství](/techniky/domenove-zalohy-a-offline-tezba-tajemstvi), [Klientské, desktopové a ne-textové artefakty po footholdu](/techniky/klientske-desktopove-a-ne-textove-artefakty-po-footholdu)

## Co si odnést do praxe

- `John the Ripper` je v projektu nejcennější jako praktický převodník a crack pipeline pro archivy, klíče a jiné chráněné artefakty.
- Největší hodnotu má ve chvíli, kdy už držíte konkrétní soubor a potřebujete z něj dostat další tajemství.
- Nejde jen o samotný cracking; často rozhoduje už to, že `John` umí daný formát vůbec převést do použitelné podoby.
