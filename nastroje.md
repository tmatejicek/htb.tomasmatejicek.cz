---
layout: default
title: "Nástroje"
permalink: /nastroje/
---
## Nástroje

Samostatná rubrika pro články o nástrojích, které se opakovaně používají v HTB rozborech strojů. Nejde o přepis manuálových stránek, ale o praktické texty k použití, omezením, rozhodování a typickým chybám při práci s těmito nástroji.

<ul>
  {% assign articles = site.nastroje | sort: "date" | reverse %}
  {% for article in articles %}
    <li>
      <h2><a href="{{ article.url }}">{{ article.title }}</a></h2>
      <small>{{ article.date | date: "%-d %B %Y" }}</small>
      <p>{{ article.content | strip_html | truncatewords: 30 }}</p>
    </li>
  {% endfor %}
</ul>
