---
title: "CQS + functional programming = sant (del 2/2)"
date: 2021-05-01
author: Fredde Johnsson, systemutvecklare
tagline: "Här kommer post 2 där vi tittar på kod för att försöka besvara frågan 'CQS är ett imperativt mönster, functional programming är en deklarativ paradigm, kan man ändå kombinera dom?'"
header:
  overlay_image: https://user-images.githubusercontent.com/460203/115292819-c3d97580-a156-11eb-972c-d0dad4361b75.jpg
  teaser: https://user-images.githubusercontent.com/460203/115826217-d4b21180-a40a-11eb-894a-e3e367bbe140.png
categories:
  - blog
tags:
  - systemutveckling
  - cqs
  - functional programming
  - funktionell programmering
---

# Kort recap från del 1
Innan vi tittar på kod gör vi en kort återblick på vad vi gick igenom i [första delen](https://techblogg.infozone.se/blog/cqs-plus-functional-eq-true-1_2/):

- CQS - Command Query Separation, skilda "pipor" för commands (operationer) och queries (frågor)
- Process - ett komplext kommando, ger oss möjligheten att realisera dom flesta systemen
- Funktionell programmering - en paradigm som är mindre felbenägen och robustare utan oväntade sidoeffekter

Det funktionella kommer att bli tydligare i och med att vi introducerar enhetstester och på det sättet påvisar dom funktionella egenskaperna.

# Intro
I den här posten tittar vi på några delar av koden till ett exempelsystem som "hanterar" användare. Vi kommer inte att titta på all kod utan några utvalda delar såsom:
- Process / UpdateUserProcess - en process enligt definition i del 1
- QueryExecuter och CommandExecuter - två nya komponenter, beskrivna nedan
- Query / GetUserBySsnQuery - en fråga (C**Q**S)
- Command / UpdateWorkCommand - ett kommando (**C**QS)
- Immutable domain model / User - ett domänobjekt, oföränderligt

# Översikt
Nedan bild visar det vi ska bygga och placerar in alla byggklossar på sina respektive platser i lösningen. Vi låter implementationen bo i en Azure Function:

![func-cqs-process](https://user-images.githubusercontent.com/460203/116928893-f490d300-ac5d-11eb-86a8-0f84910a30ae.png)

All källkod finns här [https://github.com/Fjeddo/Azure-function-CQS-pattern](https://github.com/Fjeddo/Azure-function-CQS-pattern). Innan vi sätter igång vill jag presentera dom ovan påannonserade spelarna **QueryExecuter och CommandHandler**.

## QueryExecuter och CommandHandler
Vi börjar kodgenomgången med dom två nyinförda komponenterna för att exekvera queries och hantera commands. Att centralisera detta ger oss möjligheter att "dekorera" anropen med loggning och felhantering. 

QueryExecuter och CommandHandler tillsammans med sitt respektive interface ser ut enligt följande:

```csharp
public interface IQueryExecuter
{
    Task<(bool success, TDomainModel result, int status)> Execute<TDomainModel>(IQuery<TDomainModel> query);
}

public class QueryExecuter : IQueryExecuter
{
    private readonly ILogger<IQueryExecuter> _log;

    public QueryExecuter(ILogger<IQueryExecuter> log) => _log = log;

    public async Task<(bool success, TDomainModel result, int status)> Execute<TDomainModel>(IQuery<TDomainModel> query)
    {
        var queryType = query.GetType();

        _log.LogInformation($"Executing {queryType.Name}");
        var result = await query.Execute();
        _log.LogInformation($"Executed {queryType.Name}");

        return result;
    }
}
```

```csharp
public interface ICommandHandler
{
    Task<TDomainModel> Handle<TDomainModel>(ICommand<TDomainModel> command, TDomainModel state);
}

public class CommandHandler : ICommandHandler
{
    private readonly ILogger<ICommandHandler> _log;

    public CommandHandler(ILogger<ICommandHandler> log) => _log = log;

    public async Task<TDomainModel> Handle<TDomainModel>(ICommand<TDomainModel> command, TDomainModel state)
    {
        var commandType = command.GetType();

        _log.LogInformation($"Handling {commandType.Name}");
        var result = await command.Execute(state);
        _log.LogInformation($"Handled {commandType.Name}");

        return result;
    }
}
```

Vi ser att queryn returnerar en tuple, på det sättet som beskrivs i [den här posten](https://techblogg.infozone.se/blog/tuples-might-be-good/). 
Implementationerna loggar typen av query och command som hanteras, frågan exekveras/kommandot hanteras och respektive resultat returneras. Här kan man tänka sig att lägga felhantering också, men i exemplet för den här posten ligger den hanteringen i processen. Vi kommer till det senare.

> Den här lösnigen liknar Decorator Pattern. Mer om det mönstret finns här [https://www.dofactory.com/net/decorator-design-pattern](https://www.dofactory.com/net/decorator-design-pattern).
 
# Process
Processen är den klass som kontrollerar flödet i en funktion, den innehåller affärslogiken och definierar vilka frågor och kommandon som ska utföras. I exemplet för den här posten implementeras en process för att uppdatera namn och arbete för en användare. Domänen består av User-objekt som identifieras med hjälp av personnummer, ssn. 

UpdateUserProcess implementerar IProcess och dessa ser ut enligt:

```csharp
public interface IProcess<TIn, TOut>
{
    Task<(bool success, TOut model, int status)> Run(TIn request);
}
```

```csharp
public class UpdateUserProcess : IProcess<UpdateUserRequest, User>
{
    private readonly IQueryExecuter _queryExecuter;
    private readonly ICommandHandler _commandHandler;
    private readonly IUserStorage _userStorage;
    private readonly ILogger _log;

    public UpdateUserProcess(IQueryExecuter queryExecuter, ICommandHandler commandHandler, IUserStorage userStorage, ILogger<UpdateUserProcess> log)
    {
        _queryExecuter = queryExecuter;
        _commandHandler = commandHandler;
        _userStorage = userStorage;

        _log = log;
    }

    public async Task<(bool success, User model, int status)> Run(UpdateUserRequest request)
    {
        _log.LogInformation($"Running process {GetType().Name}");

        try
        {
            var getUserQuery = new GetUserBySsnQuery(request.Ssn, _userStorage);

            var (success, updatedUser, status) = await _queryExecuter.Execute(getUserQuery);
            if (!success)
            {
                _log.LogInformation($"Failed getting user {request.Ssn}");

                return (false, default, status);
            }

            var updateNameCommand = new UpdateNameCommand(request.Name);
            updatedUser = await _commandHandler.Handle(updateNameCommand, updatedUser);

            var updateWorkCommand = new UpdateWorkCommand(request.Work);
            updatedUser = await _commandHandler.Handle(updateWorkCommand, updatedUser);

            return (true, updatedUser, 0);
        }
        catch (Exception exception)
        {
            _log.LogError(exception, $"Failed process {GetType().Name}");

            return (false, default, 555);
        }
        finally
        {
            _log.LogInformation($"Ran process {GetType().Name}");
        }
    }
}
```
Värt att notera i processen är att:
- Här finns hela **funktionens affärslogik för att sköta uppdateringen av användare**. Affärslogiken hanterar både lyckade och misslyckade uppdateringar, t.ex. då den eftersökta användare inte finns.
- Alla **beroenden som processen har injiceras** i konstruktorn. IoC-konfigurationen återfinns i [Startup.cs](https://github.com/Fjeddo/Azure-function-CQS-pattern/blob/master/az-function-cs-cqs-pattern/Startup.cs). 
- UserStorage injiceras också, och passas vidare ner till dom klasser som behöver ha tillgång till den.
- Här finns en basal **felhantering**. Man skulle kunna tänka sig att underliggande komponenter, commands och queries, kastar specifika undantag och varje sådant skulle kunna hanteras här, loggas och och översättas för att returnera något bra uppåt. Det är viktigt att hålla stringens och en bra struktur på felhantering för att underlätta framtida felsökning och underhåll. Läs mer om **happy-, sad- och error-paths [här](https://techblogg.infozone.se/blog/happy-sad-error/)**.

## En funktionell process?
Hur kan man se till att få fram den funktionella paradigmen i processen ovan? Det som primärt ställer till det för oss är processens alla beroenden vilket gör det svårt att uppfylla dom viktigaste egenskaperna i funktionell programmering. Vi inser snabbt att vi får tänka lite utanför ramarna och försöka se till att uppnå en nivå som är tillräckligt bra.

Låt oss begränsa strävan mot en funktionell process genom att ta kontroll över omgivningen. Det som direkt borde dyka upp i tankarna då är *enhetstester*. Om vi bygger enhetstester för processen så MÅSTE vi ta kontroll över dess beroenden. Kan vi då få den att passa in i den funktionella paradigmen? Svaret är enligt mig 'JA'.

Låt oss titta på två olika enhetstester för processen:
```csharp
public class UpdateUserProcessTests
{
    private UpdateUserProcess _sut;
    private IQueryExecuter _queryExecuter;
    private ICommandHandler _commandHandler;

    public UpdateUserProcessTests()
    {
        _queryExecuter = Substitute.For<IQueryExecuter>();
        _commandHandler = Substitute.For<ICommandHandler>();
        
        _sut = new UpdateUserProcess(
            _queryExecuter,
            _commandHandler,
            Substitute.For<IUserStorage>(),
            new NullLogger<UpdateUserProcess>());
    }

    // Happy flow -> query returnera ett User-objekt och kommandona exekveras
    [Fact]
    public async Task When_everything_is_fine_expected_queries_and_commands_should_be_invoked()
    {
        _queryExecuter.Execute(Arg.Any<GetUserBySsnQuery>()).Returns(Task.FromResult((true, new User("1234567890", "Nils", "Gold smith"), 0)));
        
        await _sut.Run(new UpdateUserRequest {Name = "Nisse", Ssn = "1234567890", Work = "Gold digger"});

        await _queryExecuter.Received().Execute(Arg.Any<GetUserBySsnQuery>());
        await _commandHandler.Received().Handle(Arg.Any<UpdateNameCommand>(), Arg.Any<User>());
        await _commandHandler.Received().Handle(Arg.Any<UpdateWorkCommand>(), Arg.Any<User>());
    }

    // Sad flow -> query hittar inget, commands exekveras INTE, resultatet är negativt och innehåller en status
    [Fact]
    public async Task When_user_is_not_found_the_commands_should_not_be_invoked()
    {
        _queryExecuter.Execute(Arg.Any<GetUserBySsnQuery>()).Returns(Task.FromResult((false, default(User), 987)));

        var result = await _sut.Run(new UpdateUserRequest { Name = "Nisse", Ssn = "1234567890", Work = "Gold digger" });

        result.Should().Be((false, null, 987));

        await _queryExecuter.Received().Execute(Arg.Any<GetUserBySsnQuery>());
        await _commandHandler.DidNotReceive().Handle(Arg.Any<UpdateNameCommand>(), Arg.Any<User>());
        await _commandHandler.DidNotReceive().Handle(Arg.Any<UpdateWorkCommand>(), Arg.Any<User>());
    }
}
```

Lyckades vi "göra processen funktionell"? Svaret är nja. Vi lyckas om vi ser till att göra den testbar, om vi tar kontroll över dess beroenden och på så sätt får den helt förutsägbar och låter den enbart vara beroende av dess inparametrar. I det här fallet är processen beroende av en request-instans som innehåller ett "filter" i form av ett personnummer och vad man vill uppdatera namn och arbete till. På det här sättet kan vi alltså se enhetstesternas användning av processen, Act-delen i testerna, som funktionella i just den här kontexten. Vi kan då säga att processen uppfyller dom flesta egenskaperna för den funktionella paradigmen. 

> En del kanske tycker det här är en massa nonsens. Vaddå funktionell process? Den är ju inte funktionell! Man kan ju göra det mesta i enhetstester! Ja, visst kan man det, men i min värld så handlar funktionell programmering om att ha kontroll på inparametrar, bygga kod som ger ett förutsägbart resultat och att kunna exekvera koden flera gånger och VARJE gång ska koden fungera och ge det resultatet tillbaka som jag förväntar mig. Vi lämnar kodens egentliga omgivning och exekverar den i en känd och kontrollerad omgivning och först då kan vi uppnå robusthet och förutsägbarhet. Det är på det sättet vi närmar oss den funktionella paradigmen, även för den komplexa processen.

 
# Query
En fråga, att läsa eller hämta data i någon källa, ska absolut vara funktionell. Det är enkelt att uppfylla många av dom egenskaper som den fuktionella paradigmen lutar sig emot. Implementationen av queryn i exempelsystemet finns att titta på [här](https://github.com/Fjeddo/Azure-function-CQS-pattern/blob/master/az-function-cs-cqs-pattern/Queries/GetUserBySsnQuery.cs). Den implementerar [IQuery](https://github.com/Fjeddo/Azure-function-CQS-pattern/blob/master/az-function-cs-cqs-pattern/Queries/IQuery.cs) för att underlätta testning och injicering av beorenden i resten av implementationen.

> Om man ska vara petig så kan man såklart diskutera vad en queries inparametrar består av. I det här fallet är frågan självklart beroende av en extern datakälla, men om dess tillstånd är känt vid exekvering så får man ändå det entydiga förutsägbara beteendet hos frågan som man strävar efter.
 
Vi ser att frågan med dess execute uppfyller:
- Pure function
- Referential transparency
- No side effects

Alla dessa egenskaper syns enklast i [enhetstesterna](https://github.com/Fjeddo/Azure-function-CQS-pattern/blob/master/Tests/GetUserBySsnQueryTests.cs) för frågan.

Vi lämnar frågan i och med detta.

# Command
Ett kommando, en uppmaning eller önskan att utföra en operation på någon enhet, entitet, känt tillstånd, är lite svårare att "få funktionell". I dom flesta exemplen på CQS-implementationer returnerar inte ett kommando något vilket gör det svårt att uppfylla t.ex. *Referential transparancy* i ett kommando. 

I det här exemplet returnerar däremot frågan det domänobjekt som är resultatet av operationen. Det här valet gjorde jag i samband med implementationen av ett kommando som persisterar ett domänobjekt i något datalager. Resultatet av anropet till datalagret var det id som entiteten fick och det var en enkel och felsäker "utökning" av ett command att låta det returnera domänobjektet tillsammans med detta id.

Om vi tittar på kommandot [UpdateWorkCommand](https://github.com/Fjeddo/Azure-function-CQS-pattern/blob/master/az-function-cs-cqs-pattern/Commands/UpdateWorkCommand.cs) så ser vi att det kommandot inte alls opererar på något externt datalager utan har bara som uppgift att uppdatera namnet på domänobjektet OCH returnera ett nytt user-objekt med det nya namnet:

```csharp
public class UpdateWorkCommand : ICommand<User>
{
    private readonly string _work;

    public UpdateWorkCommand(string work) => _work = work;

    public async Task<User> Execute(User domainModel) => domainModel.WithWork(_work);
}
```

Här syns också spår på hur domänomodellen User eventuellt är immutable. Låt oss kolla efter.

# Immutable domain model

Att ha immutable objects att jobba med i sin domän underlätta å det grövsta när det gäller att undvika fallgropar som genererar buggar och oväntade beteenden. Läs om immutable objects [här](https://en.wikipedia.org/wiki/Immutable_object) och även så kallade ValueObjects [här](https://en.wikipedia.org/wiki/Value_object). I anslutning till dessa två begrepp kan det vara intressant att läsa om [Builder pattern](https://en.wikipedia.org/wiki/Builder_pattern).

I domänen för exempelsystemet finns en typ, en User:

```csharp
public class User
{
    public string Ssn { get; }
    public string Name { get; private set; }
    public string Work { get; private set; }

    public User(string ssn, string name, string work)
    {
        Ssn = ssn;
        Name = name;
        Work = work;
    }
    
    private User Clone() => new(Ssn, Name, Work);

    public User WithWork(string work)
    {
        var clone = Clone();
        clone.Work = work;

        return clone;
    }

    public User WithName(string name)
    {
        var clone = Clone();
        clone.Name = name;

        return clone;
    }
}
```

Med en domänmodell implementerad enligt ovan så kan inte kommandot UpdateWorkCommand ha några oönskade sidoeffekter på sina inparametrar. Såklart kan operationen resultera i större "konsekvenser" såsom persistering till något datalager, genererande av ett eller flera anrop till externa tjänster som får ett förändrat tillstånd. Det inses med lätthet att dom flesta kommandona i ett system inte är idempotenta, det vill säga att dom ger samma resultat ALLA gånger dom anropas med SAMMA inparametrar.

Domänmodellen ovan är fullt testbar:

```csharp
public class UserTests
{
    [Fact]
    public void WithWork_should_clone_not_modify()
    {
        var sut = new User("1234567890", "Karo Hero", "Meisterhirte");

        var clone = sut.WithWork("alter Meisterhirte");

        clone.Should().NotBe(sut);
        clone.Should().NotBeEquivalentTo(sut);
        clone.Work.Should().Be("alter Meisterhirte");
        sut.Work.Should().Be("Meisterhirte");
    }

    [Fact]
    public void WithName_should_clone_not_modify()
    {
        var sut = new User("1234567890", "Karo Hero", "Meisterhirte");

        var clone = sut.WithName("alter Karo Hero");

        clone.Should().NotBe(sut);
        clone.Should().NotBeEquivalentTo(sut);
        clone.Name.Should().Be("alter Karo Hero");
        sut.Name.Should().Be("Karo Hero");
    }
}
```

I dom här två testerna påvisas ett User-objekts immutability. Alla egenskapers set-metoder är gömda från exponering utåt och modifiering görs genom speciella funktioner. Varje förändring av en egenskap returnerar en ny instans, en kopia men med modifierad egenskap. Instanserna är INTE samma, vilket verifieras i och med `clone.Should().NotBe(sut);` i dom båda testfallen.

# Wrap-up
Under skrivandet av den här delen insåg jag snabbt att det är svårt att på ett riktigt tydligt sätt sätta fingret på vad som kan göras funktionellt i CQS-mönstret. Den observante läsaren kanske direkt insåg att enhetstesterna skulle vara det enklaste sättet att påvisa hur man kan sammanföra dom två olika typerna av paradigmer. Allt kokade egentligen ner till att man bör skriva kod som är fullständigt testbar, enhetstester är extremt viktigt för att bygga robust kod. Om man tänker efter en gång till på sådant som gör felsökning svår, vad som får funktioner att inte fungera etc, så är det när dom beter sig på ett icke förutsägbart sätt. 

- **Funktionell programmering handlar om** att skaffa sig **förutsägbar kod**.
- **CQS handlar om** att **separera ansvar** och inte försätta sig i svårbemästrade beroenden. När en funktion i ett system "färdig", det vill säga implementerad, testad och produktionssatt, så ska **en ny funktion inte kunna förstöra dom färdiga funktionerna**.

Jag hoppas att dom här posterna gav något, om inte annat provocerade fram lite lust att utmana tanken på att kombinera CQS och funktionell programmering och om det verkligen finns något skäl att göra det. Titta gärna igenom enhetstesterna för dom olika delarna i systemet, process, query och command. Det som borde framkomma där är att faktiskt ALLA delar i ett system kan göras testbara. Att ha med sig det när man skriver kod vill jag påstå ger en mycket bättre stringens i designen och underlättar både felsökning och felavhjälpning.

All kod finns [här](https://github.com/Fjeddo/Azure-function-CQS-pattern) tillsammans med enhetstester.
