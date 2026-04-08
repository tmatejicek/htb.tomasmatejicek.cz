---
layout: post
author: Tomáš Matějíček
title: "Impacket pro relay, tajemství a delegovaná práva"
date: 2020-10-29
tags: nastroje active-directory windows relay dcsync gmsa kerberos
---
## Úvod a kontext

Jakmile v AD řetězci přestanete řešit první identity a začnete pracovat s delegovanými právy, servisními tajemstvími nebo replikační logikou domény, přichází jiná rodina utilit. Sem prakticky patří hlavně `ntlmrelayx.py`, `secretsdump.py`, `impacket-getST` a v těsném sousedství i `gMSADumper.py`. Ne všechny jsou součástí Impacketu striktně vzato, ale v reálném workflow fungují jako jedna pracovní rodina: z autentizace nebo servisního účtu dělají přímý přístup k tajemstvím a privilegiím.

Na [Forestu](/forest) to znamená relay do LDAP a DCSync. Na [Intelligence](/intelligence) čtení gMSA hesla a service ticket pro impersonaci. Na [APT](/apt) offline i online těžbu doménových tajemství. Na [Sauně](/sauna) potvrzení, že servisní účet už má dost síly na replikační dump.

## Co tahle rodina v praxi řeší

Prakticky jde o utility pro tři typy situací:

- když lze přimět cizí identitu k autentizaci a zneužít relay,
- když už máte účet nebo hash a chcete z něj vytěžit doménová tajemství,
- nebo když servisní identita dovolí vyžádat ticket nebo číst heslo jiného účtu.

To je kvalitativně jiná fáze než první AD enumerace. Tady už nejde o to "kdo existuje", ale o to "jakou moc teď ta identita opravdu má".

## Nejčastější scénáře využití v tomto projektu

### `ntlmrelayx.py`: převod vynucené autentizace na změnu práv

Na [Forestu](/forest) nestačilo mít `svc-alfresco`. Rozhodující bylo, že šlo přes PrivExchange vynutit autentizaci a relaynout ji proti LDAP:

```bash
ntlmrelayx.py -t ldap://10.10.10.161 --escalate-user svc-alfresco
```

Praktická hodnota `ntlmrelayx.py` je právě v tom, že převádí jednu technickou vlastnost prostředí do druhé: z NTLM autentizace udělá zápis do AD nebo jiný důvěryhodný krok na cílové službě. Bez pochopení doménové logiky by tenhle nástroj nepomohl. S dobrou hypotézou ale často rozhodne celý řetězec.

### `secretsdump.py`: offline a online těžba identitních tajemství

Na [APT](/apt) je `secretsdump.py` pěkně vidět v obou hlavních rolích.

Nejprve offline nad uniklou zálohou:

```bash
impacket-secretsdump -ntds "Active Directory/ntds.dit" -system registry/SYSTEM -hashes lmhash:nthash LOCAL -outputfile ntlm-extract
```

A později online přes získanou identitu:

```bash
impacket-secretsdump -hashes __CENSORED__:__CENSORED__ 'apt.htb/APT$@apt.htb.local'
```

Na [Sauně](/sauna) je pattern ještě čistší. Účet `svc_loanmgr` už má taková práva, že lze doménové hashe vytáhnout přímo:

```bash
secretsdump.py 'EGOTISTICAL-BANK.LOCAL/svc_loanmgr:Moneymakestheworldgoround!@sauna.EGOTISTICAL-BANK.LOCAL'
```

Právě tahle utilita velmi dobře odděluje "mám zajímavý účet" od "ten účet opravdu vede k doménovým tajemstvím".

### `gMSADumper.py` a `impacket-getST`: servisní identita jako most k cizímu ticketu

Na [Intelligence](/intelligence) je vidět jiná rodina moci. Účet `Ted.Graves` sám o sobě nestačil, ale mohl číst heslo gMSA účtu `svc_int$`:

```bash
python3 gMSADumper.py -u 'Ted.Graves' -p 'Mr.Teddy' -d 'intelligence.htb' -l 'dc.intelligence.htb'
```

Jakmile byl k dispozici NTLM hash gMSA, navázal `impacket-getST`:

```bash
impacket-getST intelligence.htb/svc_int$ -spn WWW/dc.intelligence.htb -hashes :c699eaac79b69357d9dabee3379547e6 -impersonate Administrator
```

To je prakticky důležitý moment. Nejde o "další hash". Jde o servisní identitu, která umí otevřít ticket v cizím kontextu. Právě proto patří tahle rodina utilit mezi nejdůležitější AD post-foothold nástroje.

Širší kontext téhle logiky je v článku [Delegovaná práva v AD: DCSync, gMSA, relay, deleted objects](/techniky/delegovana-prava-v-ad-dcsync-gmsa-relay-deleted-objects).

## Kdy jsou tyto utility nejlepší volba

Tahle rodina dává největší smysl tehdy, když:

- už máte účet, hash nebo vynucenou autentizaci,
- řešíte DCSync, relay, gMSA nebo service tickety,
- nebo potřebujete potvrdit, zda servisní identita vede k doménovým tajemstvím.

Naopak to nejsou nástroje pro úplně první foothold. Bez dobré identity nebo přesného privilegovaného kontextu jen vytváří hluk.

## Nejčastější chyby v praxi

### Nechápat rozdíl mezi tajemstvím a použitelným oprávněním

To, že účet zná nějaké heslo nebo umí číst nějaký objekt, ještě neznamená, že vede k DCSync nebo impersonaci. Vždy je potřeba ověřit konkrétní dopad.

### Míchat relay, dump a tickety do jedné kategorie

Všechny tyto utility pracují s identitou, ale řeší různé otázky:

- `ntlmrelayx.py` mění průběh autentizace,
- `secretsdump.py` čte tajemství,
- `gMSADumper.py` a `impacket-getST` pracují se servisní identitou a Kerberem.

Nejlépe fungují, když víte, kterou z těchto vrstev právě řešíte.

### Přeskakovat obranný kontext

Forest i Intelligence ukazují, že nejde o jeden "nástrojový trik". Jde o kombinaci slabého trust modelu, delegovaných práv a provozních identit. Bez toho jsou utility jen transport.

## Související články v projektu

- Rozbory: [APT](/apt), [Forest](/forest), [Intelligence](/intelligence), [Sauna](/sauna)
- Techniky: [Delegovaná práva v AD: DCSync, gMSA, relay, deleted objects](/techniky/delegovana-prava-v-ad-dcsync-gmsa-relay-deleted-objects), [Doménové zálohy a offline těžba tajemství](/techniky/domenove-zalohy-a-offline-tezba-tajemstvi), [NTLM coercion a vynucená autentizace](/techniky/ntlm-coercion-a-vynucena-autentizace)
- Nástroje: [Impacket](/nastroje/impacket), [BloodHound](/nastroje/bloodhound)

## Co si odnést do praxe

- Tato rodina utilit je nejcennější v okamžiku, kdy už máte identitu a potřebujete z ní vytěžit delegovanou moc nebo tajemství.
- Prakticky pokrývá relay, dump hashů, čtení gMSA a práci se service tickety.
- Nejde o sadu náhodných skriptů. Jde o pracovní vrstvu pro AD situace, kde rozhodují důvěra, replikační práva a servisní identity.
