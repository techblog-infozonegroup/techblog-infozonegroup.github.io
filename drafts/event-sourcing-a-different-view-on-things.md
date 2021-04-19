---
title: "Event Sourcing - Ett annat synsätt"
date: 2021-04-01
author: Andreas Hagsten, systemutvecklare
tagline: "Event sourcing ger dig nya infallsvinklar på din applikations tillstånd."
header:
  overlay_image: https://www.infozone.se/wp-content/uploads/2020/03/programmering-i-team.jpg
categories:
  - blog
tags:
  - systemutveckling
  - event sourcing
---
I denna artikel tänker jag ge er, som inte är bekanta med Event Sourcing, en liten annorlunda syn på en applikations tillstånd och hur man lagrar och läser information. Är du redan är bekant med Event Sourcing hoppas jag ändå att du kan ta med dig något från denna läsning.

Låt oss titta på grundkonceptet inom Event Sourcing!

# Grundkonceptet
Grundidén med Event Sourcing är att man lagrar sin data som en serie av händelser (ofta kallad "Events"). Dessa händelser kan inte i efterhand ändras, det är därför immutable. Ni kan tänka er att det är som en logg eller en journal av saker som hänt. Loggar och journaler ändrar man inte på i efterhand, utan man lägger endast till nya information. Det är även sant inom Event Sourcing och det kallas för "Append only". 

Om vi slår ihop allt detta till en konkret mening så är Event Sourcing i sin grund "An immutable, append only, stream of events". 

För att få en komplett bild av en patients tillstånd så måste läkaren titta igenom alla händelser i journalen. Samma taktik används i Event Sourcing. För att läsa upp sin applikations tillstånd så tittar vi på alla events i ordning och först när vi når det senaste eventet har vi fått en korrekt bild, vilket ofta är den bild som man ser lagrad i en traditionell databas, t.ex en SQL-databas.

[skiss]

Att läsa upp sin applikations tillstånd innebär rent konkret, och kodtekniskt, att varje event appliceras via en eventhanterare. Kommer i denna artikel att visa kodsnuttar från en applikation som hanterar en fruktkorg. Det känns som en fysisk domän som är greppbar och enkel att komma på mer eller mindre vettiga krav för. Här nedan är en så kallad event hanterare för händelsen att ett äpple har lagts till i vår fruktkorg. Bjuder även på hanteraren för päron! I detta fall bryr vi oss om vilket frukt det är och dess vikt.

```csharp
private void Apply(AppleAddedEvent e)
{
    _things["apples"].Add(e.Id);
    _weights.Add(e.Id, e.Weight);
}

private void Apply(PearAddedEvent e)
{
    _things["pears"].Add(e.Id);
    _weights.Add(e.Id, e.Weight);
}
```

Det du får på köpet med denna stil är att du får ett bra fokus på vad som ska göras givet att nånting sker. Det blir enklare att hålla nere komplexiteten på affärslogik och tillståndsförändring eftersom du bara behöver fokusera på en förändring. Jag säger inte att eventhanterare inte kan bli komplexa, det jag säger är att de blir fokuserade vilket gör det enklare att hålla nere komplexiteten.

# Projektioner och läsmodeller
Kodexemplet här ovanför ligger i en klass vid namn "CurrentThings". Denna klass har som uppgift att ge svaret på vilka saker som just nu ligger i din fruktkorg samt innehållets totala vikt. Här spelar det ingen roll vilken typ av äpple som ligger i korgen, vilka färger frukterna har eller om de är färska eller ruttna. Detta kallas ofta för en projicering, eller "projection" på engelska. Är du insatt i DDD är det sannolikt att dina läsmodeller är projektioner.

En projicering är en specifik synvinkel på eventströmmen (journalen). Events "spelas upp", i ordning, genom dessa eventhanterare och projiceringens tillstånd tar form och solidifieras när sista eventet har applicerats. 

För att slå ner spiken så bjuder jag på en annan infallsvinkel på eventströmmen. Om du vill veta om en speficik frukt är ätbar så kan du skapa en projicering som lyssnar på fruktevents och i händelse av att frukten har ruttnat så är den inte ätbar, i alla andra fall är den det. Detta är såklart ett förenklat exempel, men du förstår.

