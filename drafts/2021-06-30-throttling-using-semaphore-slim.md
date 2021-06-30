---
title: "Kontrollera antalet parallela anrop"
date: 2021-06-30
author: Andreas Hagsten, systemutvecklare
tagline: "SemaphoreSlim - enkelt och effektivt sätt att strypa anrop mot API:er"
header:
  overlay_image: https://raw.githubusercontent.com/techblog-infozonegroup/resources.techblog-infozonegroup/main/semaphoreslim/crowd_queue.jpg
  teaser: https://raw.githubusercontent.com/techblog-infozonegroup/resources.techblog-infozonegroup/main/event-store-db/eventstore-logo.png
categories:
  - blog
tags:
  - systemutveckling
  - semaphore
  - throttling
  - dotnet 5
---

I den här bloggposten vill jag lite snabbt tipsa om en klass som, oförtjänt, verkar leva vid sidan av rampljuset. Jag pratar om [SemaphoreSlim](https://docs.microsoft.com/en-us/dotnet/api/system.threading.semaphoreslim?view=net-5.0). SemaphoreSlim är ett utmärkt verktyg när man vill ösa på med anrop mot en funktion eller ett API men samtidigt göra det under kontrollerade former.

Idén om denna post kommer från ett avsnitt från [DotNetRocks](https://dotnetrocks.com/), där de lyfter fram klassen i det inledande "Better know a framework". 

# Problemställning
Tänk er ett system som integrerar med API:er av olika slag. Det är inte helt ovanligt att moderna API:er har begränsningar när det kommer till samtidiga anrop, eller antal anrop över en tidsperiod. Säg att begränsningen är 100 samtidiga anrop men att ditt system har betydligt fler inkommande anrop än så. Du har ett problem.

Det finns såklart många lösningar på problemet, en kan vara att ha samma begränsning i ditt system eller att batcha anrop. SemaphoreSlim kan dock lösa dessa problem på ett transparent och på ett oinvasivt sätt. Det är det vi ska se närmre på.

# Kontrollera anropen

Här nedan följer en enkel bit kod som anropar ett API 10 000 gånger, parallelt. Det här API:et har en begränsning om 100 samtidiga anrop. Det är högst sannolikt att nedanstående kod kommer börja kasta fel med statuskod 429 - Too many requests.

```csharp
public static void HammerTheApiUnControlled()
{
    var client = new HttpClient();
    var input = "the string";

    Parallel.ForEach(Enumerable.Range(0, 10000), async (_, _, _) =>
    {
        var response = await client.GetStringAsync($"https://localhost:44360/reverse?text={input}");

        Console.WriteLine($"{input} --> {response}");
    });
}
```

## SemaphoreSlim
Vi kan med ett fåtal nya kodrader få ovanstående kod att fungera mycket bättre. Vi behåller all kod, dvs även de 10 000 parallela anropen mot API:et, men lägger en begränsning kring varje enskilt anrop. Vi instansierar ett objekt av typen SemaphoreSlim och begränsar den till 100 samtidiga trådar, samma begränsning som API:et har. Runt funktionskroppen inne i loopen lägger vi ```await semaphore.WaitAsync()``` och ```semaphore.Release()```. Det är allt som behövs. WaitAsync kommer se till att endast 100 trådar får tillgång till den kod som följer, fram till och med anropet till Release.

```csharp
public static void HammerTheApi()
{
    var client = new HttpClient();
    var input = "the string";

    var semaphore = new System.Threading.SemaphoreSlim(100);

    Parallel.ForEach(Enumerable.Range(0, 10000), async (_,_,_) =>
    {
        await semaphore.WaitAsync();

        var response = await client.GetStringAsync($"https://localhost:44360/reverse?text={input}");

        Console.WriteLine($"{input} --> {response}");

        semaphore.Release();
    });
}
```

Det fina med detta, utöver dess enkelhet, är att det inte är invasiv kod. Koden kommer inte förändra de gränsnitt som finns definierat i ert system. Nytjandet av SemaphoreSlim blir helt transparent för anroparen. 

# Sammanfattning
SemaphoreSlim blir kraftfullt i sin enkelhet och transparens. Lägg på ett abstraktionslager samt konfigurering så har ni snabbt en bra motor för att kontrollera samtidiga anrop mot API:er eller annan kod som kan lida av för hög frekvens av anrop.