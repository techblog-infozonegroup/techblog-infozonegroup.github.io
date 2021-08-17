---
title: "Fem tips för en robust kodbas i Azure Functions"
date: 2021-06-05
author: Fredde Johnsson, systemutvecklare
tagline: "Serverless, såsom Azure Functions, bjuder in till att direkt börja knacka kod, vilket såklart är bra, men det är trots allt bra att ta det lite lugnt och tänka igenom hur man kan skapa en bra grund för att få en stringent och stabil kodbas."
header:
  overlay_image: https://user-images.githubusercontent.com/460203/120365047-446ec280-c30e-11eb-8121-60e2dca36e56.jpg
  teaser: https://user-images.githubusercontent.com/460203/120427707-b7118980-c372-11eb-85b3-b71fb2cc563f.jpg
categories:
  - blog
tags:
  - systemutveckling
  - azure-functions
  - serverless
  - azure
---
# Intro
I den här artikeln tänkte jag gå igenom min topp-5-lista med kodförbättrande grunder och/eller åtgärder. Vissa av dom är hårt knutna till Http-triggade Azure Functions, en del är applicerbara i andra tillämpningar. Till varje avsnitt finns ett exempel-projekt för att man ska kunna plocka hem och navigera koden i lugn och ro i sin egen utvecklingsmiljö. Koden för alla exempel finns här [**az-func-five-tips**](https://github.com/Fjeddo/az-func-five-tips).

Vi kommer att titta på:
- [Dependency injection (DI)](#dependency-injection-di)
- [Model binding](#model-binding)
- [Request interception (preview + basklass)](#request-interception)
- [Request/trigger interface => domain](#requesttrigger-interface--domain)
- [Stringent retur-hantering](#stringent-retur-hantering)

# Dependency injection (DI)
I andra bloggposter [här](http://blog.headlight.se/ioc-di-ramverk-eller-inte/) och [här](http://blog.headlight.se/mer-om-ioc-och-di/) har jag ventilerat min tveksamhet till att i alla lägen konfigurera och använda ett DI-ramverk. Att injicera beroenden, framförallt genom [Constructor injection](https://en.wikipedia.org/wiki/Dependency_injection#Constructor_injection), kräver egentligen inget ramverk. Sedan ganska långt tillbaka har dock .NET Core inbyggt stöd för IoC-konfiguration, vilket har gjort användningen väldigt mycket enklare. Jag tycker att .NET Core's implementation känns mycket mer lättviktig jämfört med andra ramverk och trots sin lättviktighet så är den absolut tillräcklig.

Det som jag har noterat i och med ökad erfarenhet av serverless-tekniker, framför allt Azure Functions, är att man skulle bli väldigt begränsad om man inte hade möjlighet att konfigurera funktioners beroenden med hjälp av ett ramverk. Om en Azure Function App ska innehålla flera funktioner, kanske vill man implementera något som liknar ett web api med flera olika typer av datakällor eller externa tjänster att konsumera, så anser jag att man MÅSTE ha ett sätt att konfigurera beroenden på ett bra sätt. Detta ökar förvaltningsbarheten massor och det underlättar mycket för när man vill hålla stringensen i kodbasen. 

Precis som man är van vid från ASP.NET Core's IoC-konfiguration så bygger allt på att man skapar en uppstartsklass, ofta kallad Startup. I Azure Functions låter man klassen ärva från den abstrakta klassen [FunctionsStartup](https://github.com/Azure/azure-functions-dotnet-extensions/blob/main/src/Extensions/DependencyInjection/FunctionsStartup.cs) och den aktiveras vid uppstart med hjälp av attributet `[assembly: FunctionsStartup(...)]`. 

Basklassen innehåller en abstrakt metod, `void Configure(IFunctionsHostBuilder builder)`, och en metod som man kan överrida vid behov, `void ConfigureAppConfiguration(IFunctionsConfigurationBuilder builder)`. Den förstnämda är den som ska innehålla IoC-konfigurationen:

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
> Exemplen ovan finns i utförligare form här [DependencyInjectionFunction](https://github.com/Fjeddo/az-func-five-tips/tree/master/DependencyInjection).

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

Om inkommande http post request-objekt är ett json-objekt enligt nedan, så kommer `name`- och `age`-egenskaperna ha dom förväntade värdena:
```json
{
    "Name": "Kalle",
    "Age": 10
}
```

> Exemplen ovan finns i utförligare format här [ModelBindingFunction](https://github.com/Fjeddo/az-func-five-tips/tree/master/ModelBinding).

# Request-interception
Många gånger skriver man mer eller mindre identisk kod på flera ställen i sin lösning. Det kan handla om kod för att logga inkommande anrop eller aktiveringar, det kan handla om att implementera mätpunkter för prestanda eller annan övervakning, det kan handla om felhantering. Det skulle då vara skönt att skriva den koden på ett och samma ställe och återanvända den i alla sina funktioner.

## FunctionInvocationFilter och IFunctionExceptionFilter (preview)
Tyvärr finns det inte något enkelt tillrättalagt sätt att bygga en request-response-pipeline i Azure functions, likt den OWIN-pipeline som finns i ASP.NET (MVC). Det finns däremot möjlighet att implementera ett interface för att fånga inkommande aktiveringar och även fånga svaret på väg ut ur funktionen:
```csharp
public interface IFunctionInvocationFilter : IFunctionFilter
{
  Task OnExecutingAsync(
    FunctionExecutingContext executingContext,
    CancellationToken cancellationToken);

  Task OnExecutedAsync(
    FunctionExecutedContext executedContext,
    CancellationToken cancellationToken);
}
```
Det finns ytterligare ett inteface att implementera för att fånga exceptions som kastas inne i funktionen, `IFunctionExceptionFilter`:
```csharp
public interface IFunctionExceptionFilter : IFunctionFilter
{
  Task OnExceptionAsync(
    FunctionExceptionContext exceptionContext,
    CancellationToken cancellationToken);
}
```
*Notera att det kommer att genereras kompileringsvarningar om man implementerar dom här interfacen. De är markerade som `[Obsolete("Filters is in preview and there may be breaking changes in this area.")]` och är alltså i preview. Tyvärr har dom varit det ganska länge.*

> Exempel på implementationer av dessa två interface finns här [RequestInterceptionFunction](https://github.com/Fjeddo/az-func-five-tips/blob/master/RequestInterception/RequestInterceptionFunction.cs).

## Basklass och dekorering
Man kan såklart uppnå mer eller mindre samma resultat genom att ha en metod som dekorerar själva funktionen som ska köras med förbearbetning, exekvering, efterbearbetning och eventuell exception-hantering:
```csharp
public class AnotherInterceptingFunction : InterceptingBaseFunction
{
    public AnotherInterceptingFunction(IHttpContextAccessor httpContextAccessor, ILogger<InterceptingBaseFunction> log) 
        : base(httpContextAccessor, log) { }

    [FunctionName("AnotherInterceptingFunction")]
    public async Task<IActionResult> Run([HttpTrigger(AuthorizationLevel.Function, "get", "post", Route = null)] HttpRequest req, ILogger log)
    {
        return await Execute(async () =>
        {
            log.LogInformation("C# HTTP trigger function processed a request.");

            return new OkObjectResult(new {success = true, data = DateTimeOffset.UtcNow});
        });
    }
}

public abstract class InterceptingBaseFunction
{
    private readonly IHttpContextAccessor _httpContextAccessor;
    private readonly ILogger<InterceptingBaseFunction> _log;

    protected InterceptingBaseFunction(IHttpContextAccessor httpContextAccessor, ILogger<InterceptingBaseFunction> log)
    {
        _httpContextAccessor = httpContextAccessor;
        _log = log;
    }

    protected async Task<IActionResult> Execute(Func<Task<IActionResult>> func)
    {
        try
        {
            var requestCookie = _httpContextAccessor.HttpContext.Request.Cookies["required-cookie"];
            if (requestCookie == null)
            {
                return new BadRequestResult();
            }

            var result = await func();

            _log.LogInformation("This is after the function executed");

            return result;
        }
        catch (Exception e)
        {
            _log.LogError("Exception caught", e);
            throw;
        }
    }
}
```
I exemplet ovan omsluts funktionskoden i `AnotherInterceptingFunction` av en `Execute`-funktion som ligger i basklassen `InterceptingBaseFunction`. I basklassen kontrolleras så att inkommande HTTP-anrop har med sig en cookie. Om så inte är fallet returneras direkt `400 Bad Request` och funktionskoden körs aldrig. Funktionen i basklassen hanterar även ohanterade fel, som loggas och kastas vidare i den här lösningen.

> Koden för exemplet på detta finns här [AnotherInterceptingFunction](https://github.com/Fjeddo/az-func-five-tips/blob/master/RequestInterception/AnotherInterceptingFunction.cs), tillsammans med basklassen här [InterceptingBaseFunction](https://github.com/Fjeddo/az-func-five-tips/blob/master/RequestInterception/InterceptingBaseFunction.cs).

# Request/trigger interface => domain
På samma sätt som när man utvecklar webbapplikationer i ASP.NET MVC eller med hjälp av andra ramverk så är det viktigt att inte låta beroenden i yttre gränssnitt följa med in i domänen. Det handlar egentligen om att lämna tekniken som aktiverar funktionen så fort som möjligt, konvertera nödvändiga indata till kända modeller i domänen och börja jobba där. 

För Azure Functions är det bra att försöka följa samma strategi, oavsett vilken typ av trigger som aktiverar funktionen. Exemplet nedan är ett exempel på en process, i en CQS-implementation, där vi nyttjar model binding och inkommande request används direkt för att skapa en process:
```csharp
public static class RequestToDomainFunction
{
    [FunctionName(nameof(RequestToDomainFunction))]
    public static async Task<IActionResult> Run(
        [HttpTrigger(AuthorizationLevel.Function, "post", Route = null)] UpdateUserWorkRequest updateUserWorkRequest, 
        ILogger log)
    {
        // "Leave the incoming request behind" as soon as possible, get into the domain instead
        var process = new UpdateUserWorkProcess(updateUserWorkRequest.Ssn, updateUserWorkRequest.Work);
        var (success, model, status) = process.Run();

        if (!success)
        {
            // Do some mapping of status to proper http status
            var httpStatus = status == -1 ? StatusCodes.Status404NotFound : StatusCodes.Status500InternalServerError;

            return new StatusCodeResult(httpStatus);
        }

        return new OkObjectResult(model);
    }
}

public class UpdateUserWorkRequest
{
    public string Ssn { get; set; }
    public string Work { get; set; }
}
```
```csharp
public interface IProcess<T>
{
    (bool success, T model, int status) Run();
}

public class UpdateUserWorkProcess : IProcess<User>
{
    public UpdateUserWorkProcess(string ssn, string work) { }
    public (bool success, User model, int status) Run() => (true, default, 0);
}
```
Läs mer om Azure Functions och CQS här [CQS + functional programming = sant, del 1](https://techblogg.infozone.se/blog/cqs-plus-functional-eq-true-1_2/) och [del 2](https://techblogg.infozone.se/blog/cqs-plus-functional-eq-true-2_2/).

> Källkoden för exemplet ovan finns här [RequestToDomainFunction](https://github.com/Fjeddo/az-func-five-tips/tree/master/RequestToDomain).

# Stringent retur-hantering
En extremt viktig detalj för att göra funktioner möjliga att använda är att dess konsumenter vet hur dom beter sig och varför dom i vissa fall inte returnerar det som man kan förvänta sig. Ett tråkigt men ack så effektivt sätt att få detta att fungera dokumentera varje funktions yttre gränssnitt. Med det menar jag att göra det tydligt vad en funktion vill ha för indata och vad den kan ge för svar och då handlar det både om lyckade och misslyckade anrop, det vill säga alla potentiella felkoder i retur.

Läs mer om retur-stringens här [Tupler före klasser kanske är bra?](https://techblogg.infozone.se/blog/tuples-might-be-good/) och här [Stabilisera genom att ta kontrollen över din happy, sad och error path](https://techblogg.infozone.se/blog/happy-sad-error/).

När det gäller användandet av tupler så kommer C#'s stöd för deconstruction, switch expressions och pattern matching väl till pass vid mappning från domän tillbaka ut till gränssnittet:
```csharp
public static class TuplePatternMatchingFunction
{
    [FunctionName(nameof(TuplePatternMatchingFunction))]
    public static async Task<IActionResult> Run(
        [HttpTrigger(AuthorizationLevel.Function, "post", Route = null)] UpdateUserWorkRequest updateUserWorkRequest,
        ILogger log)
    {
        var process = new UpdateUserWorkProcess(updateUserWorkRequest.Ssn, updateUserWorkRequest.Work);
        var (success, model, status) = process.Run();

        // Do some mapping of status to proper http status
        return (success, status) switch
        {
            (true, _) => new OkObjectResult(model),
            (false, -1) => new NotFoundResult(),
            (false, -999) => new StatusCodeResult(400),

            _ => new InternalServerErrorResult()
        };
    }
}
```
> Källkoden i exemplet ovan kommer från [TuplesPatternMatchingFunction](https://github.com/Fjeddo/az-func-five-tips/tree/master/TuplesPatternMatchingFunction).

# Avslutande ord
Jag hoppas att det här har gett dig som läsare något att ta med dig vid etablering av kodbaser, framför allt för Azure Functions. Att hålla sig till ett mindre antal enkla grunder kommer direkt ha positiva effekter på hur enkel koden blir att förvalta och dessutom blir det enkelt att komma ihåg hur tankarna gick vid det initiala rörmokeriet. Varför inte skriva kort i en README.MD-fil som ligger tillsammans med källkoden, där principerna och besluten beskrivs?

Tveka inte att kommentera och dela med dig av dina tankar kring hur man etablerar en stabil kodbas, speciellt i dom fallen när man är ute på ett helt nytt orört grönt fält.