```csharp
private void Apply(IDomainEvent e)
{
    Value = e switch
    {
        FruitDecomposedEvent => false,
        _ => Value
    };
}
```

## Domänaggregat
I Domän Driven Design pratar man ofta om Aggregatrötter [TODO: extern länk]. Dessa är också projiceringar av events! I detta fall med en frukkorg är korgen en aggregatrot. Vi kallar den för Basket. En korg kan endast skapas från en eventström (eller den tomma strömmen) via en fabriksmetod "Replay". Av namnet att döma spelar vi upp alla events som funktionen anropas med. Replay delegerar vidare till respektive Apply-funktion som applicerar eventet och dess data på aggregatet.

Den uppmärksamme ser att vi har två publika metoder, AddFruit och GrabAThing. Dessa metoder innehåller domänlogik. Än mer intressant är att dessa metoder skapar events och kör dessa genom respektive Apply metod; samma tillvägagångssätt som i Replay-metoden. Samma sätt att göra tillståndsförändringar alltså. Detta är viktigt. Om någon annan metod än Apply gör tillståndsförändringar kommer dessa gå förlorade när du läser upp aggregatet nästa gång.

```csharp
// Viss kod är borttagen för läsbarhetens skull
public class Basket
{
    public static Basket Replay(ICollection<IDomainEvent> events)
    {
        var basket = new Basket();

        foreach (var e in events)
        {
            basket.Apply((dynamic)e);
        }

        return basket;
    }    

    public void AddFruit(IFruit fruit)
    {
        EnsureThatFruitNotAlreadyAdded(fruit);

        IDomainEvent ev = fruit switch
        {
            Apple => new AppleAddedEvent(fruit.Id, fruit.Weight, fruit.FruitCondition),
            Pear => new PearAddedEvent(fruit.Id, fruit.Weight, fruit.FruitCondition),
            _ => new UnknownFruitAddedEvent(fruit.Id, fruit.Weight, fruit.FruitCondition)
        };

        _events.Add(ev);

        Apply((dynamic)ev);
    }
    
    public void GrabAThing(string thingId)
    {
        EnsureThatThingIsInTheBasket(thingId);

        var theThing = _things.Single(x => x.Id == thingId);

        var ev = theThing is IFruit ? new FruitGrabbedEvent(thingId) : new ThingGrabbedEvent(thingId);

        _events.Add(ev);

        Apply((dynamic)ev);
    }

    private void Apply(AppleAddedEvent e)
    {
        _things.Add(new Apple(e.Id, e.Weight, e.FruitCondition));
    }

    private void Apply(PearAddedEvent e)
    {
        _things.Add(new Pear(e.Id, e.Weight, e.FruitCondition));
    }

    private void Apply(ThingGrabbedEvent e)
    {
        var theThing = _things.SingleOrDefault(x => x.Id == e.Id);

        _things.Remove(theThing);
    }

    private void EnsureThatFruitNotAlreadyAdded(IFruit fruit)
    {
        if (_things.Any(x => x.Id == fruit.Id))
        {
            throw new System.Exception("Fruit already added...");
        }
    }

    private void EnsureThatThingIsInTheBasket(string thingId)
    {
        var theThing = _things.SingleOrDefault(x => x.Id == thingId);

        if (theThing is null)
        {
            throw new System.Exception("no no, someone already took it out");
        }
    }
}

```

## Ingen data tas bort

Med projiceringar kan du se på din eventström på många olika sätt och svara på många olika frågor. Frågor som inte alls behöver ha varit kända när systemet byggdes. Du kan svara på frågan "vilka äpplen låg i korgen den 3e april 2018 kl 08:30". Om detta inte var ett ursprungligt krav och du lagrade informationen i en SQL-databas hade det blivit svårt att ge svaret på den frågan. Hemligheten ligger i att ingen data tas bort, någonsin. Event Sourcing är en, som nämnt, "append only, immutable stream of events". Det finns inga DELETE-statements. Det finns inga UPDATE-statements. Den senare är faktiskt en implicit DELETE då vi skriver över föregående värden med nya värden, således är det en DELETE på de gamla värdena.

Detta är oerhört kraftfullt!

# Hantering av Event Streams

# Snapshots

# Sammanfattning
