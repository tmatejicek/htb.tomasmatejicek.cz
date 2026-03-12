---
layout: post
author: Tomáš Matějíček
title: "Multimaster"
date: 2020-12-18
tags: linux exploit privesc enumeration hackthebox
---
[Multimaster](https://www.hackthebox.eu/home/machines/profile/232) je stroj z Hack The Box. Cílem je přejít od prvotní enumerace až k plnému ovládnutí systému.

### Vyhledání otevřených portů
Nejdřív mapuji služby, které jsou dostupné zvenku.
`ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP`

### Získání user flagu
`TODO`

### Získání root flagu
`TODO`
