---
layout: default
title: "Techniky"
permalink: /techniky/
---
## Techniky a zranitelnosti

Samostatná rubrika pro výukové a souhrnné texty. HTB rozbory strojů zůstávají v hlavním seznamu, tady jsou články zaměřené na techniky, třídy chyb, zneužitelné vzory a obranné souvislosti.

<ul>
  {% assign articles = site.techniky | sort: "date" | reverse %}
  {% for article in articles %}
    <li>
      <h2><a href="{{ article.url }}">{{ article.title }}</a></h2>
      <small>{{ article.date | date: "%-d %B %Y" }}</small>
      <p>{{ article.content | strip_html | truncatewords: 30 }}</p>
    </li>
  {% endfor %}
</ul>
