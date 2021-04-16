---
title: "CQS + functional programming = sant"
date: 2021-04-01
author: Fredde Johnsson, systemutvecklare
tagline: "CQS är ett imperativt mönster, functional programming är en deklarativ paradigm, kan man ändå kombinerar dom?"
header:
  overlay_image: https://www.infozone.se/wp-content/uploads/2020/03/programmering-i-team.jpg
categories:
  - blog
tags:
  - systemutveckling
  - cqs
  - functional-programming
---
# Outline
- Vad är functional programming? (kort, topp-5, referera till [http://blog.headlight.se/funktionell-programmering-med-csharp/](http://blog.headlight.se/funktionell-programmering-med-csharp/))
- Vad är CQS? 
  - command, 
  - query, 
  - process, 
  - query- och commandexecuter, 
  - dekorera med loggning, 
  - testning
- CQS + functional (varför? hur?)
- Exempel (Az func med DI)

# Bakgrund
I senaste kunduppdraget har kodandet varit till stor del fokuserat på serverless, e.g. Azure Functions, både node/javascript och C#. Av någon anledning är det lätt att bara kasta ihop sin kod, deploya och testa och sen är man nöjd med lösningen så länge den fungerar. Det behövs oftast inte så mycket kod för att lösa respektive problem, vilket är en av dom viktigaste styrkorna med Azure Functions och serverless. 

Men! Det finns ett stort MEN här. Det känns trots allt viktigt att hålla stringensen i sin kod, att följa ett mönster som gör koden robust och testbar och i och med det även förvaltningsbar över tid. Just nu är mitt favoritmönster CQS, [Command-Query separation](https://en.wikipedia.org/wiki/Command%E2%80%93query_separation). Det är relativt lättviktigt och jag tycker att det passar bra när man utvecklar Azure Functions. Jag gillar även paradigmen [functional programming](https://en.wikipedia.org/wiki/Functional_programming), en [deklarativ stil](https://en.wikipedia.org/wiki/Declarative_programming) att skriva källkod. Denna stil sägs ofta vara motsatsen till [imperativ programming](https://en.wikipedia.org/wiki/Imperative_programming). CQS klassas som ett imperativt mönster, så hur kan man kombinera CQS med functional då? Låt oss försöka! 

Men först lite basics rörande CQS och functional programming.

# CQS-pattern
CQS-mönstrets grund består av två grundläggande egenskaper:
- En **Query** ska **enbart returnera data**, *ALDRIG modifiera*
- Ett **Command utför en operation** och *FÅR modifiera* data

Dom här egenskaperna är lätta att förhålla sig till så länge man bygger system eller tjänster med väldigt enkla domäner och modeller. Att ha funktioner som enbart returnerar data, det vill säga queries, är oftast inte speciellt svåra bygga och hålla stringensen i. Däremot kan det vara svårt att avgränsa en modifierande operation på samma sätt som en läsande. Oftast blir det problem i samband med att kommandona är beroende av data från en eller flera queries eller i "värsta fall" andra kommandon. Vad gör man då? 

## Hantera komplicerade commands
Låt oss på något löst sätt definiera ett komplext kommando som ett kommando där en query körs som en del i kommandoexekveringen. Man kan också se ett komplext kommando som ett kommando där ett annat kommando körs som en del i kommandoexekveringen. Om man tillåter sådana här kommandon, alltså komplexa, så känns det som att CQS skulle kunna tillämpas i mer eller mindre alla tänkbara system. Så länge man inte är beroende av en extrem prestanda så tror jag läget är lugnt. *Med det menar jag INTE att CQS lider av prestandaproblem, då sådana problem oftast inte beror på valt mönster.*

En enkel googling ledde mig till en post på Stack Overflow: [https://stackoverflow.com/questions/36578997/is-running-a-query-from-a-command-a-violation-of-command-query-separation](https://stackoverflow.com/questions/36578997/is-running-a-query-from-a-command-a-violation-of-command-query-separation). Man ska egentligen inte blanda commands och queries, men lösningen på det skulle kunna vara att definiera olika typer av dessa. Ett svar i SO-tråden listar tre typer av kommandon:
- Command (top-level)
- Command Strategy (mid-level)
- Data Command (direct data access)

Det är den sista typen som oftast återfinns i dom grundläggande exemplen på CQS som finns tillgängliga. Skulle man begränsa sig till denna typ av kommandon så är det svårt att använda CQS i praktiken. Om man tittar på dom två översta så är min tanke att man kan helt bortse från den översta, top-level, och istället titta närmare på 'Command Strategy (mid-level)'. Här tror jag nyckeln ligger, att man på något sätt ser ett komplext kommando som ett recept eller strategi som löser uppgiften. Min idé på lösning är att man definierar en **process**:
- En process är ett recept eller ett flöde av commands och queries som löser den önskade uppgiften.

Med den definitionen kommer man väldigt långt. Funktionen man bygger exekverar en process. Processen består av commands och queries, flödet av commands och queries specificeras i processen och flödet kontrolleras av affärslogik i processen. Strategin för att lösa uppgiften ligger i processen, varje enskild byggsten i strategin är antingen ett commando eller en query. Processen KAN returnera ett värde.

> Den uppmärksamme kanske redan nu har börjat fundera på komplicerade queries, finns dom? Mitt svar är nej. En komplex query skulle möjligtvis bestå av flera queries vilket då fortfarande inte har "lyft" komplexiteten till nivån av komplicerad query. Skulle en komplicerad query, alltså en process med många queries som exekveras, innehålla en kommandoexekvering så är queryn per definition inte längre en query. Det byter då skepnad och blir ett kommando och i det komplicerade fallet en process, enligt samma definition som ovan.
 
Vi lämnar CQS ett tag och går vidare med att titta på grunderna i funktionell programmering.

# Functional programming
Om introt till CQS var kort så kommer här en ännu kortare intro om funktionell programmering. Jag har tidigare skrivit en post om [funktionell programmering i C#](http://blog.headlight.se/funktionell-programmering-med-csharp/), där fokus ligger på en topp-5-lista med egenskaper som funktionell programmering innebär. Här kommer dom, om man inte känner för att läsa den länkade posten:
- **Pure functions**, e.g. en funktions resultat är enbart beroende av sina in-parametrar
- Rekursion före iteration
- **Referental transparency**, e.g. ett anrop till en funktion ska kunna ersättas av sitt returvärde utan att resultatet eller beteende i programmet förändras
- First-class functions/higher-order functions
- **Immutable types**, e.g. en instans av en typ är oföränderlig, dvs det är inte möjligt att förändra dess egenskaper när den en gång är skapad

Varför är tre av dom fem punkterna fetstilta? Jo, även om det är en topp-5-lista så har jag tre absoluta favoriter. Om man vill koka ner lite ytterligare så kan man försöka förhålla sig till följande:
- **Immutable types**
- **Funktioner ska INTE ha några sidoeffekter**

Läs den länkade artikeln, den ger längre förklaringar till begreppen och även kodexempel för respektive koncept.

Nu är det dags att försöka få ihop det här med CQS också.

# CQS + functional
## Bakgrund
Som antyddes ovan så är CQS klassad som ett imperativt mönster medan functional programming är en deklarativ paradigm. Eftersom dessa står mer eller mindre i motsats till varandra så kan kombinationen verka svår att få ihop. Min syn på kombinationen är att den blir imperativ och **att försöka se på den som en funktionell, deklarativ, modell handlar om att definiera tydliga gränser**. Vad menas med det, vad är en tydlig gräns i ett flöde, process som ska lösa en uppgift?

## Gränsdragning
Man skulle kunna förklara gränsdragningen väldigt kort genom:
- Varje ingående del i funktionen ska vara separat testbar, med hjälp av enhetstest

För att göra den här "definitionen" tydligare så kommer jag föra en diskussion kring följande bild, där CQS-implementationen med en process, commands och queries lever i en Azure function:

![func-cqs-process](https://user-images.githubusercontent.com/460203/114912924-ddaa4e00-9e20-11eb-8153-8caf6dd7b4f2.png)

Vi börjar med dom enkla grejor, commands och queries.

## Commands och queries
Commands och queries är dom allra minsta delarna i ett CQS-mönstrat system. Dom är båda per definition väldigt avgränsade och ett av varje skulle kunna se ut enligt:
```csharp
/*---------------------------------*/
/*-----------  COMMAND  -----------*/
/*---------------------------------*/
...
using template_az_function_cs_cqs_pattern.Domain;
...

namespace template_az_function_cs_cqs_pattern.Commands
{
    public interface ICommand<T>
    {
        Task<T> Execute(T domainModel);
    }

    public class UpdateNameCommand : ICommand<User>
    {
        private readonly string _name;

        public UpdateNameCommand(string name)
        {
            _name = name;
        }

        public async Task<User> Execute(User domainModel) => domainModel.WithName(_name);
    }
}

/*----------------------------------*/
/*------------  QUERY  -------------*/
/*----------------------------------*/

namespace template_az_function_cs_cqs_pattern.Queries
{
    public interface IQuery<TDomainModel>
    {
        Task<(bool success, TDomainModel result, int status)> Execute();
    }
    
    public class GetUserBySsnQuery : IQuery<User>
    {
        private readonly string _ssn;

        public GetUserBySsnQuery(string ssn)
        {
            _ssn = ssn;
        }
        
        public async Task<(bool success, User result, int status)> Execute()
        {
            return (true, new User(_ssn, "Nisse Hult", "Maurer"), 0);
        }
    }
}
```
> Notera att GetUserBySsnQuery är influerad av tidigare post här på bloggen, [Tupler före klasser är kanske bra](https://techblogg.infozone.se/blog/tuples-might-be-good/). 

Kommandots Execute-funktion tar också som inparameter ett User-objekt på vilken det utför den tänkta operationen, nämligen uppdatera Work-egenskapen.
Det som är värt att notera i kodexemplet ovan, som kanske skiljer sig lite från "vanlig" CQS, är att ett command faktiskt *returnerar* något, i det här fallet ett domän-objekt i form av en User. Det returnerade User-objektet är en kopia av inparametern men med annat värde på Work. User-instansen som returneras är resultatet av dess WithWork-funktion:
```csharp
namespace template_az_function_cs_cqs_pattern.Domain
{
    public class User
    {
        ...
        public string Work { get; private set; }

        public User(..., string work)
        {
            ...
            Work = work;
        }
        
        private User Clone()
        {
            return new User(..., Work);
        }

        public User WithWork(string work)
        {
            var clone = Clone();
            clone.Work = work;

            return clone;
        }
        ...
    }
}
```
Vi ser ganska enkelt att alla dom här klasserna, kommandot, frågan och domänobjektet är till 100% testbara. Med givet starttillstånd så kommer operationer och frågor ALLTID att ge förväntade resultat och INGA sidoeffekter uppkommer. Du tänker nog nu att det här är ju löjligt, inte kan man bygga ett system på det här sättet, med commands och queries på den här nivån! Det kommer bli otroligt många klasser att hålla reda på, väldigt mycket kod att skriva och väldigt "plottrigt". Jag håller med om all oro, men vågar samtidigt påstå att det kan vändas till en fördel. Låt följande sjunka in en stund:

Ponera att du har "skrivit klart" en funktion i ditt system. Funktionen uppdaterar yrket på en person. Funktionen är täckt med enhetstester och den är driftsatt, fungerar i produktion och alla är nöjda. Nu kommer ett önskemål in att systemet också ska ha stöd för att pensionera en person...
