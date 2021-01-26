---
title: "C# 9 - Records, pattern matching och mycket mer"
date: 2021-02-01
author: Andreas Hagsten, systemutvecklare
tagline: "Skapa objekt som är immutable i grunden med Records och kraftfulla nya features till Pattern matching. Det och mycket mer i denna artikel. "
header:
  overlay_image: https://www.infozone.se/wp-content/uploads/2020/10/close-up-of-hands-contemporary-website-developer-man-typing-and-code-picture-id1167467556.jpg
categories:
  - blog
tags:
  - microsoft
  - systemutveckling
---

Utökad pattern matching, ny referenstyp med värdetypssemantik som dessutom är byggd med immutability i åtanke samt möjligheten att initiera immutable properties. Detta och mycket mer i den senaste versionen av C# - 9.0.  

C# 9 släpptes i mitten av november 2020 och skeppas med .NET 5.0 och finns med i version 16.8 av Visual Studio 2019. Här har ni en lista av alla nya features https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-9.

I denna artikel tänker jag fokusera på Records, Init only setters samt pattern matching. Det kommer bli mycket kod, håll till godo!

# Records
Records är en ny referenstyp (likt klasser) men med semantik som en värdtyp (likt structs). Med det menar jag att likhet beräknas på värdena av instansens egenskaper och inte på instansens referens. Två instanser med samma värden på egenskaper ses därför som lika. Exemplet nedan demonstrerar detta. De två kandidaterna anses lika genom Equals samt operatorn "==" men inte om man jämför referensen. Dvs, ett record är en referenstyp men med semantik som en värdetyp.

{% gist 4f4a482a25728513ca4bf81e58661faa file-valueequality-cs %}
https://gist.github.com/Hagsten/4f4a482a25728513ca4bf81e58661faa#file-valueequality-cs

En annan mycket trevlig egenskap som ett Record har är att den är implicit immutable. Med det menar jag att om inget annan anges i deklarationen av ett record blir alla dess egenskaper låsta för förändring. Immutability är en mycket trevlig och ofta nödvändig egenskap som gör att objekt blir trådsäkra. Ett vanligt problem som flertrådade applikationer lider av är när flera trådar ändrar på samma objekts egenskaper vilket medför intermittenta och svårlösta buggar. Med objekt som är oföränderliga (immutable) kan en säkert skicka runt dess referens till flera trådar och en garanti kan lämnas att dess tillstånd förblir detsamma. Jag skriver en hel del om detta i en annan artikel (https://www.infozone.se/2018/11/13/lita-pa-dina-objekt-mjukvaruarkitektur-del-1/).

I exemplet nedan ser du tre sätt att deklarera ett record. Det första sättet medför implicit immutability på alla egenskaper, den andra är helt ekvivalent med den första men med explicit deklaration av "init settes". Det tredje exemplet skapar ett mutable record, dvs dess egenskaper kan förändras efter initiering.

{% gist 4f4a482a25728513ca4bf81e58661faa#file-employee-cs %}
https://gist.github.com/Hagsten/4f4a482a25728513ca4bf81e58661faa#file-employee-cs

Immutable Records är ypperliga att använda i många fall, framförallt i flertrådade applikationer av ovanstående anledning eller när dataflödet är enkelriktat genom en applikations lager (t.ex. API --> Domänlager --> Databas). Nästa applikationslager kan helt enkelt lita på objekten, något jag skriver om i https://www.infozone.se/2018/11/13/lita-pa-dina-objekt-mjukvaruarkitektur-del-1/. Ett konkret exempel på enkelriktat dataflöde är i CQS/CQRS där Commands och Queries är objekt som är enkelriktade från den som begär frågan eller kommandot ända ned till respektive hanterare (CommandHandler/QueryHandler). Svaret från en QueryHandler är ofta en läsmodell - ytterligare ett ypperligt användningsområde för records.

Ofta finns dock behovet att skapa nya objekt med ett eller flera förändrade värden. Förr innebar det att skapa nya objekt och stansa av egenskaperna en efter en. Lyckligtivs stödjer records nyckelordet "with" som möjligör att skapa nya records med samma innehåll. Det fina är dock att man ges möjligheten att förändra egenskaperna innan det nya objektet skapas. Ni kan se ett exempel på detta här då vi vill göra en förändring på en kandidat efter att denne genomgått en första utvärdering.

{% gist 4f4a482a25728513ca4bf81e58661faa#file-with-cs %}
https://gist.github.com/Hagsten/4f4a482a25728513ca4bf81e58661faa#file-with-cs

Resulatet är att "evaluatedCandidate" innehåller samma värden som "candidate" men med Evaluated satt till true.

# Pattern matching
Pattern matching är relativt nytt men har funnits i tidigare versioner av C#, i 9.0 kommer det dock en mängd nya features. Hela listan hittar ni på Microsofts dokumentation (https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-9#pattern-matching-enhancements). Detta är inte en djupgående genomgång av alla sätt att skriva pattern matching utan jag kommer helt enkelt ge exempel som nyttjar en del nya features. Här är ett exempel där en kandidats lön skall beräknas givet ett antal kriterier.

{% gist 4f4a482a25728513ca4bf81e58661faa#file-negotiatesalary-cs %}
https://gist.github.com/Hagsten/4f4a482a25728513ca4bf81e58661faa#file-negotiatesalary-cs

Kanske är denna syntax helt främmande för er, kanske har ni stenkoll och kan direkt se de nya features som kommer med C# 9. En objektiv bedömning av koden är att den är kort och konsis och en måhända mer subjektiv bedömning är att den är lättläst. Mycket av fokuset ligger på villkoren och vad resultatet av dem är och mindre fokus på syntax och andra konstruktioner.

Ni kan se 6 mönster, var och ett beskrivna nedan

{ Evaluated: false } => 0

Mönstret innehåller inga nyheter men läses "Om kandidaten inte har utvärderats ger ingen lön".

{ Age: <= 25, YearsOfExperience: <= 3 } => 200

Detta mönster tar hänsyn till flera egenskaper på kandidaten - dess ålder och erfarenhet. Detta görs med ett komma efter varje uttryck. För att detta mönster skall matcha måste båda villkoren vara uppfyllda. Mönstret läses "Om kandidaten är 25 år eller yngre och under 4 års eftarenhet ges en lön om 200".

{ YearsOfExperience: >= 3 and <= 6 } => 300

Detta mönster innehåller en nyhet - "and". Mönstret läser "Om kandidaten har mellan 3 och 6 års erfarenhet ges en lön om 300".

{ Degree: Phd } => 600

Mönstret läser "om kandidaten har en doktorsexamen ges en lön om 600". Phd är en typ, dvs ett record, en klass eller annan form av typ. Detta är också nytt i C# 9 - Type patterns.

{ Degree: Bachelor or Master } => 400

Mönstret läser "om kandidaten har en kandidat eller master examen ges en lön om 400". Även här används Type patterns tillsammans med "or" vilket likt "and" är nytt i C# 9.

_ => 0

Discard pattern. Om inget annat matchar ges lönen 0. 

Detta i sig är inget revolutionerande utan som nämnt förut ett kort och konsist sätt att skriva villkor (i jämförelse med t.ex. if-satser). En kan hävda att det är lätt att läsa, förstå och resonera kring denna typ av kod och förenklar processen att stansa av verksamhetens alla krav.