---
title: "CQS + functional programming = sant"
date: 2021-04-01
author: Fredde Johnsson, systemutvecklare
tagline: "CQS är ett imperativt mönster, functional programming är en deklarativ paradigm, kan man ändå kombinerar dom?"
header:
  overlay_image: https://www.infozone.se/wp-content/uploads/2020/03/programmering-i-team.jpg
categories:
  - blog
tags:
  - systemutveckling
  - cqs
  - functional-programming
---
# Outline
- Vad är functional programming? (kort, topp-5, referera till [http://blog.headlight.se/funktionell-programmering-med-csharp/](http://blog.headlight.se/funktionell-programmering-med-csharp/))
- Vad är CQS? 
  - command, 
  - query, 
  - process, 
  - query- och commandexecuter, 
  - dekorera med loggning, 
  - testning
- CQS + functional (varför? hur?)
- Exempel (Az func med DI)

# Bakgrund
I senaste kunduppdraget har kodandet varit till stor del fokuserat på serverless, e.g. Azure Functions, både node/javascript och C#. Av någon anledning är det lätt att bara kasta ihop sin kod, deploya och testa och sen är man nöjd med lösningen så länge den fungerar. Det behövs oftast inte så mycket kod för att lösa respektive problem, vilket är en av dom viktigaste styrkorna med Azure Functions och serverless. 

Men! Det finns ett stort MEN här. Det känns trots allt viktigt att hålla stringensen i sin kod, att följa ett mönster som gör koden robust och testbar och i och med det även förvaltningsbar över tid. Just nu är mitt favoritmönster CQS, [Command-Query separation](https://en.wikipedia.org/wiki/Command%E2%80%93query_separation). Det är relativt lättviktigt och jag tycker att det passar bra när man utvecklar Azure Functions. Jag gillar även paradigmen [functional programming](https://en.wikipedia.org/wiki/Functional_programming), en [deklarativ stil](https://en.wikipedia.org/wiki/Declarative_programming) att skriva källkod. Denna stil sägs ofta vara motsatsen till [imperativ programming](https://en.wikipedia.org/wiki/Imperative_programming). CQS klassas som ett imperativt mönster, så hur kan man kombinera CQS med functional då? Låt oss försöka! 

Men först lite basics rörande CQS och functional programming.
