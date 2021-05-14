---
title: "Fem tips hur hantera inkommande requests i en Azure Function"
date: 2021-05-01
author: Fredde, systemutvecklare
tagline: "Att hantera inkommande requests, aktiveringar, på ett bra sätt kan spara mycket tid i samband med felsökning och strävan efter en robust aktivering. Här kommer fem tips på hur man kan hantera det som är en Azure Functions 'ansikte utåt'."
header:
  overlay_image: https://www.infozone.se/wp-content/uploads/2020/03/programmering-i-team.jpg
categories:
  - blog
tags:
  - systemutveckling
  - azure-functions
  - serverless
---
# Outline
- kort intro vad som ska lösas
  - länk till repo
  - länk till Az func triggers
  - länk till Az func output binding
- model binding
- functioninvocationfilter
- request-object -> något annat, lämna inkommande gränssnitt bakom dig
- returnera 'rätt'


# Model binding
Den här featuren är nästan lite hemlig. Många av er kommer säkert från en ASP.NET MVC-bakgrund och är vana vid att kunna binda inkommande request direkt till ett [POCO](https://en.wikipedia.org/wiki/Plain_old_CLR_object).
När man skapar en Azure Function med en http-trigger i Visual Studio så ser funktionen som mallen ger ut enligt:

```csharp
[FunctionName(nameof(ModelBindingFunction))]
public static async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Function, "get", "post", Route = null)] Person personReq,
    ILogger log)
{
    log.LogInformation("C# HTTP trigger function processed a request.");

    var name = personReq.Name;
    var age = personReq.Age;

    return new OkObjectResult(new {name, age});
}
```

Det inkommande request-objektet är av typen HttpRequest och man börjar direkt att försöka läsa av query-parametrar och request-bodyn. Det är såklart inte fel, speciellt inte i det här fallet när funktionen tillåter både http post och get.

Om vi skulle haft en funktion som bara tillåter http post så kan man direkt binda inkommande `HttpRequest req` till ett objekt. I exemplet nedan sköter ramverket deserialiseringen till ett Person-objekt åt dig:

```csharp
[FunctionName("Function1")]
public static async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Function, "get", "post", Route = null)] Person personReq,
    ILogger log)
{
    log.LogInformation("C# HTTP trigger function processed a request.");

    var name = personReq.Name;
    var age = personReq.Age;
    ...
}
```
```csharp
public class Person
{
    public string Name { get; set; }
    public int Age { get; set; }
}
```

Om inkommande http post request-objekt är ett json-objekt enligt nedan, så kommer `name`- och `age`-propertyna ha dom förväntade värdena:
```json
{
    "Name": "Kalle",
    "Age": 10
}
```

