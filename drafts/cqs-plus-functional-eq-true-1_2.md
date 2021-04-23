---
title: "CQS + functional programming = sant (del 1/2)"
date: 2021-04-01
author: Fredde Johnsson, systemutvecklare
tagline: "CQS är ett imperativt mönster, functional programming är en deklarativ paradigm, kan man ändå kombinera dom?"
header:
  overlay_image: https://user-images.githubusercontent.com/460203/115292819-c3d97580-a156-11eb-972c-d0dad4361b75.jpg

categories:
  - blog
tags:
  - systemutveckling
  - cqs
  - functional-programming
---
# Bakgrund
I senaste kunduppdraget har kodandet varit till stor del fokuserat på serverless, Azure Functions, både node/javascript och C#. Av någon anledning är det lätt att bara kasta ihop sin kod, deploya och testa och sen är man nöjd med lösningen så länge den fungerar. Det behövs oftast inte så mycket kod för att lösa respektive problem, vilket är en av dom viktigaste styrkorna med Azure Functions och serverless. 

Men! Det finns ett stort MEN här: 
- Det är viktigt att hålla stringensen i sin kod, att följa ett mönster som gör koden robust och testbar och i och med det även förvaltningsbar över tid. 

Just nu är mitt favoritmönster CQS, [Command-Query separation](https://en.wikipedia.org/wiki/Command%E2%80%93query_separation). Det är relativt lättviktigt och jag tycker att det passar bra när man utvecklar Azure Functions. Jag gillar även paradigmen [functional programming](https://en.wikipedia.org/wiki/Functional_programming), en [deklarativ stil](https://en.wikipedia.org/wiki/Declarative_programming) att skriva källkod. Denna stil sägs ofta vara motsatsen till [imperativ programming](https://en.wikipedia.org/wiki/Imperative_programming). CQS klassas som imperativ så frågan är om man trots allt kan kombinera den med deklarativ funktionell programmering? 

I den här första posten av två kommer jag fokusera på ett teoretiskt resonemang kring CQS och funktionell programmering och vad dess olika styrkor är. I post nummer två gör jag ett försök att kombinera stilarna och vi tittar på en massa kod.

# CQS-pattern
CQS-mönstrets grund består egentligen bara av två typer av objekt och dessa har varsin egenskap:
- En **Query** ska **enbart returnera data**, *ALDRIG modifiera*; <br>exempel **GetPersonBySsn => en person med givet personnummer**
- Ett **Command utför en operation** och *FÅR modifiera* data; <br>exempel **UpdatePersonWork => uppdatera en persons yrke**

En schematisk bild av CQS-mönstret skulle då kunna se ut så här:

![cqs-overview](https://user-images.githubusercontent.com/460203/115136703-dbfba880-a021-11eb-9ed4-e29a0ebacfcb.png)

I bilden syns tydligt **separationen** mellan **commands och queries**. Ett command är en egen liten isolerad enhet och så länge man inte ändrar dess implementation så ska den aldrig kunna ändra beteende. Detsamma gäller för en query, den är en egen isolerad liten enhet. Om man *bara ändrar ett command eller en query* så ska man känna sig trygg att *inget runt omkring den påverkas*. I daglig tal kan man säga att *ett kommando lever i en helt egen pipa och en query likaså*. Det är en av dom klart största styrkorna i mönstret.

Dom här egenskaperna är lätta att förhålla sig till så länge man bygger system eller tjänster med väldigt enkla domäner och modeller. Att ha funktioner som enbart returnerar data, det vill säga queries, är oftast inte speciellt svåra bygga och hålla stringensen i. Däremot kan det vara svårt att avgränsa en modifierande operation på samma sätt som en läsande. Det blir ofta problem i samband med att kommandona är beroende av data från en eller flera queries eller i "värsta fall" har man ett beroende till andra kommandon. Vad gör man då? 

## Hantera komplicerade commands
Låt oss på ett löst sätt definiera ett komplext kommando som ett kommando där en query körs som en del i kommandoexekveringen. Man kan också se ett komplext kommando som ett kommando där ett annat kommando körs som en del i kommandoexekveringen. Om man tillåter sådana här kommandon, alltså komplexa, så känns det som att CQS skulle kunna tillämpas i mer eller mindre alla tänkbara system. Så länge man inte är beroende av en extrem prestanda så tror jag läget är lugnt. *Med det menar jag INTE att CQS lider av prestandaproblem, då sådana problem oftast inte beror på valt mönster.*

En enkel googling ledde mig till en post på Stack Overflow, [https://stackoverflow.com/questions/36578997/is-running-a-query-from-a-command-a-violation-of-command-query-separation](https://stackoverflow.com/questions/36578997/is-running-a-query-from-a-command-a-violation-of-command-query-separation). Man ska egentligen inte blanda commands och queries, men lösningen på det skulle kunna vara att definiera olika typer av kommandon. Ett svar i SO-tråden listar tre typer av kommandon:
- Command (top-level)
- Command Strategy (mid-level)
- Data Command (direct data access)

Det är den sista typen som oftast återfinns i dom grundläggande exemplen på CQS som finns tillgängliga. Skulle man begränsa sig till denna typ av kommandon så är det svårt att använda CQS i praktiken. Om man tittar på dom två översta så är min tanke att man kan helt bortse från den översta, top-level, och istället titta närmare på 'Command Strategy (mid-level)'. Här tror jag man har nyckeln till framgång, att man på något sätt ser ett komplext kommando som ett recept eller strategi som löser uppgiften. 

Min idé på lösning är att man definierar en **process**:
- En process är ett recept eller ett flöde av commands och queries som löser den önskade uppgiften.

Med den definitionen kommer man väldigt långt. Funktionen man bygger exekverar alltså en process. Processen består av commands och queries och flödet av commands och queries specificeras i processen. Flödet genom processen styras av affärslogik, villkor och kontroller. 
 
Strategin för att lösa uppgiften ligger i processen, varje enskild byggsten i strategin är antingen ett command eller en query. Processen KAN returnera ett värde.

> Den uppmärksamme kanske redan nu har börjat fundera på komplicerade queries, finns dom? Mitt svar är nej. En komplex query skulle möjligtvis bestå av flera queries vilket då fortfarande inte har "lyft" komplexiteten till nivån av komplicerad query. Skulle en komplicerad query, alltså en process med många queries som exekveras, innehålla en kommandoexekvering så är queryn per definition inte längre en query. Den byter då skepnad och blir ett kommando och i det komplicerade fallet en process, enligt samma definition som ovan.

> Ytterligare en observation som den uppmärksamme läsaren kanske har gjort är att ett komplicerat kommando väldigt mycket liknar en orkestrering eller en saga. Det som skiljer ett komplicerat kommando, enligt ovan definition, från t.ex. en saga är att kommandot inte innehåller någon funktion för att göra rollback. En *saga hanterar* problemen som man måste lösa när man ska hantera *distribuerade transaktioner över servicegränser*, då varje ingående del i en saga också innehåller en så kallad *compensating action*, dvs en rollback av den aktuella aktionen. Läs mer om orchestrations och sagas här [Microservices Pattern: Sagas](https://microservices.io/patterns/data/saga.html).

Vi lämnar CQS och går vidare med att titta på grunderna i funktionell programmering.

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

Nu är det dags att försöka få ihop det här med CQS också. Det gör vi i post nummer två, som kommer senare. Länk återfinns här när den är publicerad.
