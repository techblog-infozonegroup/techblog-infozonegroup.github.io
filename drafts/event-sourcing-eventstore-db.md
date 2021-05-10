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
  - eventstoredb
---

Jag har tidigare [skrivit om Event Sourcing](https://techblogg.infozone.se/blog/event-sourcing-a-different-view-on-things/) och nu dyker jag ner i EvensStoreDB, databasen byggd för Event Sourcing. Häng med!

# Kort om EventStoreDB
EventStore DB är en databas byggd för Event Sourcing. Den lagrar din data i form av strömmar som ej går att förändra i efterhand, så kallade "Immutable streams". EventStore kommer med många kraftfulla funktioner "out of the box". Prenumerering på strömmar är ett exempel som gör det möjligt att reagera i realtid på förändringar i databasen. En form av PubSub helt enkelt. En annan bra funktion som jag vill belysa är inbyggda projektioner. Dessa är oftast kategoriseringar av dina egna strömmar och events, bra om man till exempel vill läsa upp alla events av en viss typ. Jag går in djupare på detta [i bloggposten om Event Sourcing.](https://techblogg.infozone.se/blog/event-sourcing-a-different-view-on-things/#projektioner-och-l%C3%A4smodeller).

> Denna artikel beskriver endast grunderna i EventStore. Nyttjandet kommer på sina ställen vara naivt och uppsättningen enkel. 

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

Kör "docker-compose up" i den katalog som yml-filen ligger i för att starta din instans av EventStore.

> Detta är inte en produktionsuppsättning, men det man kan ta med sig är att vi sätter upp en enda nod (cluster_size=1) och att vi nyttjar alla inbyggda projektioner. Notera också portarna för att kunna hitta till ESDB-dashboarden.

## .NET klienter
I denna artikel kommer jag ge kodexempel i C#, därför kommer fokus ligga på det .NET-stöd som EventStore kommer med. Men förutom .NET kommer EventStore även med stöd för node, Java, Go och Rust.

Det finns olika .NET-klienter som kan prata med databasen. Det som särskiljer klienterna är vilket protokoll som de pratar snarare än olika API. Det finns klienter för TCP, HTTP och gRPC. gRPC är den senaste och numera den primära klienten framöver, rekommendationen är alltså att ni använder gRPC om ni bygger nytt med EventStore!

## Koppla upp mot sin EventStore-instans
När instansen är startad, via "docker-compose up", blir en dashboard tillgänglig på http://localhost:2113/. Här kan du se en mängd metrics men också, kanske mer intressant för oss, våra framtida streams. Se det som SQL Server Management Studio om du kommer från en SQL-bakgrund.

[Bild på ES-dashboard]

Instansen är igång! Nästa steg är att koppla upp oss mot den via .NET-klienten. Det vi behöver är en instans av EventStoreClient och en connectionstring. Koden nedan visar en mycket enkel connection string där vi endast konfigurerar klienten att inte nyttja tls. Det finns mängder av andra inställningar att grotta ned sig i [på EventStores webbsida](https://developers.eventstore.com/clients/dotnet/5.0/connecting/connecting-to-a-server.html#eventstoreconnection).

```csharp
var client = new EventStoreClient(EventStoreClientSettings.Create("esdb://localhost:2113?tls=false"));
```

Nu ska du förhoppningsvis vara uppkopplad mot din ES-instans och därmed ha tillgång att läsa och skriva events.

# Skriva events
Grundidén med Event Sourcing är att lagra sin applikations data som en events. Dessa events har en tidsföljd och bildar eventströmmar (engelskans event streams). I EventStore skapar du därför strömmar dit du skriver dina events. Man kan i en förenklad värld jämföra en ström med en tabell.

## Strömmar är billiga
EventStore är byggt för att hantera mängder av strömmar. Ur prestandaskäl finns det inget negativt med att skapa en ström, tvärtom så rekommenderas det att använda många strömmar eftersom det får trevliga effekter som att de t.ex. blir kortare. Långa strömmar är en stor källa till prestandaproblem och komplex logik. När en ström blir "för lång" kommer man ofta in på ämnet snapshotting som kan vara svårt att få till bra. 

Håll nere storleken på era strömmar, var inte rädd att skapa nya. **Strömmar är billiga!** Ett konkret tips är en ström per entitet.

## Namngivning av strömmar
Namngivning har betydelse, EventStore kategoriserar nämligen dina strömmar med allt till vänster om det första bindestrecket ("-"). Om du till exempel har strömmarna

- User-1
- User-2
- User-42

skapar Eventstore en ny ström med namn $ce-User som innehåller alla events för användare med ID 1, 2 och 42. Ett vanligt use case för detta är man vill prenumererar på user-events, oavsett vilken användare eventet rör. Dessa kategoriseringar, eller projektioner som det heter, kan prenumereras på precis som vilken ström som helst. 

> Strömmar som börjar med $ är systemströmmar. Det är fritt fram att nyttja dem precis som dina egna strömmar. $all är t.ex. den ström som innehåller alla events som nånsin lagrats.

> "ce" står för **c**ategory **e**vent. Läs mer om vilka systemprojektioner som finns [här](https://developers.eventstore.com/server/v20.10/docs/projections/system-projections.html).

För att summera så är rekommendationen att namnge sina events med bindestreck för att få med kategorisering. Nån form av standard skulle kunna vara formatet "Vad-Vilken" (User-1).

## Lagra events med .NET-klienten
Här nedanför följer en funktion som ansvarar för att lagra events. Den får in typ av entitet (t.ex. user), entitetens ID samt en kollektion av domänevents som skall lagras. 

Från domäneventen skapas en array av typen EventData, detta är en EventStore typ. Vi nyttjar sedan klienten skapad i [avsnittet om att koppla upp sig](#koppla-upp-mot-sin-eventstore-instans). Notera att event lagras som en byte-array och det är upp till oss att serialisera på valfritt sätt.

```csharp
public async Task Add(string entity, string entityId, IReadOnlyCollection<IDomainEvent> events)
{
    var eventData = events
        .Select(e => new EventData(Uuid.NewUuid(), e.GetType().Name, Encoding.UTF8.GetBytes(JsonConvert.SerializeObject(e))))
        .ToArray();

    await _client.AppendToStreamAsync(
        $"{entity}-{entityId}",
        StreamState.Any,
        eventData, configureOperationOptions: (o) =>
        {
            // Konfigurera timeouts med mera.
        });
}
```
> IDomainEvent implementeras av typer som kan serialiseras. Helt vanliga databärande klasser alltså.

> StreamState.Any säger att strömmen inte behöver finnas. Om den inte finns kommer den skapas. Möjliga värden är Any, NoStream och StreamExists.

# Läsa events
Vi börjar med kod denna gång. I sin enklaste form behöver vi veta namnet på strömmen, den får vi genom att slå ihop typ av entitet och dess ID, vilket är en implementationsdetalj i exemplet.

För att läsa en ström från början anropar vi *ReadStreamAsync* framlänges från start. Vi gör också en kontroll på att strömmen finns och om den inte finns returnerar vi den tomma listan.

```csharp
public async Task<IReadOnlyCollection<ResolvedEvent>> GetEventStream(string entity, string entityId)
{
    var state = _client.ReadStreamAsync(
        Direction.Forwards,
        $"{entity}-{entityId}",
        StreamPosition.Start);

    var readState = await state.ReadState;

    if (readState == ReadState.StreamNotFound)
    {
        return new List<ResolvedEvent>();
    }

    return await state.ToListAsync();
}
```

> ResolvedEvent innehåller eventets metadata samt eventet i sin serialiserade form och måste därför deserialiseras.

## Stream directions
Du kan läsa en ström framlänges eller baklänges med hjälp av *Direction*. Framlänges är nog den enklaste och vanligaste formen och används typiskt när du vill läsa upp ditt objekt i kronologisk ordning, från det äldsta till det yngsta. Baklänges används ofta när man vill hitta det senaste eventet av något slag, t.ex. den senaste snapshoten. 

### Ett par exempel
- Läsning från första till sista eventet görs via *Direction.Forwards* i kombination med *StreamPosition.Start*. 
- Hitta det senaste eventet av något slag görs via *Direction.Backwars* i kombination med *StreamPosition.End*
- Läsa från en given position framåt görs via *StreamPosition.FromStreamRevision(...)* i kombination med *Direction.Forwards*.

# Prenumerera på strömmar
Ett annat sätt att läsa från strömmar är att prenumerera på dem. Med prenumerationer kan du få reda på ett event så fort det har persisterats i databasen. Det fungerar som en pub sub. Detta är en kraftfull funktion med många användningsområden som t.ex. att lägga meddelanden på köer i integrationssyfte, skicka e-post som en reaktion på att något hänt, loggning och mycket mer.  

I koden nedan visar jag på hur man, till konsollen, kan logga ut alla events som lagras. Vi anger att vi vill börja lyssna på slutet av strömmen, dvs bara nya events. Det finns också en eventhanterare som loggar ut info kring eventet och slutligen ett filter. I detta exemplet nyttjar vi ett regular expression "Event", det betyder att eventhanteraren bara mottar events som innehåller "Event". Andra möjligheter vi har är *ExcludeSystemEvents* samt *Prefix*.

```csharp
public static async Task ConsoleLogAllEvents()
{
    await _client.SubscribeToAllAsync(Position.End, async (subscription, evnt, cancellationToken) =>
    {
        Console.WriteLine($"{Environment.NewLine}{evnt.Event.EventType} appended{Environment.NewLine}");

        await Task.CompletedTask;
    }, filterOptions: new SubscriptionFilterOptions(EventTypeFilter.RegularExpression("Event")));
}
```

## Läsmodeller och catch up subscriptions
Jag har redan nämnt ett par användningsområden men vill här belysa ett till. Med hjälp av subscriptions kan vi skapa läsmodeller. Det fina med subscriptions är att du kan ange en position att lyssna från. *Position.End* ger dig live-events, dvs nya events. *Position.Start* ger dig alla events från början *innan* du börjar få live-events. Detta kallas ofta för **catch up subscriptions**. Nu har du möjligheten att projicera dina events till en eller flera läsmodeller samt hålla modellerna uppdaterade i realtid när nya events skrivs. Oerhört kraftfullt!

# Sammanfattning
Jag hoppas ni fått en bra grundläggande bild om EventStore och framförallt vad man kan göra i dess .NET-klient. Koden som är inklippt kommer från mitt [labbrepo](https://github.com/Hagsten/EventStoreDBPlayground). Notera *labbrepo*. EventStore är kraftfullt och kommer med många bra funktioner. Det som slog mig när jag började hacka är hur lite kod som behövs för att få upp en, om än naiv, implementation mot databasen. Det som sparas är domänevents, det behövs ingen mappning till ett datalager och heller ingen relationsmappning. Känslan är att man kan fokusera på domänen och dess regelverk och inte så mycket databasen.

Tack för mig!