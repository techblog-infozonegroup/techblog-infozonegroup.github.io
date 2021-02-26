---
title: "Test av draft"
date: 2021-02-28
author: Fredde, systemutvecklare
tagline: "Hej hopp!"
header:
  overlay_image: https://www.infozone.se/wp-content/uploads/2020/03/programmering-i-team.jpg
categories:
  - blog
tags:
  - bicep
  - azure
  - arm
  - devops
---
[Docker](https://www.docker.com/), [kubernetes](https://kubernetes.io/) och annan containerteknik kanske ibland känns som verktyg eller plattformar för servrar, hosting, PaaS och stora jättar uppe bland molnen. För mig har det däremot dom senaste 4-5 åren varit en naturlig del av mitt verktygsbälte med utvecklingsverktyg.

I den här posten tänkte jag, bara som ett exempel, visa hur jag använde docker för att slippa installera grejor på min arbetsmaskin. I dom flesta fallen kanske det handlar om att slippa installera serverkomponenter såsom Elasticsearch, Redis, Sql Server (for Linux) osv, men i det här exemplet beskriver jag hur jag använder docker för att testa [Project Bicep](https://github.com/Azure/bicep) och dess CLI för att generera ARM Templates (Azure Resource Manager).

# Docker (Desktop for Windows)
Eftersom jag till 100% sitter och utvecklar på en Windowsmaskin så var det en fröjd den dagen [Docker for Windows](https://hub.docker.com/editions/community/docker-ce-desktop-windows) kändes stabilt och bra. Idag är det ett verktyg som ingår i min standardinstallation av arbetsmaskin. Smidigheten i att alltid ha det till hands lokalt gör det otroligt användbart, möjligheten att välja om jag ska köra Windows- eller Linux-containrar är väldigt värdefull.

Den absoluta merparten av tiden köra jag Linux-containrar, så även i det här exemplet. En stor anledning till att det är så är att Microsoft gör ett allt bättre jobb vad gäller plattformsoberoende verktyg och kompatibilitet i sina SDK:er och plattformar.

Vad docker är och containerization innebär finns det oändlig information på Internet.

# Project Bicep och ARM Templates

# Bicep CLI i container

