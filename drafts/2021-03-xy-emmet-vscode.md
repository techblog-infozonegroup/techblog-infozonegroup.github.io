---
title: "Emmet i Visual Studio Code"
date: 2021-03-10
author: Andreas Hagsten, systemutvecklare
tagline: "Hej hopp!"
header:
  overlay_image: https://www.infozone.se/wp-content/uploads/2020/03/programmering-i-team.jpg
categories:
  - blog
tags:
  - systemutveckling
  - Emmet
  - VSCode
  - IDE
---

Emmet är ett produktivitetsverktyg för webbutvecklare som tar kodsnuttar, så kallade "snippets", till en helt ny nivå! Emmet är primärt riktat för dem som frekvent arbetar med HTML och CSS. I denna artikel tänker jag visa hur du kan använda Emmet i VSCode. Häng med!

# Installation
Emmet följer med Visual Studio Code vilket gör att du inte behöver installera någonting. Däremot kan du behöva skruva på några parametrar. TBD

# Syntax
Emmet har en syntax som är snarlik CSS och bygger till mycket på de operatorer som du redan idag använder i CSS:

- \# för ID
- \. för klass
- \+ för sibling
- \> för nästa nivå

Men också andra operatorer som

- ^ för att gå upp en nivå
- \* för att duplicera ett element
- () för att gruppera element

På samma sätt som man i CSS skulle skriva följande för att applicera CSS på alla list items 

```
body > nav.main-nav li {
  /* styles */
}
```

kan man generera markup i Emmet genom

```
body>nav.main-nav>ul>li{Menu item}

//Ovanstående genererar
<body>
  <nav class="main-nav">
    <ul>
      <li>Menu item</li>
    </ul>
  </nav>
</body>
```

Rätt najs tycker jag. Det fina är att Emmet fungerar med andra filtyper än HTML, t.ex. jsx (React), php, scss, sass, less och många fler! Man kan även mappa om filtyper som det inte finns stöd för till de som har stöd, razor t.ex

```{"razor": "html"}```

## Avancerat exempel

Jag ska inte gå in på teori mer, istället bjuder jag på ett mer avancerat uttryck som innehåller många delar av Emmet-syntaxen. 

```
html>head+body>(nav.main-nav[role="navigation"]>ul>li.nav-item*3>{Menu item $})+.main-content#main[role="main"]>p{Hello world}
```

```
<html>
  <head></head>
    <body>
      <nav class="main-nav" role="navigation">
        <ul>
          <li class="nav-item">Menu item 1</li>
          <li class="nav-item">Menu item 2</li>
          <li class="nav-item">Menu item 3</li>
        </ul>
      </nav>
      <div class="main-content" id="main" role="main">
        <p>Hello world</p>
      </div>
    </body>
</html>
```