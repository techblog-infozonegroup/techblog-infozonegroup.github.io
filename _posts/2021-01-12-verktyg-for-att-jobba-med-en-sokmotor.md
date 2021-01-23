---
title: "Verktyg för att jobba med en sökmotor"
tagline: "Då mitt arbete på senare tid har involverat utveckling mot sökmotorn Apache Solr har detta inspirerat till en ny bloggpost. Begrepp kommer därför därifrån men koncepten bör vara överförbara till andra sökmotorer."
date: 2021-01-12
author: Erik Lenells, Systemutvecklare på Infozone
header:
  overlay_image: https://www.infozone.se/wp-content/uploads/2021/01/binary-code-inspection-picture-id1128503636.jpg
categories:
  - blog
tags:
  - systemutveckling
---
# Om Solr
Solr är en ledande open source-sökmotor av Apache Software. Det används av alla möjliga företag och tjänster, bland annat Netflix och Ticketmaster. Framförallt används Solr för sökfunktioner och autocomplete-funktioner där uppslagen behöver ske snabbt och ge relevanta resultat. Alla dess funktioner fokuserar på sökning och indexering.

# Inledning
De finns många delar som går att påverka som utvecklare när det kommer till en implementation mot en sökmotor. I det här inlägget beskrivs några av dessa och hur de interagerar med varandra. För enkelhetens skull använder detta inlägg ett exempel där sökindexet består av nyhetsartiklar på en webbplats som har delats in i fälten titel, ingress, brödtext och nyckelord.

# Ranking
När en sökfråga sker mot ett sökindex så rankas resultaten via en mängd olika faktorer. Här följer några exempel på mer eller mindre uppenbara saker som påverkar rankningspoängen:

## Fältets vikt
Olika fält ges olika vikt när en sökning görs. Om en sökning ger träff i en artikels titelfält bör det rankas högre än om det gett träff i brödtextfältet. Se mer detaljer i avsnittet för viktning.

## Antal ord i fältet
Antalet ord i fältet, eller fältets längd, påverkar rankingen – om den procentuella matchningen mellan en sökning och ett fält är högre så får fältet högre vikt. Ett annat sätt att beskriva det är att ju fler ord i ett fält desto lägre rankningspoäng. Notera att detta går att stänga av per fält – om nyckelordsfältet innehåller många ord är det dumt om det påverkar rankningspoängen, funktionen är mer logisk för brödtext- och ingressfältet.

## Antal förekomster
Om en sökfras förekommer många gånger i ett fält ökar det rankningspoängen. Detta ställs i relation till hur många gånger frasen förekommer i andra artiklar som matchats. För att räkna ut rankningspoängen som ska tilldelas för antal förekomster använder Solr en algoritm som heter BM25.

# Viktning
Vikten på ett fält bestämmer till vilken grad en sökträff i det fältet ska påverka rankingen för en artikel.

## Indexeringsviktning vs query-viktning
Det finns två typer av möjliga viktningar. Indexeringsviktning är en statisk variant av viktning där fält får sin vikt redan vid indexeringstillfället. Denna viktning kan sedan kompletteras eller överskridas av en query-viktning. En query-viktning innebär att viktningen anges vid söktillfället. Den blir därför möjlig att ändra vid behov beroende på kontext eller andra parametrar.

## Tiebreaker
Tiebreaker handlar om till vilken grad de olika fältens rankningspoäng ska kombineras vid rankningen av en artikel. Om ett sökord ger träff på rubriken för artikel A men ger träff på både rubrik och brödtext för artikel B så är det Tiebreaker-värdet som anger om B ska anses som mer relevant än A för att de matchades i fler fält. I Solr anges Tiebreaker i ett värde från 0.0 till 1.0 där 0.0 innebär att man inte kombinerar fältens rankningspoäng och 1.0 innebär att man fullt ut kombinerar poängen. Anger man 0.5 så kombineras fälten till viss grad. I de flesta fall rekommenderas att ett lågt värde anges, exempelvis 0.2. Med ett lågt värde får man effekten att de högt viktade fälten förblir just viktigast men att övriga fält ändå kan få spela en avgörande roll då två artiklar hamnar väldigt lika i sin rankningspoäng – därav passar uttrycket Tiebreaker mycket bra. Se även tankeexperimentet kring viktning längre ner.

# Stoppord
Det finns många ord som inte bör tas hänsyn till när en sökträff ska matchas mot artiklar. Egentligen är det alla ord som inte tillför något kontexten. Några exempel på dessa ord är: Och, att, i, till. Listan bör anpassas till den verksamhet som ska använda sökningen och kan ha stor effekt på kvaliteten för sökningarna. Om en användare söker på exempelvis ”dykning vid Estonia” så vill man inte lista alla artiklar som matchar ordet ”vid”.

# Mer läsning
De finns fler koncept som är bra att känna till vid en implementation mot en sökmotor men som jag inte kommer gå in på här. Använd valfri sökmotor för att ta reda på mer om dem! Här är några: Boost queries, Phrase slop och Spellchecking.

# Exempel och tankeexperiment för viktning
Notera att exemplet bara tar hänsyn till fältens viktning och inte andra rankningsfaktorer.

## Förutsättningar
- Säg att vi har 2 dokument och de har fälten titel, beskrivning och brödtext
- Titel har en vikt på 3, beskrivning 2 och brödtext 1.5
- Tiebreaker-parametern är satt till 0.0
- Sökordet ”Dykning” matchar endast titel i dokument 1
- Sökordet matchar beskrivning och brödtext i dokument 2

Med dessa förutsättning ger sökmotorn:
- Dokument 1: 3 poäng
- Dokument 2: 2 poäng
- Resultat: Dokument 1 hamnar överst i sökningen

Vi sätter om Tiebreaker-parametern till 1.0 och sökmotorn ger:
- Dokument 1: 3 poäng
- Dokument 2: 2 + (1,5×1.0) = 3.5 poäng
- Resultat: Dokument 2 hamnar överst i sökningen

Vi sätter om Tiebreaker-parametern till 0.5 och sökmotorn ger:
- Dokument 1: 3 poäng
- Dokument 2: 2 + (1.5×0.5) = 2.75 poäng
- Dokument 1 hamnar överst i sökningen

Tänk att Tiebreaker är 0.1 och att det finns fler fält, viktningar och dokument, hamnar resultaten då i önskad ordning? Vilket värde på Tiebreaker kommer ge bäst utfall för din sökfunktion?

Vill du veta mer om mitt arbete som systemutvecklare på Infozone, kan ni besöka denna sidan.
