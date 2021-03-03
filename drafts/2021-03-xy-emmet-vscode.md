---
title: "Emmet i Visual Studio Code"
date: 2021-03-03
author: Andreas Hagsten, systemutvecklare
tagline: "Öka din produktivitet i ditt webbutvecklande"
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

Emmet är ett produktivitetsverktyg för webbutvecklare som tar kodsnuttar, så kallade "snippets", till en helt ny nivå! Emmet är primärt riktat för dig som frekvent arbetar med HTML och CSS. I denna artikel tänker jag visa hur du kan använda [Emmet i VSCode](https://code.visualstudio.com/docs/editor/emmet). Häng med!

# Syntax
Emmet har en syntax som är snarlik CSS och bygger till stor del på de operatorer som du redan idag använder i CSS:

- \# för ID
- \. för klass
- \+ för sibling
- \> för nästa nivå

Men också andra operatorer som

- ^ för att gå upp en nivå
- \* för att duplicera ett element
- () för att gruppera element

På samma sätt som man i CSS skulle skriva följande för att applicera styles på alla list items 

``` css
body > nav.main-nav li {
  /* styles */
}
```

kan man generera markup i Emmet genom

```
body>nav.main-nav>ul>li{Menu item}
```

vilket genererar denna HTML

``` html
<body>
  <nav class="main-nav">
    <ul>
      <li>Menu item</li>
    </ul>
  </nav>
</body>
```

Rätt najs tycker jag. Det fina är att Emmet fungerar med andra filtyper än HTML, t.ex. jsx (React), php, scss, sass, less med mera. Man kan också mappa om filtyper som det inte finns stöd för via inställningar i VSCode. Nedan följer ett exempel för razor-filer. Inställningar hittar du genom F1 och "default settings (JSON)".

``` json
"emmet.includeLanguages": {
    "razor": "html"
}
```

## CSS
Emmet fungerar även i CSS-filer för egenskaper och dess värden. Den kortaste formen kan användas när egenskapens värde är numeriskt, t.ex padding. I dessa fall kan man helt enkelt börja skriva på egenskapens namn, eller dess förkortning, och när VSCode föreslår egenskapen du är ute efter fylla på med det numeriska värdet. Defaultenhet används om enhet ej är angiven. 

``` css
.item {
  /* mb10 */
  margin-bottom: 10px;

  /* p10 */
  padding: 10px;

  /* p10% */
  padding: 10%;
}
```

Man kan också skilja på vänster och höger led med kolon (:) och då kan man även nyttja Emmet för värden som inte är numeriska, t.ex:

``` css
.item {
  /* d:i */
  display: inline;

  /* fw:b */
  font-weight: bold;
}
```

Riktigt trevligt och en fin produktivitetsboost i ditt CSS-författande!

## Ett större exempel

Innan jag avslutar tänkte jag bjuda på en större snippet som involverar flera delar av Emmet-syntaxen. Här nedan nyttjar jag bl.a. gruppering, multiplicering, html-attribut och en räknare. 

```
html>head+body>(nav.main-nav[role="navigation"]>ul>li.nav-item*3>{Menu item $})+.main-content#main[role="main"]>p{Hello world}
```

``` html
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

Detta kanske är old news för många av er, men jag insåg alldeles nyligen att detta stöd finns i VSCode. Jag tycker att detta verkar riktigt bra och kommer testa på det nästa gång jag sitter och utvecklar en webbapplikation. 