---
title: "Undersökning av ref-keyword i C#"
date: 2021-08-31
author: Fredde Johnsson, systemutvecklare
tagline: "I den här posten tittar vi på hur man kan använda nyckelordet ref i C# och vad det har för effekter på funktioners parametrar. Text och kod finns i form av en .NET Interactive/Jupyter Notebook."
header:
  overlay_image: https://user-images.githubusercontent.com/460203/129669260-65dc36a5-2f02-444e-b1d2-36065504a8ce.jpg
  teaser: https://user-images.githubusercontent.com/460203/115826217-d4b21180-a40a-11eb-894a-e3e367bbe140.png
categories:
  - blog
tags:
  - systemutveckling
  - c#
---
# Introduktion
Dom flesta av dom som programmerar C# har säkert stött på nyckelordet `ref` vid något tillfälle? Man har kanske inte funderat så mycket kring dess betydelse utan helt enkelt bara rättat till det fel som C#-kompilatorn sporttar ur sig om man råkar glömma `ref` framför inparametern till funktionen som nyttjar nyckelordet. Det har säkert gått bra i dom flesta fallen, men ibland kan det vara bra att fundera en stund kring vad det har för effekter på koden och systemet. I den här posten tittar vi närmare på `ref`-nyckelordet i C# och hur det fungerar.

> All kod och tillhörande skrift finns i en .NET Interactive Notebook, en Jupyter Notebook för .NET Core. Jag fick upp ögonen för det här sättet att dela exekverbar kod och text i ett och samma dokument när jag tittade på introt för Cherita Ousleys och Scott Hanselmans serie [Let's Learn C# with Scott and Cherita](https://channel9.msdn.com/Shows/Reactor/Lets-Learn-C-with-Scott-and-Cherita-at-the-Microsoft-Reactor-Part-1). Där introducerade dom även Microsofts nya site [.NET for students - Learn to code i C# programming language](https://dotnet.microsoft.com/learntocode) och i kursen använder dom .NET Interactive Notebooks för att dela kod och text, så jag var tvungen att testa! Alla prerequisites finns i början av filmen och på Learn to code-sajten.

# Resurser
Notebooken som visar användning av `ref` för **value types**, **reference types with value type behaviors** och **reference types** finns på GitHub [Fjeddo/notebooks/ref-keyword/book](https://github.com/Fjeddo/notebooks/tree/main/ref-keyword/book.ipynb). 

Dela gärna med er av era intryck och tankar kring det här sättet att dela kod och text!
