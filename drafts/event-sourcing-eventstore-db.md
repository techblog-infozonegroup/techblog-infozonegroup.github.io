---
title: "Event Sourcing med EventStoreDB"
date: 2021-04-29
author: Andreas Hagsten, systemutvecklare
tagline: "EventStoreDB - en databas gjord för Event sourcing. Vi kollar på dess gRPC .NET klient."
header:
  overlay_image: https://raw.githubusercontent.com/techblog-infozonegroup/resources.techblog-infozonegroup/main/event-sourcing-a-different-view/eventsourcing-header.jpg
  teaser: https://raw.githubusercontent.com/techblog-infozonegroup/resources.techblog-infozonegroup/main/event-sourcing-a-different-view/eventsourcing-teaser.jpg
categories:
  - blog
tags:
  - systemutveckling
  - event sourcing
---

Jag har tidigare [skrivit om Event Sourcing](https://techblogg.infozone.se/blog/event-sourcing-a-different-view-on-things/) och nu dyker jag in i EvensStoreDB, databasen byggd för Event Sourcing. Häng med!

- Inledning om vad ESDB är
- Installation (docker image)
- gRPC, TCP, http
- Koppla upp sig mot sin ESDB-instans
- Lagra events i strömmar
    - Namngivning av strömmar --> kategorier
    - Strömmar är billiga!
- Läsa events från strömmar
    - Systemströmmar
        - $all
    - Filter
    - Stream directions och dess use cases
    - Audit logs exempel
- Läsa events från inbyggda projektioner (kategorier, etc)
- Prenumerera på strömmar