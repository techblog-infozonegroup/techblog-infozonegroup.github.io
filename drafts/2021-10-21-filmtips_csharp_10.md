---
title: ""
date: 2021-10-21
author: Fredde Johnsson, systemutvecklare
tagline: "I C# 10, som sannolikt kommer att releasas tillsammans med .NET 6 i början av november finns det ett gäng intressanta nya features."
header:
  overlay_image: https://user-images.githubusercontent.com/460203/129669260-65dc36a5-2f02-444e-b1d2-36065504a8ce.jpg
  teaser: https://user-images.githubusercontent.com/460203/115826217-d4b21180-a40a-11eb-894a-e3e367bbe140.png
categories:
  - blog
tags:
  - systemutveckling
---
# C# 10 nya features
Microsoft rullar på snabbt och stabilt med nya versioner av C#. Dom senaste versionerna har kommit ungefär en gång per år och aktuell version är C# 9. 

I samband med release av .NET 6 i början av november, på [.NET Conf 2021](https://www.dotnetconf.net/), kommer man även att släppa nästa version av C#. 
I denna versionen kommer det ett gäng intressanta features och i filmklippet nedan visar [Nick Chapsas](https://www.youtube.com/channel/UCrkPsvLGln62OMZRO6K-llg) dessa 
med tydliga och bra exempel på användning på nästan alla. Min känsla är att trenden från .NET-teamet hos Microsoft på sistone har varit att streamlina språket och att få bort "onödig" kod som man helt automatiskt bara skriver, gång på gång, för att man SKA göra det.

Kolla in Nicks klipp här **[Every feature added in C# 10 with examples](https://www.youtube.com/watch?v=Vft4QDUpyWY)**, det är väl värt 15 minuter.

Dom features som han går igenom är följande. Varning! Dom på slutet är en smula flummiga:
- Global usings
- File scoped usings
- Interpolated strings constants
- Förbättringar av lambda
- Utökad pattern matching på propertys
- Record structs
- Sealed ToString method in record type
- Förbättrad struct-typ
- Tilldelning och deklaration i deconstruct statement
- AsyncMethodBuilder
- Static abstract members

Vill man läsa om den senaste release-kandidaten av C# 10 så kan man göra det här [Announcing .NET 6 Release Candidate 2](https://devblogs.microsoft.com/dotnet/announcing-net-6-release-candidate-2/).
