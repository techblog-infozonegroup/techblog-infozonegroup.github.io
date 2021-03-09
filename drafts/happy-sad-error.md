---
title: "Stabilisera genom att ta kontrollen på din happy, sad och error path"
date: 2021-03-10
author: Fredde Johnsson, systemutvecklare
tagline: "Kod som är svår att förvalta är lätt att skriva, men det betyder inte att det är svårt att skriva kod som är mycket lättare att förvalta. Om man håller koll på sina 'happy paths', 'sad paths' och sin felhantering så har man kommit en bra bit påväg!"
header:
  overlay_image: https://raw.githubusercontent.com/techblog-infozonegroup/resources.techblog-infozonegroup/main/happy-sad-error/wide_traffic_light.jpg
  teaser: https://raw.githubusercontent.com/techblog-infozonegroup/resources.techblog-infozonegroup/main/happy-sad-error/teaser-traffic_light.jpg
categories:
  - blog
tags:
  - systemutveckling
  - förvaltning
---
Den här posten bygger på egna erfarenheter och upplevelser vad gäller förvaltning av kod eller försök att skapa förvaltningsbar kod. Slutsatserna är mina egna, en del kanske inte stämmer överens med totalt korrekt koddesign eller designmönster. Målet med posten är dock att vara helt mönsteragnostisk i bemärkelsen att tankarna kan appliceras oberoende av valt mönster. 

