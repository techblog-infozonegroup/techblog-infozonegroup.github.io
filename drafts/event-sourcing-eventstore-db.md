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

# Kort om EventStoreDB
EventStore DB är en databas byggd för Event Sourcing. Den lagrar din data i form av strömmar som ej går att förändra i efterhand, så kallade "Immutable streams". EventStore kommer med många kraftfulla funktioner "out of the box". Prenumerering på strömmar är ett exempel som gör det möjligt att reagera i realtid på förändringar i databasen. En form av PubSub helt enkelt. En annan bra funktion som jag vill belysa är inbyggda projektioner. Dessa är oftast kategoriseringar av dina egna strömmar och events, bra om man till exempel vill läsa upp alla events av en viss typ. Jag går in djupare på detta [#TBD](här).

> Denna artikel beskriver endast grunderna i Event Store. Nyttjandet kommer på sina ställen vara naivt och uppsättningen enkel. 

# Att komma igång
Jag vill gärna börja hacka kod så fort som möjligt för att kunna känna och klämma på EventStore. Ett enkelt sätt att få upp något att hacka mot är att nyttja en Docker compose.

## Installation - Docker compose
Vi behöver inte krångla till det för mycket med installationen utan jag nyttjar följande Docker compose-fil för att få upp en enkel testmiljö av EventStore. 

```yml
version: '3.4'

services:
  eventstore.db:
    image: eventstore/eventstore
    environment:
      - EVENTSTORE_CLUSTER_SIZE=1
      - EVENTSTORE_RUN_PROJECTIONS=All
      - EVENTSTORE_START_STANDARD_PROJECTIONS=true
      - EVENTSTORE_EXT_TCP_PORT=1113
      - EVENTSTORE_EXT_HTTP_PORT=2113
      - EVENTSTORE_INSECURE=true
      - EVENTSTORE_ENABLE_EXTERNAL_TCP=true
      - EVENTSTORE_ENABLE_ATOM_PUB_OVER_HTTP=true
    ports:
      - "1113:1113"
      - "2113:2113"
    volumes:
      - type: volume
        source: eventstore-volume-data
        target: /var/lib/eventstore
      - type: volume
        source: eventstore-volume-logs
        target: /var/log/eventstore

volumes:
  eventstore-volume-data:
  eventstore-volume-logs:
```

> Detta är inte en produktionsuppsättning, men det man kan ta med sig är att vi sätter upp en enda nod (cluster_size=1) och att vi nyttjar alla inbyggda projektioner. Notera också portarna för att kunna hitta till ESDB-dashboarden.

> För att starta din instans av EventStore kör du kommandot "docker-compose up" i den katalog som yml-filen ligger i

## .NET klienter
I denna artikel kommer jag ge kodexempel i C#, därför kommer fokus ligga på det .NET-stöd som EventStore kommer med. Men förutom .NET kommer EventStore även med stöd för node, Java, Go och Rust.

Det finns olika .NET-klienter som kan prata med databasen. Det som särskiljer klienterna är vilket protokoll som de pratar snarare än olika API. Det finns klienter för TCP, HTTP och gRPC. gRPC är den senaste och numera den primära klienten framöver, rekommendationen är alltså att ni använder gRPC om ni bygger nytt med EventStore!

## Koppla upp mot sin EventStore-instans
När instansen är skapad, via "docker-compose up", blir en dashboard tillgänglig på http://localhost:2113/. Här kan du se en mängd metrics men också, kanske mer intressant för oss, våra framtida streams. Se det som SQL Server Management Studio om du kommer från en SQL-bakgrund.

[Bild på ES-dashboard]

Instansen är igång! Nästa steg är att koppla upp oss mot den via .NET-klienten. Det vi behöver är en instans av EventStoreClient och en connectionstring. Koden nedan visar en mycket enkel connection string där vi endast konfigurerar klienten att inte nyttja tls. Det finns mängder av andra inställningar att grotta ned sig i [på EventStores webbsida](https://developers.eventstore.com/clients/dotnet/5.0/connecting/connecting-to-a-server.html#eventstoreconnection).

```csharp
var client = new EventStoreClient(EventStoreClientSettings.Create("esdb://localhost:2113?tls=false"));
```

Nu ska du förhoppningsvis vara uppkopplad mot din ES-instans och därmed ha tillgång att läsa och skriva events.

# Skriva events
Grundidén med Event Sourcing är att lagra sin applikations data som en events. Dessa events har en tidsföljd och bildar eventströmmar (engelskans event streams). I Event Store skapar du därför strömmar dit du skriver dina events. Man kan i en förenklad värld jämföra en ström med en tabell.

## Strömmar är billiga
Event Store är byggt för att hantera mängder av strömmar. Ur prestandaskäl finns det inget negativt med att skapa en ström, tvärtom så rekommenderas det att använda många strömmar eftersom det får trevliga effekter som att de t.ex. blir kortare. Långa strömmar är en stor källa till prestandaproblem och komplex logik. När en ström blir "för lång" kommer man ofta in på ämnet snapshotting som kan vara svårt att få till bra. 

Håll nere storleken på era strömmar, var inte rädd att skapa nya. **Strömmar är billiga!** Ett konkret tips är en ström per entitet.

## Namngivning av strömmar
Namngivning har betydelse, Event Store kategoriserar nämligen dina strömmar med allt till vänster om det första bindestrecket ("-"). Om du till exempel har strömmarna

- User-1
- User-2
- User-42

skapar Eventstore en ny ström med namn $ce-User som innehåller alla events för användare med ID 1, 2 och 42. Ett vanligt use case för detta är man vill prenumererar på user-events, oavsett vilken användare eventet rör. Dessa kategoriseringar, eller projektioner som det heter, kan prenumereras på precis som vilken ström som helst. 

> Strömmar som börjar med $ är systemströmmar. Det är fritt fram att nyttja dem precis som dina egna strömmar. $all är t.ex. den ström som innehåller alla events som nånsin lagrats.

För att summera så är rekommendationen att namnge sina events med bindestreck för att få med kategorisering. Nån form av standard skulle kunna vara formatet "Vad-Vilken" (User-1).

# Läsa events

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