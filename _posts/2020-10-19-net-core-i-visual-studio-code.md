---
title: "Utveckla .NET applikationer i Visual Studio Code"
date: 2020-10-19
author: Andreas Hagsten, systemutvecklare
tagline: "Den här bloggposten är en enkel steg för steg-post för att komma igång med utveckling av .NET core i Visual Studio Code. En kollega berättade nyligen att han precis hade gått över från Visual Studio till Visual Studio Code som sin primära editor när det kommer till .NET-applikationer. Jag hade koll på att det var klart möjligt men trodde inte att det var så pass moget att gå över till helt. Det lät intressant och det började givetvis klia i fingrarna."
header:
  overlay_image: https://www.infozone.se/wp-content/uploads/2020/10/close-up-of-hands-contemporary-website-developer-man-typing-and-code-picture-id1167467556.jpg
categories:
  - blog
tags:
  - microsoft
  - systemutveckling
---
För att utveckla med .NET core i Visual Studio Code behövs till att börja med givetvis Visual Studio Code som du kan ladda hem [här](https://code.visualstudio.com/download), du behöver även [.NET Core SDK och Runtime](https://dotnet.microsoft.com/download).

# Installera tillägg
VSCode är en text editor snarare än en IDE – dock med det lilla extra! Med editorn följer många bra funktioner som auto complete, intelliSense och inbyggt GIT-stöd. Men för att få ut maximalt av editorn behöver du installera tillägg (och många finns det…). Ett tillägg som måste installeras för att komma vidare är [C# extension](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csharp) som kan hämtas via föregående länk eller via ”Extentions” inne i editorn.

Andra bra tillägg för att komma närmre känslan av en IDE likt Visual Studio är [.NET Core Test Explorer](https://marketplace.visualstudio.com/items?itemName=formulahendry.dotnet-test-explorer) samt [NuGet Package Manager](https://marketplace.visualstudio.com/items?itemName=jmrog.vscode-nuget-package-manager).

# Fördelar med Visual Studio Code
Vänta nu, varför vill man gå över till Visual Studio Code när Visual Studio är en så pass bra IDE? Även om denna artikel egentligen inte handlar om VSCode jämfört med Visual Studio så vill jag kort belysa några av de stora fördelarna.

- Cross platform – det betyder att du kan utveckla på Windows, macOS och Linux
- Gratis både för privat och kommersiellt bruk
- Lättviktigt och fokuserat på det dagliga systemutvecklingsarbetet som kodning, testning, debuging samt versionshantering av källkod

# Sätt upp en .NET Core applikation
Öppna upp editorn och navigera till ”Open Folder…” under ”File”-menyn. Skapa en ny mapp på valfritt ställe och välj den. Det är dags att skapa ett .NET core projekt. Som beskrivet tidigare behövs [.NET Core SDK och Runtime](https://dotnet.microsoft.com/download).

Öppna en ny terminal (via ”Terminal”-menyn) om en inte redan är öppen. Skriv in ”dotnet new webapi” för att skapa en ny applikation via mallen för ett WebAPI,  och tryck Enter.

Om det är första gången du gör detta kommer det installeras en del mjukvaror som behövs för att kompilera och debugga applikationen, t.ex. OmniSharp och Razor Language Server. Det ser ut ungefär så här:

```powershell
Installing C# dependencies...
Platform: win32, x86_64
Downloading package 'OmniSharp for Windows (.NET 4.6 / x64)' (35884 KB).................... Done!
Validating download...
Integrity Check succeeded.
Installing package 'OmniSharp for Windows (.NET 4.6 / x64)'
Downloading package '.NET Core Debugger (Windows / x64)' (42010 KB).................... Done!
Validating download...
Integrity Check succeeded.
Installing package '.NET Core Debugger (Windows / x64)'
Downloading package 'Razor Language Server (Windows / x64)' (50489 KB).................... Done!
Installing package 'Razor Language Server (Windows / x64)'
Finished
```

Om du öppnar en C#-fil kommer du känna igen färgkodningen från t.ex. Visual Studio, så kallad syntax highlighting. Är du observant ser du även funktionen ”code lens” (0 referenses) precis som i Visual Studio.

![Färgkodning i C# samt funktionen code lens](https://www.infozone.se/wp-content/uploads/2020/10/syntax_codelens.png)

Det ser rätt trevligt och bekant ut, eller hur? Om du börjar skriva lite kod märker du snabbt att det finns intellisense precis som du är van vid och om du håller musen över en funktion får du information densamma, som du får i Visual Studio.

![Trevligt med intellisense](https://www.infozone.se/wp-content/uploads/2020/10/intellisence.png)

# Debugging i Visual Studio Code
Nu har du förhoppningsvis fått upp ett webbprojekt och hunnit klicka runt lite i filerna. Det är dags att köra igång projektet och testa debuggern. Klicka på symbolen för debug i vänstermenyn. Här måste du skapa en ”launch.json”-fil, det gör du enklast genom att klicka på länken ”create a launch.json file” och välj ”.NET Core” i menyn som dyker upp. Efter detta är du redo att köra igång.

![](https://www.infozone.se/wp-content/uploads/2020/10/configure_debug.png)

Låt filen som genereras vara som den är, det intressanta för vår del nu är att det nu finns en debugknapp högst upp i vänsterpanelen. Sätt en brytpunkt någonstans i Get-funktionen och starta debuggern (via F5 eller klick på knapp). Webbläsaren bör ladda in https://localhost:5001, men där finns inget. Navigera istället till https://localhost:5001/weatherforecast för att komma till brytpunkten.

![](https://www.infozone.se/wp-content/uploads/2020/10/debug-e1601558240871.png)

I vänsterpanelen finns bekanta ting som brytpunkter, lokala variabler och bevakade egenskaper, det finns även en Debugkonsoll där uttryck kan skrivas och inspekteras. Att lägga till bevakade egenskaper, så kallade ”watches” fungerar på samma sätt som i Visual Studio (t.ex. genom att högerklicka på variabeln…).

För att stega i koden används bekanta tangenter som t.ex. F10, F11 samt F5. Bekant är ordet.

# Enhetstester i Visual Studio Code
Det sista jag tänkte gå igenom är hur du kör enhetstester, här rekommenderar jag tillägget [.NET Core Test Explorer](https://marketplace.visualstudio.com/items?itemName=formulahendry.dotnet-test-explorer).

![Enhetstester i Visual Studio Code](https://www.infozone.se/wp-content/uploads/2020/10/unittests.png)

Testbiblioteket som används är xUnit, det och tillägget ovan är det enda som behövs för att finna projektets alla enhetstester. Text-explorern tillåter en köra testerna i alla nivåer – med eller utan debugger. En liten bonusfunktion är att kontextmenyn har alternativ som ”Run Tests in context” och ”Debug Tests in context” vilket kör det test som markören står i, eller alla test i en klass om markören står utanför en testmetod – tjusigt!

Jag hoppas du fått upp ögonen för att det faktiskt inte är en hög tröskel att börja använda Visual Studio Code för dina .NET-core projekt. Tänker mig att jag ska ge editorn en ärlig chans i ett riktigt projekt nu, kanske kommer en mer ingående artikel på det temat i framtiden.
  
