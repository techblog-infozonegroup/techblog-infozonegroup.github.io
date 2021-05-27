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
- DI
- model binding
- functioninvocationfilter
- request-object -> något annat, lämna inkommande gränssnitt bakom dig
- returnera 'rätt'

# Dependency injection (DI)
I andra bloggposter [här](http://blog.headlight.se/ioc-di-ramverk-eller-inte/) och [här](http://blog.headlight.se/mer-om-ioc-och-di/) så har jag ventilerat min tveksamhet till att i alla lägen konfigurera och använda DI. Att injicera beroenden, framförallt genom [Constructor injection](https://en.wikipedia.org/wiki/Dependency_injection#Constructor_injection), kräver egentligen inget ramverk. Sedan ganska långt tillbaka har .NET Core inbyggt stöd för DI och IoC-konfiguration, vilket har gjort användningen väldigt mycket enklare. Jag tycker att .NET Core's implementation känns mycket mer lättviktig gämfört med andra ramverk. Trots sin lättviktighet så är den tillräcklig.

Precis som man är van vid från ASP.NET Core's IoC-konfiguration så bygger allt på att man skapar en uppstartsklass, ofta kallad Startup, som ärver från den abstracta klassen [FunctionsStartup](https://github.com/Azure/azure-functions-dotnet-extensions/blob/main/src/Extensions/DependencyInjection/FunctionsStartup.cs) och triggas med hjälp av attributet `[assembly: FunctionsStartup(typeof(Startup))]`. Denna klassen innehåller en abstrakt metod, `void Configure(IFunctionsHostBuilder builder)`, och en metod som man kan överrida vid behov, `void ConfigureAppConfiguration(IFunctionsConfigurationBuilder builder)`. Den förstnämda är den som ska implementera IoC-konfigurationen:

```csharp
[assembly: FunctionsStartup(typeof(Startup))]

namespace DependencyInjection
{
    public class Startup : FunctionsStartup
    {
        public override void Configure(IFunctionsHostBuilder builder)
        {
            /////////
            // Examples, built in services
            builder.Services.AddLogging();
            builder.Services.AddHttpContextAccessor();

            /////////
            // Examples function implementation specific services
            builder.Services.AddTransient<IPersonService, PersonService>();

            /////////
            // Interfaces are not required
            builder.Services.AddScoped<SomePersonProcess>();
            builder.Services.AddSingleton<SomeUtilityClass>();

            /////////
            // Factories
            builder.Services.AddScoped(builder =>
            {
                // this is invoked when creating an instance
                return new SomePersonProcess();
            });
            
            builder.Services.AddTransient<IPersonService>(builder =>
            {
                // this is invoked when creating an instance
                return new PersonService();
            });
        }
    }
}
```

För att förbättra läsbarhet kan man såklart bygga extensionmetoder för IServiceCollection:
```csharp
public class Startup : FunctionsStartup
{
    public override void Configure(IFunctionsHostBuilder builder)
    {
        builder.Services.AddDomainServices();
        ...
    }
}

public static class DomainIoCConfiguration
{
    public static IServiceCollection AddDomainServices(this IServiceCollection services)
    {
        services.AddScoped<IPersonService, PersonService>();
        services.AddTransient<SomePersonProcess>();

        return services;
    }
}
```

# Model binding
Den här featuren är nästan lite hemlig. Många av er kommer säkert från en ASP.NET MVC-bakgrund och är vana vid att kunna binda inkommande request direkt till ett [POCO](https://en.wikipedia.org/wiki/Plain_old_CLR_object).
När man skapar en Azure Function med en http-trigger i Visual Studio så ser funktionen som mallen ger ut enligt:

```csharp
public static class Function1
{
    [FunctionName("Function1")]
    public static async Task<IActionResult> Run(
        [HttpTrigger(AuthorizationLevel.Function, "get", "post", Route = null)] HttpRequest req,
        ILogger log)
    {
        log.LogInformation("C# HTTP trigger function processed a request.");
        
        string name = req.Query["name"];

        string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
        ...
    }
}
```

Det inkommande request-objektet är av typen HttpRequest och man börjar direkt att försöka läsa av query-parametrar och request-bodyn. Det är såklart inte fel, speciellt inte i det här fallet när funktionen tillåter både http post och get.

Om vi skulle haft en funktion som bara tillåter http post så kan man direkt binda inkommande `HttpRequest req` till ett objekt. I exemplet nedan sköter ramverket deserialiseringen till ett Person-objekt åt dig:

```csharp
public static class ModelBindingFunction
{
    [FunctionName(nameof(ModelBindingFunction))]
    public static async Task<IActionResult> Run(
        [HttpTrigger(AuthorizationLevel.Function, "post", Route = null)] Person personReq,
        ILogger log)
    {
        log.LogInformation("C# HTTP trigger function processed a request.");

        var name = personReq.Name;
        var age = personReq.Age;
        ...
    }
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

# Request-interception mha FunctionInvocationFilter (preview)
qwerty
