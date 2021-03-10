---
title: "Tuples före klasser för 'privat bruk'"
date: 2021-03-11
author: Fredde Johnsson, systemutvecklare
tagline: "För att skapa tydlighet och stringens mellan olika lager i ett system implementation används förhoppningsvis typape retur-objekt mellan services. På så sätt upprätthålls en bra fasad. Låt oss kolla på hur man skulle kunna använda tupler istället för att uppnå ungefär samma sak"
header:
  overlay_image: https://raw.githubusercontent.com/techblog-infozonegroup/resources.techblog-infozonegroup/main/happy-sad-error/wide_traffic_light.jpg
  teaser: https://raw.githubusercontent.com/techblog-infozonegroup/resources.techblog-infozonegroup/main/happy-sad-error/teaser-traffic_light.jpg
categories:
  - blog
tags:
  - systemutveckling
  - förvaltning
---
Som grund tar vi och tittar på [AfterRefactor-projektet i master-branchen i Github-repot här](https://github.com/Fjeddo/HappyPathSadPathErrorHandling.CSharp/tree/master/AfterRefactor). I branchen [*usage_of_tuples_between_services* här](https://github.com/Fjeddo/HappyPathSadPathErrorHandling.CSharp/tree/usage_of_tuples_between_services/AfterRefactor) har klasserna ServiceResult och HttpResult ersatts med tuples.

Låt oss titta på skillnaderna och några små hjälpsamma funktioner.

# Med eller utan klasser
## HttpService som använder ***HttpResult***
Vi börjar att titta på [`HttpService`](https://github.com/Fjeddo/HappyPathSadPathErrorHandling.CSharp/blob/master/AfterRefactor/Infrastructure/HttpService.cs) och dess `async Task<HttpResult> Get`-funktion, före tuple-användning, tillsammans med [`HttpResult`](https://github.com/Fjeddo/HappyPathSadPathErrorHandling.CSharp/blob/master/AfterRefactor/Infrastructure/HttpResult.cs):

```csharp
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

// from HttpResult.cs
public class HttpResult
{
    public HttpResult(bool success, JObject body, HttpStatusCode status)
    {
        Success = success;
        Body = body;
        Status = status;
    }

    public bool Success { get;  }
    public JObject Body { get;  }
    public HttpStatusCode Status { get; }
}
```

Vi ser ovan att vi returnerar ALLTID ett HttpResult-objekt i och med 
```csharp
...
return new HttpResult(success, body, status);
```
och 
```csharp
...
return new HttpResult(false, null, HttpStatusCode.BadRequest);
```
oavsett om http-anropet lyckades eller inte och det ska vi såklart fortsätta med även i fallet när vi nyttjar tupler istället.

## HttpService som använder ***tuple***
Vi tittar nu på hur det skulle kunna se ut om vi använder tupler istället, vilket kommer att påverka konsumenten av `HttpService` också såklart:

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
- `return (success, body, statusCode);` visar hur en tuple skapas och returneras
- `return ErrorResult(HttpStatusCode.BadRequest);` nyttjar hjälpmetoden `ErrorResult` för default-värdeshanteringen på success och body

Hur slår det här mot konsumenterna av `HttpService`? Låt oss titta på före och efter:
```csharp
// CustomerService.cs in master branch, before tuple refactor
public async Task<ServiceResult<Customer>> GetCustomerById(int id)
{
    var response = await _httpService.Get($"https://reqres.in/api/users/{id}");

    if (response.Success)
    {
        var customer = new Customer(response.Body);
        ...
}

///////////////////////////////////////
// CustomerService.cs in usage_of_tuples_between_services branch, using tuple
public async Task<(Customer customer, int status)> GetCustomerById(int id)
{
    var (success, body, statusCode) = await _httpService.Get($"https://reqres.in/api/users/{id}");
    
    if (success)
    {
        var customer = new Customer(body);
        ...
    }
}
```

Det vi kan se här är att tuple-returen från HttpService direkt kan tilldelas lokala variabler och det blir ganska elegant syntax i `if(success) ...` och `var customer = new Customer(body);`.

> Notera att även CustomerService i usage_of_tuples_between_services branch ovan returnerar en tuple `async Task<(Customer customer, int status)>`, se nedan.

## CustomerService som använder ***tuple***
Vi hoppar över att titta i detalj på hur [`CustomerService`](https://github.com/Fjeddo/HappyPathSadPathErrorHandling.CSharp/blob/master/AfterRefactor/Services/CustomerService.cs) ser ut i master-branchen, när den hanterar [`ServiceResult`](https://github.com/Fjeddo/HappyPathSadPathErrorHandling.CSharp/blob/master/AfterRefactor/Services/ServiceResult.cs) och går direkt på hur den ser ut vid användande av tuple, tillsammans med [`Customers.cs`](https://github.com/Fjeddo/HappyPathSadPathErrorHandling.CSharp/blob/usage_of_tuples_between_services/AfterRefactor/Customers.cs) som använder servicen:

```csharp
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

// Customers.cs 
```

Här ser vi:
-  två default-värdeshanteringar, överlagrade med minimalt textuellt fingeravtryck. 
- dom tre intressanta värdena från _httpService.Get används på ett väldigt tydligt sätt: 
   - kontroll om lyckat http-serviceanrop via `if(success)` 
   - parsning av body vid konstruktion av en Customer-instans `new Customer(body)`
   - retur av felkod i form av http-statuskoden `(int)statusCode`

## Customers använder också tuple
Ovan tuple-hantering i `CustomerService` "smittar" såklart av sig på funktionen i [`Customers`](https://github.com/Fjeddo/HappyPathSadPathErrorHandling.CSharp/blob/usage_of_tuples_between_services/AfterRefactor/Customers.cs) som blir väldigt ren:

```csharp
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

# Sammanfattning
Det är inte alltid som tupler är lämpliga att använda, det kan bli svårt att läsa kod, speciellt om man inte använder namngivna egenskaper på tuplen. För att det ska bli läsbart i konsumenten av tuplen så är det mer eller mindre ett måste.

Notera att man inte på något sätt förlora stöd för debugging eller ökar risken för fel i runtime. Man nyttjar fortfarande hårt typade objekt.

Hoppas det här kan inspirera till att massera kod och se hur det blir efteråt! För min del är kodmassage ett av dom klart bästa sätten att lära mig nya saker, nya konstruktioner i språken och försöka förbättra existerande kod i olika implementationer.