Vill man sortera in idéerna i något fack så skulle dom passa väldigt bra i S:et i S.O.L.I.D. Läs om SOLID här [https://en.wikipedia.org/wiki/SOLID](https://en.wikipedia.org/wiki/SOLID).

# Bakgrund och problem
Jag har vid några tillfällen tidigare skrivit poster i ämnet förvaltning av kod, läs här om [gammal kod](http://blog.headlight.se/ar-gammal-kod-alltid-dalig-kod/) och här om [val av mönster](http://blog.headlight.se/alla-dessa-val-av-monster/). Erfaranheten av gammal kod, nyskriven kod, kod skriven av både juniora och seniora utvecklare, kod som jag själv har skrivit, är att man ganska ofta kan hitta konstruktioner som är en genväg eller ett avsteg från det valda mönstret. Om flera utvecklare har deltagit i kodskrivandet, vilket är det absolut vanligaste, så ser man ganska tydligt olika stilar och särdrag i koden som gör att man kan se tydliga skillnader. Koden man skriver har något som kan liknas vid vanlig handstil, varje individ har en unik stil.

**Inget ont om olika stilar! Jag uppskattar verkligen skillnader i stilar och för min del är det ett sätt att utvecklas i min egen stil att skriva kod.**

Det som jag däremot har problem med är att ibland hamnar man i situationer där man INTE KAN ändra kod utan att stora delar av funktionen i aktuellt system "går sönder". Det kan handla om beroenden mellan olika komponenter, klasser eller moduler, men jag vill påstå att det i dom flesta fallen beror på *ett icke avgränsat ansvar hos enskilda komponenter*. Effekterna av detta gör att komponenter får osunda förhållanden till sina omkringliggande diton. Vad är det som gör förhållandena osunda? Jo, det handlar väldigt ofta om olika retur-vägar i komponenten, vad returnerar komponenten vid lyckats jobb, ej lyckat jobb och i felsituationer?

För att formalisera bakgrunden till posten så skulle man kunna koka ner problemen ovan till:

**Oberoende av valt designmönster, se till att ha en robust**
- **happy path**
- **sad path**
- **felhantering**

Jag skulle vilja ta med följande två påståenden in i avsnitten med kodexempel:

- har man koll på dom här tre punkterna så kommer man höja förvaltningsbarheten i den skrivna kod avservärt
- känslan är att det är väldigt lätt att göra avsteg från ovan när man utvecklar serverless

# Exempel
Jag tänkte visa en Azure Function App skriven i C#-version. Den är implementerad i ett, vad jag tror, ganska typiskt mönster för en serverless funktion.

## C#
Så här skulle en implementation kunna se, som i sig är liten och hanterbar, men om man låter den leva och växa en tid framöver, alltså få fler funktioner, så kan den bli svår att hantera ganska snart:

```csharp
public class CustomerService
{
    private readonly HttpClient _httpClient;
    private readonly ILogger _log;

    public CustomerService(HttpClient httpClient, ILogger<CustomerService> log)
    {
        _httpClient = httpClient;
        _log = log;
    }
    
    public async Task<Customer> GetCustomerById(int id)
    {
        try
        {
            var response = await _httpClient.GetAsync($"https://reqres.in/api/users/{id}");
        
            if (response.IsSuccessStatusCode)
            {
                var body = JObject.Parse(await response.Content.ReadAsStringAsync());
                return new Customer(id, $"{body["data"]["first_name"]} {body["data"]["last_name"]}", 0);
            }

            _log.LogWarning("Something did not work out correctly");
            return null;
        }
        catch (Exception e)
        {
            _log.LogError(e, "Something blew up");
            throw;
        }
    }
}
```

Vid en första anblick kan den se bra ut. Vi tittar efter:
- den utnyttjar dependency injection, HttpClient och ILogger
- den har byggt någon slags felhantering i och med try-catch
- den har bra stöd för loggning om något skulle gå fel

Det som gör den här implementation en smula svår att hantera är dom tre utgångarna från funktionen, en *happy path* där ett kundobjekt returneras, en *sad path* där null returneras och sen har vi en felhantering där felet som fångats loggas och kastas vidare till anroparen.

Om vi skulle vilja använda servicen och metoden ovan skulle det kunna se ut ungefär enligt:
```csharp
[FunctionName("GetCustomer")]
public async Task<IActionResult> Run([HttpTrigger(AuthorizationLevel.Function, "get", Route = "api/customers/{id}")] HttpRequest req, int id, ILogger log)
{
    log.LogInformation("C# HTTP trigger function processed a request.");

    try
    {
        var customer = await _customerService.GetCustomerById(id);
        if (customer == null)
        {
            return new NotFoundResult(); // Really?
        }
    
        return new OkObjectResult(customer);
    }
    catch (Exception e)
    {
        log.LogError(e, "Catching a exception");
        return new InternalServerErrorResult();
    }
}
```
Vad händer här? Anropet till servicen kan ge lyckat resultat, null eller så kan ett fel kastas, dvs precis samma utseende som inne i servicen. Så länge det här är den enda funktionen som ska byggas i det här api:et så kan den såklart hålla, men så fort vi ska lägga till en ny funktion, api-endpoint, så hamnar vi i ett lite svårare läge. Vi kanske måste lägga till ytterligare ett anrop till http-api:et som servicen konsumerar via http-klienten? Då måste vi även hantera dess returer, happy och sad paths och kastade fel. Ska vi inte börja med att bryta ut den interaktionen till ett bättre ställe och göra den robust?

Låt oss arbeta igenom dom tre punkterna som är ämnet för posten. 
Vi börjar bakifrån och försöker få kontroll på felhanteringen i servicen som tvingas fram av http-klienten. Vi gör det genom att bryta loss en ny service vars enda ansvar är att se till att göra ett http-anrop, hantera resultatet och felen som kan uppkomma. 

När vi har kontroll på felhanteringen är vi renare i CustomerSerive och kan fokusera på det vi egentligen är intresserade av, dvs om resultatet var lyckat eller inte:
```csharp
public class CustomerService
{
    private readonly HttpService _httpService;
    private readonly ILogger _log;

    public CustomerService(HttpService httpService, ILogger<CustomerService> log)
    {
        _httpService = httpService;
        _log = log;
    }

    public async Task<ServiceResult<Customer>> GetCustomerById(int id)
    {
        var response = await _httpService.Get($"https://reqres.in/api/users/{id}");

        if (response.Success)
        {
            return new ServiceResult<Customer>(0, new Customer(response.Body));
        }

        _log.LogWarning("Something did not work out correctly");
        return new ServiceResult<Customer>((int) response.Status, null);
    }
}
```
Vi ser:
- införande av en HttpService (se nedan)
- try-catch är borta, ersatt med en ren response.Success-check som en följd av att
- resultatet från Get-anropet i den nya HttpServicen är något annat är tidigare
- returen från servicen är ett ServiceResult-objekt (se nedan)

> Notera att vi i CustomerService förlitar oss på att data från HttpService är korrekt, följer det format som vi förväntar oss, när vi "parsar" det genom att skapa ett kund-objekt utifrån response-body.

Resultatet från HttpService ser ut enligt:

```csharp
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

I mina ögon är CustomerService nu lite renare, vi har en tydligt happy path och en tydlig sad path. Vad gäller error handling så sköts den isolerat och robust inne i HttpService genom följande:
```csharp
public HttpService(HttpClient httpClient, ILogger<HttpService> log)
{
    _httpClient = httpClient;
    _log = log;
}

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
    catch (HttpRequestException e)
    {
        _log.LogError("Returning a HttpStatusCode.BadGateway", e);

        return new HttpResult(false, null, HttpStatusCode.BadGateway);
    }
    catch (TaskCanceledException e)
    {
        _log.LogError("Returning a HttpStatusCode.RequestTimeout", e);

        return new HttpResult(false, null, HttpStatusCode.RequestTimeout);
    }
    catch (Exception e)
    {
        _log.LogError("Returning a HttpStatusCode.InternalServerError", e);

        return new HttpResult(false, null, HttpStatusCode.InternalServerError);
    }
}
```
En väldigt viktig detalj att notera här är att **servicen returnerar ALLTID ett giltigt objekt till sin konsument**. Det kommer *inte att kastas några fel* från den här servicen, dvs man har tagit full kontroll över sin happy path, sad path och sin felhantering. Användningen av servicen förenklas alltså markant.

> Dom tre specifika felen som fångas är enligt [dokumentation från Microsoft](https://docs.microsoft.com/en-us/dotnet/api/system.net.http.httpclient.getasync?view=netcore-3.1).

Om vi tittar kort på hur själv ingången till funktionen nu ser ut, med den nya retur-typen från CustomerService, så ser vi att även den blir mycket enklare att förstå:
```csharp
public class Customers
{
    private readonly CustomerService _customerService;

    public Customers(CustomerService customerService)
    {
        _customerService = customerService;
    }

    [FunctionName("GetCustomer")]
    public async Task<IActionResult> Run([HttpTrigger(AuthorizationLevel.Function, "get", Route = "api/customers/{id}")] HttpRequest req, int id, ILogger log)
    {
        log.LogInformation("C# HTTP trigger function processed a request.");

        var customer = await _customerService.GetCustomerById(id);

        return customer.State == 0 
            ? new OkObjectResult(customer.Data) 
            : new StatusCodeResult(customer.State);
    }
}
```
> Ja, vi antar att customer-objektets state-property innehåller en giltig http-status-kod i fallet då det är skilt från 0 (noll).

## Övriga kodkommentarer
Båda lösningarna, före och efter refaktor, finns att [titta på här](https://github.com/Fjeddo/HappyPathSadPathErrorHandling.CSharp). Man kan sammanfatta jobbet med refaktoreringen som gjorts såhär:
- felhantering i form av att fånga fel just där dom kastas gör beroenden sundare
- returnera ett tydligt resultat vid lyckad exekvering/invokering
- returnera ett tydligt resultat vid ej lyckad exekvering/invokering
- returnera samma typ i alla ovan fallen för att förenkla för konsumenterna

=> **En konsument ska inte behöva veta om mer än kontraktet som dess beroende exponerar. Kontraktet bör innehålla en tydlig happy path, en tydlig sad path OCH gömma implementationsdetaljer kring felhantering**.

# Slutord
Hoppas en del av ovanstående resonemang kan hjälpa dig och ditt team i det dagliga arbetet med utveckling och förvaltning av kod. Det krävs oftast inte så stora ombyggnationer för att stabilisera existerande kod och funktion, det är egentligen bara frågan om att bryta isär till mindre delar att hantera och på så sätt tydliggöra ingående delars ansvar.

Jag vill även, ännu en gång, poängtera att tankarna beskrivna här i posten inte är knutna till teknik, mönster eller språk. Idén till posten föddes faktiskt i samband med att en förändring skulle göras i en Vue-applikation skriven i javascript, alltså ganska långt ifrån exemplet jag använder här, en Azure Function backend i C#.
