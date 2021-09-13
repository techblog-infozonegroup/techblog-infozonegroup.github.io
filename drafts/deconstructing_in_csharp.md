---
title: "Undersökning av deconstructing i C#"
date: 2021-09-10
author: Fredde Johnsson, systemutvecklare
tagline: "I tidigare post har vi tittat på tuples och hur man kan använda dessa för att returnera flera värden samtidigt utan definiera en klass. I den här posten undersöker vi hur man kan nyttja sk deconstructing för att 'veckla ut' typer på ett smidigt sätt."
header:
  overlay_image: https://user-images.githubusercontent.com/460203/129669260-65dc36a5-2f02-444e-b1d2-36065504a8ce.jpg
  teaser: https://raw.githubusercontent.com/techblog-infozonegroup/resources.techblog-infozonegroup/main/tuples-might-be-good/teaser.jpg
categories:
  - blog
tags:
  - systemutveckling
  - c#
---
# Introduktion
I javascript finns något som kallas på **destructuring** vilket innebär att man kan direkt tilldela separata variabler värdena hos egenskaper hos ett objekt. Läs om det [här](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment). I C# finns ungefär motsvarande konstruktion till hands men där kallar man det istället **deconstructing**. I det enklaste av fallen, där man använder sig av tupler, finns möjligheten till deconstructing direkt i språket. I andra fall, såsom egendefinierad referenstyper, måste man skriva lite kod för att uppnå samma resultat. Läs om deconstructing i C# [här](https://docs.microsoft.com/en-us/dotnet/csharp/fundamentals/functional/deconstruct).

Nedan följer ett antal exempel på deconstructing, som kan ge en kompakt och samtidigt läsbar och robust kod.

# Kod
## Deconstructing av tupler
Det här är det enkla fallet, där C# genom sin tuple-typ, har deconstructing-stöd i språket direkt.
```csharp
static void DeconstructTuples()
{
    var successTuple = (true, new Person(), 0);
    var (isSuccess1, successModel, successStatus) = successTuple;

    var failTuple = (false, default(Person), 123);
    var (isSuccess2, failModel, failStatus) = failTuple;

    // Ignore/discard things
    var (success1, model, _) = (true, new Person(), 0);
    var (success2, _, status) = (false, default(Person), 123);
}

public class Person { ... }
```

Här ser vi två olika fall av deconstructing, där tuplen består av **(bool, object, int)**. Booleanen indikerar success = true/false, objektet är en returnerad modell/default av modellen och heltalet är en status. Variablerna till vänster i tilldelningar, t.ex. isSuccess1, successModel och successStatus är direkt tillgängliga för användning i koden.

På dom två sista raderna i exemplet ser man även möjligheten av ignorera fält i tuplen. Detta gör man genom att ange "variabelnamnet" **_** (underscore/understreck).

## Deconstructing av klasser
I exemplet nedan ser man hur deconstructing av den egendefinierade typen ResultAsClass<T> ser ut vid användning. Syntaxen ser precis ut som i tuple-fallet ovan, med den enda skillnaden som är new-operatorn vid själva skapandet av objektet.
  
Deconstructing-beteende hos den egendefinierade typen åstadkommer man genom att implementera funktionen **Deconstruct** som syns i form av en medlemsfunktion i klassen, sist i exemplet. 
  
Ignore/discard uppnår man på samma sätt som i tuple-exemplet ovan, mha **_** (underscore/understreck).
  
```csharp
static void DeconstructResultAsClassesWithClassModel()
{
    var successClass = new ResultAsClass<Person>(true, new Person(), 0);
    var (isSuccess1, successModel, successStatuts) = successClass;

    var failClass = new ResultAsClass<Person>(false, default, 123);
    var (isSuccess2, failModel, failStatus) = failClass;

    // Ignore/discard things
    var (success1, model, _) = new ResultAsClass<Person>(true, new Person(), 0);
    var (success2, _, status) = new ResultAsClass<Person>(false, default, 123);
}
  
public class Person { ... }
  
public class ResultAsClass<T>
{
    private bool IsSuccess { get; }
    private int Status { get; }
    private T Model { get; }

    public ResultAsClass(bool isSuccess, T model, int status)
    {
        IsSuccess = isSuccess;
        Model = model;
        Status = status;
    }

    public void Deconstruct(out bool success, out T model, out int status)
    {
        success = IsSuccess;
        model = Model;
        status = Status;
    }
}
```
  
## Deconstructing av record
I C#9 infördes en ny referenstyp som har samma beteende som en värdetyp, **record**. I record byggde man även in direkt stöd för deconstruting, dvs man behöver INTE implementera någon deconstruct-metod i typen. Detta gör alltså att deconstruting uppnås som en kombination av tupler och egendefinierade typer. 
  
Exemplet nedan visar hur deconstructing ser ut, med record, tillsammans med själva record-typen sist i exemplet.
  
```csharp
static void DeconstructResultAsRecordWithClassModel()
{
    var successRecord = new ResultAsRecord<Person>(true, new(), 0);
    var (isSuccess1, successModel, successStatuts) = successRecord;

    var failRecord = new ResultAsRecord<Person>(false, default, 123);
    var (isSuccess2, failModel, failStatus) = failRecord;

    // Ignore/discard things
    var (success1, model, _) = new ResultAsRecord<Person>(true, new(), 0);
    var (success2, _, status) = new ResultAsRecord<Person>(false, default, 123);
}
 
public class Person { ... }

public record ResultAsRecord<T>(bool Success, T Model, int Status);
```
Ignore uppnås på samma sätt som i dom tidigare exemplen, genom **_** (underscore/understreck).
  
# Avslutning
Hoppas det här har väckt en smula lust att skriva kompakt och effektiv kod och att det även har väckt lusten att undersöka C#-språkets andra finurliga konstruktioner.

Komplett kod för exemplen ovan finns [här](https://github.com/Fjeddo/Deconstructing).
