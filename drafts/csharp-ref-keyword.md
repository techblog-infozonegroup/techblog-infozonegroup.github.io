---
title: "Undersökning av ref-keyword i C#"
date: 2021-08-31
author: Fredde Johnsson, systemutvecklare
tagline: "I den här posten tittar vi på hur man kan använda nyckelordet ref i C# och vad det har för effekter på funktioners parametrar. Text och kod finns i form av en .NET Interactive/Jupyter Notebook [här](https://github.com/Fjeddo/notebooks/tree/main/ref-keyword)."
header:
  overlay_image: https://user-images.githubusercontent.com/460203/129669260-65dc36a5-2f02-444e-b1d2-36065504a8ce.jpg
  teaser: https://user-images.githubusercontent.com/460203/115826217-d4b21180-a40a-11eb-894a-e3e367bbe140.png
categories:
  - blog
tags:
  - systemutveckling
  - c#
---
I den här posten tittar vi närmare på nyckelordet `ref` i C#, vad har det för effekter och hur kan man använda det. 

> All kod och tillhörande skrift finns i en .NET Interactive Notebook, en Jupyter Notebook för .NET Core. Jag fick upp ögonen för det här sättet att dela exekverbar kod och text i ett och samma dokument när jag tittade på intron för Cherita Ousleys och Scott Hanselmans serie [Let's Learn C# with Scott and Cherita](https://channel9.msdn.com/Shows/Reactor/Lets-Learn-C-with-Scott-and-Cherita-at-the-Microsoft-Reactor-Part-1). Där introducerade dom Microsofts nya site [.NET for students - Learn to code i C# programming language](https://dotnet.microsoft.com/learntocode) och genom kursen använder dom .NET Interactive Notebooks, så jag var tvungen att testa! Alla prerequisites finns i början av filmen och på Learn to code-sajten.

Notebooken som visar användning av `ref` för **value types**, **reference types with value type behaviors** och **refernce types** finns på GitHub [Fjeddo/notebooks/ref-keyword/book](https://github.com/Fjeddo/notebooks/tree/main/ref-keyword/book.ipynb). 

Dela gärna med er om era intryck och tankar kring det här sättet att dela kod och text!
