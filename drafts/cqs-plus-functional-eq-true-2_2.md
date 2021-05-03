---
title: "CQS + functional programming = sant (del 2/2)"
date: 2021-05-01
author: Fredde Johnsson, systemutvecklare
tagline: "Här kommer post 2 där vi tittar på kod för att försöka besvara frågan 'CQS är ett imperativt mönster, functional programming är en deklarativ paradigm, kan man ändå kombinera dom?'"
header:
  overlay_image: https://user-images.githubusercontent.com/460203/115292819-c3d97580-a156-11eb-972c-d0dad4361b75.jpg
  teaser: https://user-images.githubusercontent.com/460203/115826217-d4b21180-a40a-11eb-894a-e3e367bbe140.png
categories:
  - blog
tags:
  - systemutveckling
  - cqs
  - functional programming
  - funktionell programmering
---

Beskriv:
- Process/UpdateUserProcess
- Query/GetUserBySsnQuery
- Command/UpdateNameCommand
- QueryExecuter
- CommandHandler
- IoC/DI

# Kort recap från del 1
Innan vi tittar på kod så gör vi en kort återblick på vad vi gick igenom i [första delen](https://techblogg.infozone.se/blog/cqs-plus-functional-eq-true-1_2/):

- CQS - Command Query Separation, skillda "pipor" från commands (operationer) och queries (frågor)
- En definition av komplext kommando, en process, ger oss verktygen att bygga dom flesta systemen
- Tänkt funktionellt, functional programming är mindre felbenägen och robust

Det funktionella kommer bli tydligare i och med att vi introducerar enhetstester och på det sättet driver fram koden eller automatiserar regressionstestningen av systemets olika delar.

# Vad ska vi bygga
En bild över det vi ska bygga placerar in alla byggklossar på sina respektive platser i lösningen. Vi realiserar lösningen genom att implementera den som en Azure function byggd enligt CQS + functional programming:

![func-cqs-process](https://user-images.githubusercontent.com/460203/116928893-f490d300-ac5d-11eb-86a8-0f84910a30ae.png)

All källkod finns här [https://github.com/Fjeddo/Azure-function-CQS-pattern](https://github.com/Fjeddo/Azure-function-CQS-pattern). Innan vi sätter igång vill jag introducera ytterligare två spelare:
- QueryExecuter - en komponent som sköter all exekvering av frågor
- CommandHandler - en komponent som hanterar alla kommandon som ska utföras

Att centralisera exekvering av frågor och hantering av kommandon öppnar upp möjligheter att "dekorera" anropen med loggning och felhantering.

> Den här lösnigen liknar Decorator Pattern. Mer om detta mönster finns här [https://www.dofactory.com/net/decorator-design-pattern](https://www.dofactory.com/net/decorator-design-pattern).

qwerty
