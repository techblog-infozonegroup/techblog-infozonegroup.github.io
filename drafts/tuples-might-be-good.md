---
title: "Tuples före klasser för 'privat bruk'"
date: 2021-03-12
author: Fredde Johnsson, systemutvecklare
tagline: "För att skapa tydlighet och stringens mellan olika lager i ett system används förhoppningsvis typade retur-objekt mellan dessa. På så sätt upprätthålls en bra fasad. Låt oss kolla på hur man skulle kunna använda tupler istället för instanser av klasser för att uppnå samma sak."
header:
  overlay_image: https://raw.githubusercontent.com/techblog-infozonegroup/resources.techblog-infozonegroup/main/tuples-might-be-good/pexels-markus-spiske-1089438.jpg
  teaser: https://raw.githubusercontent.com/techblog-infozonegroup/resources.techblog-infozonegroup/main/tuples-might-be-good/teaser.jpg
categories:
  - blog
tags:
  - systemutveckling
  - dotnet
  - c#
---
Som grund för den här post tittar vi på AfterRefactor-projektet i  `master`-branchen i Github-repot [här](https://github.com/Fjeddo/HappyPathSadPathErrorHandling.CSharp/tree/master/AfterRefactor). I branchen `usage_of_tuples_between_services` [här](https://github.com/Fjeddo/HappyPathSadPathErrorHandling.CSharp/tree/usage_of_tuples_between_services/AfterRefactor) har klasserna ServiceResult och HttpResult ersatts med tupler.

Låt oss titta på skillnaderna och några små hjälpsamma funktioner.

# Med eller utan klasser
## HttpService och ***HttpResult***
Vi börjar med att titta på [`HttpService`](https://github.com/Fjeddo/HappyPathSadPathErrorHandling.CSharp/blob/master/AfterRefactor/Infrastructure/HttpService.cs) och dess `async Task<HttpResult> Get`-funktion, före tuple-användning, tillsammans med [`HttpResult`](https://github.com/Fjeddo/HappyPathSadPathErrorHandling.CSharp/blob/master/AfterRefactor/Infrastructure/HttpResult.cs):

```csharp
///////////////////////////////////////
// from HttpService.cs
public class HttpService
{
    // ctor with DI excluded here
    
    public async Task<HttpResult> Get(string url)
    {
        try
        {
            var response = await _httpClient.GetAsync(url); 

            var body = JObject.Parse(await response.Content.ReadAsStringAsync());
            var status = response.StatusCode;
            var success = response.IsSuccessStatusCode;

            return new HttpResult(success, body, status);
        }
        catch (InvalidOperationException e)
        {
            _log.LogError("Returning a HttpStatusCode.BadRequest", e);
            
            return new HttpResult(false, null, HttpStatusCode.BadRequest);
        }
        catch (...)
        {
            ... 
        }
        // more catches exist but excluded here
    }
}

///////////////////////////////////////
// from HttpResult.cs
public class HttpResult
{
    public HttpResult(bool success, JObject body, HttpStatusCode status)
    {
        Success = success;
        Body = body;
        Status = status;
    }

    public bool Success { get; }
    public JObject Body { get; }
    public HttpStatusCode Status { get; }
}
```

Vi ser ovan att vi returnerar ALLTID ett HttpResult-objekt i och med 
```csharp
// on success
return new HttpResult(success, body, status);

// on failure
return new HttpResult(false, null, HttpStatusCode.BadRequest);
```
oavsett om http-anropet lyckas eller inte och det ska vi såklart fortsätta med även i fallet när vi nyttjar tupler istället.

## HttpService som använder ***tuple***
Vi tittar nu på hur det skulle kunna se ut om vi använder tupler istället:

```csharp
public class HttpService
{
    // ctor with DI excluded here

    public async Task<(bool success, JObject body, HttpStatusCode statusCode)> Get(string url)
    {
        try
        {
            var response = await _httpClient.GetAsync(url);

            var body = JObject.Parse(await response.Content.ReadAsStringAsync());
            var statusCode = response.StatusCode;
            var success = response.IsSuccessStatusCode;

            return (success, body, statusCode);
        }
        catch (InvalidOperationException e)
        {
            _log.LogError("Returning a HttpStatusCode.BadRequest", e);

            return ErrorResult(HttpStatusCode.BadRequest);
        }
        catch (HttpRequestException e)
        {
            _log.LogError("Returning a HttpStatusCode.BadGateway", e);

            return ErrorResult(HttpStatusCode.BadGateway);
        }
        // more catches exist but excluded here
    }

    private static (bool, JObject, HttpStatusCode) ErrorResult(HttpStatusCode statusCode) => (false, null, statusCode);
}
```
Det som kan vara värt att belysa här är:
- funktionssignaturen är förändrad till `async Task<(bool success, JObject body, HttpStatusCode statusCode)>`, returnerar nu en tuple med namngivna properties
- `return (success, body, statusCode)` visar hur en tuple skapas och returneras
- `return ErrorResult(HttpStatusCode.BadRequest)` nyttjar hjälpmetoden `ErrorResult` för default-värdeshanteringen på success och body, `false` resp `null`

Hur påverkar det här mot konsumenterna av `HttpService`? Låt oss titta på före och hur det skulle kunna se ut efter:
```csharp
///////////////////////////////////////
// CustomerService.cs in master branch, before tuple refactor
public async Task<ServiceResult<Customer>> GetCustomerById(int id)
{
    var response = await _httpService.Get($"https://reqres.in/api/users/{id}");

    if (response.Success)
    {
        return new ServiceResult<Customer>(0, new Customer(response.Body));
    }
    ...
    return new ServiceResult<Customer>((int) response.Status, null);
}

///////////////////////////////////////
// CustomerService.cs handling a tuple return value from HttpService
public async Task<ServiceResult<Customer>> GetCustomerById(int id)
{
    var (success, body, statusCode) = await _httpService.Get($"https://reqres.in/api/users/{id}");
    
    if (success)
    {
        return new ServiceResult<Customer>(0, new Customer(body));
    }
    ...
    return new ServiceResult<Customer>((int) status, null);
}
```

Det vi kan se här är att tuple-returen från HttpService direkt kan tilldelas lokala variabler vilket ger en ganska elegant syntax i `if(success) ...` och `new Customer(body)`.

> Notera att CustomerService ovan fortfarande returnerar `Task<ServiceResult<Customer>>`, men låt oss gå vidare nedan med tuple även här.

## CustomerService som använder ***tuple***
Vi hoppar över att titta på hur [`CustomerService`](https://github.com/Fjeddo/HappyPathSadPathErrorHandling.CSharp/blob/master/AfterRefactor/Services/CustomerService.cs) ser ut i master-branchen, när den hanterar [`ServiceResult`](https://github.com/Fjeddo/HappyPathSadPathErrorHandling.CSharp/blob/master/AfterRefactor/Services/ServiceResult.cs), eftersom den väldigt mycket liknar exemplet ovan. Låt oss istället gå direkt på hur den ser ut vid nyttjande av tupler, tillsammans med [`Customers.cs`](https://github.com/Fjeddo/HappyPathSadPathErrorHandling.CSharp/blob/usage_of_tuples_between_services/AfterRefactor/Customers.cs) som konsumerar servicen:

```csharp
///////////////////////////////////////
// CustomerService.cs
public class CustomerService
{
    // ctor with DI excluded here

    public async Task<(Customer customer, int status)> GetCustomerById(int id)
    {
        var (success, body, statusCode) = await _httpService.Get($"https://reqres.in/api/users/{id}");
        
        if (success)
        {
            return _(new Customer(body));
        }

        _log.LogWarning("Something did not work out correctly");

        return _((int)statusCode);
    }

    static (Customer customer, int status) _(int status) => (null, status);
    static (Customer customer, int status) _(Customer customer) => (customer, 0);
}

///////////////////////////////////////
// Customers.cs 
[FunctionName("GetCustomer")]
public async Task<IActionResult> Run([HttpTrigger(AuthorizationLevel.Function, "get", Route = "api/customers/{id}")] HttpRequest req, int id, ILogger log)
{
    log.LogInformation("C# HTTP trigger function processed a request.");

    var (customer, status) = await _customerService.GetCustomerById(id);

    return status == 0 
        ? new OkObjectResult(customer) 
        : new StatusCodeResult(status);
}
```

Här ser vi, i `CustomerService`:
-  två default-värdeshanteringar, överlagrade med minimalt textuellt avtryck 
- dom tre intressanta värdena från _httpService.Get används på ett väldigt tydligt sätt: 
   - kontroll om lyckat http-serviceanrop via `if(success)` 
   - parsning av body vid konstruktion av en Customer-instans `new Customer(body)`
   - retur av felkod i form av http-statuskoden `statusCode`

I `Customers` ser vi att tuple-hanteringen "smittar" av sig på `GetCustomer` i [Customers.cs](https://github.com/Fjeddo/HappyPathSadPathErrorHandling.CSharp/blob/usage_of_tuples_between_services/AfterRefactor/Customers.cs). Smittan är dock inget som gör skada utan snarare tvärtom. I mina ögon blir den väldigt ren och tydlig. 

> För att kunna returnera på det sättet som görs i Customers måste man se till att man har en tillräckligt hög LangVersion, 9 eller senare, satt i csprojfilen, genom `<LangVersion>9</LangVersion>` i `Project/PropertyGroup`-taggen.


# Sammanfattning
Det är inte alltid som tupler är lämpliga att använda, det kan i vissa fall bli svårare att läsa koden. För att det ska bli mer läsbart i konsumenten av tuplen så är tupler med namngivna properties mer eller mindre ett måste, iallafall i tillämpningen som visas i den här posten. **Namngivna properties i tupler stöds i C#7 och senare**.

En avgörande punkt för att man överhuvudtaget ska överväga att nyttja tupler på det här sättet är man INTE förlorar stödet för debugging eller ökar risken för fel i runtime. Man nyttjar fortfarande hård typning och alla dess fördelar.

Inspirationen till att känna på tupler på det här sättet uppkom i samband med användning av *object destructuring i javascript*, en av javascripts absolut snyggaste kodkonstruktioner. Läs mer om det [här](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment) och [här](https://techblogg.infozone.se/blog/node-and-javascript-method-signatures/).

Hoppas det här kan inspirera till att massera kod och se hur det blir efteråt! För min del är kodmassage ett av dom klart bästa sätten att lära mig nya saker, nya konstruktioner i språken och försöka förbättra existerande kod och implementationer.
