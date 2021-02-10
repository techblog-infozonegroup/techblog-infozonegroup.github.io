---
title: "Node och javascript, fixa tydliga funktionssignaturer"
date: 2021-02-10
author: Fredde Johnsson, systemutvecklare
tagline: "Som C#/.NET-utvecklare så har jag brottats med effektivitetsproblem när jag kodar javascript/node. Häromdagen hittade jag dock ett sätt att deklarera funktioner för att göra det lite tydligare för konsumenter av metoden vilka typer av inparametrar som förväntas och vad funktionen returnerar."
header:
  overlay_image: https://raw.githubusercontent.com/techblog-infozonegroup/resources.techblog-infozonegroup/main/node-and-javascript-method-signatures/pexels-jorge-jesus-614117.jpg
  teaser: https://raw.githubusercontent.com/techblog-infozonegroup/resources.techblog-infozonegroup/main/node-and-javascript-method-signatures/nodejs-vscode-tooltip-teaser.png
categories:
  - blog
tags:
  - node
  - nodejs
  - javascript
  - Visual Studio Code
---
Sedan en tid tillbaka jobbar jag i ett kundteam där en stor del av systemen är skrivna i node/javascript. Själv kommer jag från .NET-sidan där kodandet till 90+% gjordes i C# och Visual Studio. I uppdraget nu så är C# utbytt mot node/javascript och Visual Studio utbytt mot Visual Studio Code (VS Code). I den här posten tänkte jag ta upp ett, för mig, stort problem, nämligen javascript och dess avsaknad av typning i funktionssignaturer. Detta får till följd att VS Code inte kan presentera IntelliSense med den informationen som jag känner mig van vid från Visual Studio. 

# Problem
Teamet som jag jobbar i har utvecklat ett stort antal Azure Functions i node/javascript. Det är sammantaget en ganska stor kodbas, en hel del beroenden mellan moduler etc. Det är inte något fel med det, men ett problem är att man måste läsa en hel del kod för att förstå typning av inparametrar till funktioner. Iallafall måste jag det, men det kanske beror på ovana i node/javascript i den här tillämpningen.

En ganska vanlig syn i node/javascript-kod är:

```javascript
function someFunction(firstParameter, secondParameter) {
    ...
    let someValue = firstParameter + secondParameter;
    ...
    
    return someValue;
}

function otherFunction(...) {
    ...
    let val = someFunction(1,2); // Might cause runtime error
    ...
}

function yetAnotherFunction(...) {
    ...
    let anotherValue = someFunction(1,'test'); // Might also cause runtime error
    ...
}
```

Det är alltså fullt gitigt javascript-kod, båda anropen till someFunction är korrekta och det är användningen av firstParameter och secondParameter som bestämmer vilka typer parametrarna har. 

Det här ser ut enligt nedan i VS Code, vilket är helt korrekt då båda parametrarna är av "typen" any:

{: .center}
![VS Code tooltip](https://raw.githubusercontent.com/techblog-infozonegroup/resources.techblog-infozonegroup/main/node-and-javascript-method-signatures/vs_code_tooltip.png)

{: .center}
![VS Code intellisense](https://raw.githubusercontent.com/techblog-infozonegroup/resources.techblog-infozonegroup/main/node-and-javascript-method-signatures/vs_code_intellisense.png)

Det här skapar lätt förvirring och relativt svårtolkad kod. Man måste läsa koden i someFunction för att på ett någotsånär korrekt sätt förstå typerna på parametrarna. Felaktiga typer in till funktionen resulterar i dom flesta fallen i körtidsfel, vilket i min smak är alldeles för sent.

Hur löser vi det här? Hur gör vi funktionssignaturen tydligare och hur får vi bättre hjälp av VS Code?

# Lösning
Genom att utnyttja något som i javascript kallas för [object destructuring](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment#object_destructuring) tillsammans med default-värden på properties i objekten så kan man få mycket bättre stöd i verktyget.

Vi tittar på exemplet nedan:

```javascript
function someFunction({ firstParameter = 0, secondParameter = '' } = {}) {
    ...
    let someValue = firstParameter + secondParameter;
    ...
    
    return someValue;
}

function otherFunction(...) {...}

function yetAnotherFunction(...) {...}

```

Vi ser här att funktionen **someFunction** har ändrats till att ta ett objekt som inparameter, där egenskaperna också tilldelas default-värden, **firstParameter = 0** respektive **secondParameter = ''**. Dessutom tilldelas hela objektet något som kan liknas vid ett default-värde.

Det kan kännas som ett ganska klumpigt och invecklat sätt att skriva såhär, men uppsidan är i mitt tycke stor. VS Code ger oss nu mycket mer hjälp:

{: .center}
![VS Code tooltip, typed, resolved-typed](https://raw.githubusercontent.com/techblog-infozonegroup/resources.techblog-infozonegroup/main/node-and-javascript-method-signatures/vs_code_tooltip_w_types.png)<br/>
*Signaturen för someFunction nu mycket tydligare*

{: .center}
![VS Code tooltip, resolved-typed](https://raw.githubusercontent.com/techblog-infozonegroup/resources.techblog-infozonegroup/main/node-and-javascript-method-signatures/vs_code_tooltip_w_types_chain.png)<br/>
*Ger även mer info om funktion som anropar den "typade" funktionen*

På den översta bilden ser vi direkt vad someFunction förväntar sig för inparametrar, en sträng och ett tal. På den andra bilden ser vi att "typningen" följer med och funktionen yetAnotherFunction får även den en tydligare signatur där returvärdet är en sträng.

Det finns fler uppsidor enligt nedan:

```javascript
function someFunction({ firstParameter = 0, secondParameter = '' } = {}) {
    ...
    let someValue = firstParameter + secondParameter;
    ...

    return someValue;
}

function yetAnotherFunction() {

    let firstParameter = 1;
    let secondParameter = 'test';

    // Matching parameters by property names
    let anotherValue = someFunction({ firstParameter, secondParameter });   

    // Order is not important
    anotherValue = someFunction({ secondParameter, firstParameter });     

    // Utilizing default value for firstparameter ( = 0 )  
    anotherValue = someFunction({ secondParameter });  

    // The whole object utilizing default (firstParameter = 0, secondParameter = '')                    
    anotherValue = someFunction();      

    // Equal to above, utilizing default value for the object                                    
    anotherValue = someFunction({});                                        

    return anotherValue;
}
```

> Noterar att det är trots ovan konstruktioner finns inget som hindrar anrop till funktioner med inparametrar av andra typer än dom som "deklarerats", det finns alltså ingen kompilering eller transpilering som säger ifrån.

Jag hoppas att det här kan underlätta för er som är vana vid hårt typade språk, där funktioner och metoder har signaturer med typade inparametrar och returvärden. För min del är tydlighet viktigt när man programmerar, dels för mig själv när jag skriver kod men speciellt viktigt i lägen då jag ska förvalta och förädla redan skriven kod.
