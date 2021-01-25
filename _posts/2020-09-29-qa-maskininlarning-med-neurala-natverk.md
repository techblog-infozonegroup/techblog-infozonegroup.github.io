---
title: "Q&A - Maskininlärning med neurala nätverk"
date: 2020-09-29
author: Andreas Hagsten, systemutvecklare
tagline: "Våra artiklar kring neurala nätverk i C# har lett till många intressanta läsarfrågor. I denna artikel ger vi svar på tre av dessa frågor. Fortsätt ställa intressanta frågor så ser vi till att följa upp dem med flera Q&A-artiklar!"
header:
  overlay_image: https://www.infozone.se/wp-content/uploads/2020/09/questions-and-answers-discussion-forum-picture-id1156622227.jpg
categories:
  - blog
tags:
  - systemutveckling
---
Länkar till artiklarna finner ni här:

- [Del 1](https://www.infozone.se/2017/04/06/maskininlarning-med-ett-neuralt-natverk-i-c-del-1-teorin/)
- [Del 2](https://www.infozone.se/2017/04/27/maskininlarning-med-ett-neuralt-natverk-i-c-del-2-implementation/)

# *Fråga 1*: Vad är ett minima och hur kan det vara dåligt för neurala nätverk?
Inom matematiken är definitionen ”Ett minimum för en given mängd är ett element x i denna mängd, sådant att alla andra element, som kan jämföras med x, är större än vad x är.” – Wikipedia.

I sammanhanget neurala nätverk förekommer ”minima” i den funktion, Gradient decent i detta fall, som beräknar det värde som resulterar i lägst feldifferens i resultatlagret, se bild nedan.

![Gradient decent. Orange cirkel är önskvärt värde, röd cirkel är nuvarande värde.](https://www.infozone.se/wp-content/uploads/2020/09/ew-plot-300x216-1.jpg)

Målet i detta exempel är att hitta det sanna minimat vid W = 1,5. Enligt vår [artikel](https://www.infozone.se/2017/04/06/maskininlarning-med-ett-neuralt-natverk-i-c-del-1-teorin/) hittas riktningen i koordinatsystemet genom att beräkna derivatan (dE/dW). Enligt bilden ovan behöver vi röra oss i negativ riktning för att nå det sanna minimat på W=1,5. Problemet är att om vi rör oss för långt och kommer förbi det lokala maximat vid W ~= 1,25 kommer derivatan dE/dW vara negativ snarare än positiv och röra sig mot det lokala minimat vid W=1. Detta är mindre bra för nätverket då vi inte längre har den optimala funktionen för att minska feldifferensen i resultatlagret vilket medför att vi inte kommer få ett optimalt tränat nätverk.

# *Fråga 2*: Vad är det som gör att ett neuralt nätverk inte bara är en linjärkombinationsuträkning?
Ett neuralt nätverk kan absolut vara en linjärkombination, men det kommer med stora nackdelar. Om en linjär aktiveringsfunktion (bild nedan) används går det inte använda back propagation för att träna modellen. Eftersom derivatan är konstant går det inte att hitta det W (weight) som hör till det lägsta E (error), dvs det sanna minimat.

![En linjär funktion](https://www.infozone.se/wp-content/uploads/2020/09/linear_function.png)

Väldigt ofta är problemen man vill lösa genom neurala nätverk icke-linjära och således måste en olinjär aktiveringsfunktion användas.

# *Fråga 3*: Jag undrar varför man tilldelar en vikt till en signal i neurala nätverk?
Viktning av signaler är en central del i ett neuralt nätverk, de i kombination med aktiveringsfunktionen avgör om neuronen skall avge en utsignal eller ej. Förenklat kan man säga att en vikt påvisar hur viktig en signal är. Om vikten är konstant (eller obefintlig) kommer alla signaler vara lika viktiga och då kan nätverket inte tränas. Det är genom träningen vikterna sätts och de sätts på ett sätt som styr signalerna att producera rätt svar på det problem nätverket skall lösa.
