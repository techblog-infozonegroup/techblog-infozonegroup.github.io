---
title: "Kör (Bicep) CLI i en docker-container"
date: 2021-02-28
author: Fredde Johnsson, systemutvecklare
tagline: "[Docker](https://www.docker.com/), [kubernetes](https://kubernetes.io/) och annan containerteknik kanske ibland känns som verktyg eller plattformar för servrar, hosting, PaaS och stora jättar uppe bland molnen. För mig har det däremot dom senaste 4-5 åren varit en naturlig del av mitt verktygsbälte med utvecklingsverktyg."
header:
  overlay_image: https://raw.githubusercontent.com/techblog-infozonegroup/resources.techblog-infozonegroup/main/docker-bicep-cli/colors-1838392_640.jpg
  teaser: https://raw.githubusercontent.com/techblog-infozonegroup/resources.techblog-infozonegroup/main/docker-bicep-cli/teaser.png
categories:
  - blog
tags:
  - bicep
  - azure
  - arm
  - devops
---
I den här posten tänkte jag, bara som ett exempel, visa hur jag använde docker för att slippa installera grejor på min arbetsmaskin. I dom flesta fallen kanske det handlar om att slippa installera serverkomponenter såsom Elasticsearch, Redis, Sql Server (for Linux) osv, men i det här exemplet beskriver jag hur jag använder docker för att testa [Project Bicep](https://github.com/Azure/bicep) och dess CLI för att generera [ARM Templates (Azure Resource Manager)](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/).

# Docker (Desktop for Windows)
Eftersom jag till 100% sitter och utvecklar på en Windowsmaskin så var det en fröjd den dagen [Docker for Windows](https://hub.docker.com/editions/community/docker-ce-desktop-windows) kändes stabilt och bra. Idag är det ett verktyg som ingår i min standardinstallation av arbetsmaskin. Smidigheten i att alltid ha det till hands lokalt gör det otroligt användbart, möjligheten att välja om jag ska köra Windows- eller Linux-containrar är väldigt värdefull.

Den absoluta merparten av tiden köra jag Linux-containrar, så även i det här exemplet. En stor anledning till att det är så är att Microsoft gör ett allt bättre jobb vad gäller plattformsoberoende verktyg och kompatibilitet i sina SDK:er och plattformar.

Vad docker är och containerization innebär finns det oändligt mycket information om på Internet.

Nedan följer dock en kort introduktion till Prject Bicep och ARM Templates. Det kommer eventuellt en längre post i ämnet senare, där intrycken av Bicep efter lite testande sammanfattas.

# Project Bicep och ARM Templates
Den som gett sig i kast med att hantera Azure-resurser med hjälp av [ARM Templates](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/) har säkert haft åsikter om hur mycket stök och strul det kan vara med att få huvudet runt den stora mängden json-data som måste vara helt korrekt. Det underlättas såklart om man har ett bra verktyg till hands. Jag rekommenderar ett extension till VS Code, [Azure Resource Manager (ARM) Tools](https://marketplace.visualstudio.com/items?itemName=msazurermtools.azurerm-vscode-tools). Det ger mycket stöd i form av Intellisense för stora delar av json-strukturen. Det skulle såklart vara önskvärt att förenkla så mycket som möjligt, skala bort allt "onödigt" runt omkring och kunna deklarera variabler etc. Jag blev otroligt glad när jag hörde om [Project Bicep](https://github.com/Azure/bicep) och att det skulle vara ett DSL med just dom egenskaperna. Den här avsnittet [https://channel9.msdn.com/Shows/DevOps-Lab/Project-Bicep--Next-generation-ARM-Templates](https://channel9.msdn.com/Shows/DevOps-Lab/Project-Bicep--Next-generation-ARM-Templates) av [DevOps Lab på Channel 9](https://channel9.msdn.com/Shows/DevOps-Lab) var det som fångade mitt intresse, en alldeles lysande introduktion till vad Project Bicep är och dess grundläggande egenskaper.

Dom två största uppsidorna, som jag ser det teoretiskt, är:
- Modularisering av resurser som förenklar återanvändande av kod.
- En abstraktion som översätts till "vanlig" ARM-json-struktur som ger en extra validering av deklarationerna INNAN man försöker deploya till målmiljön.

Med detta i tankarna planerar jag att inom kort eventuellt hålla en lunch-dragning om Bicep eller iallafall att skriva en bloggpost i ämnet. Nedan tänkte jag visa hur man kan köra **Bicep CLI i en docker-container** och på så sätt kanske locka till att använda en container i andra sammanhang där man behöver en CLI-installation men inte önskar ta det beroendet i form av en installation på sin utvecklardator.

# Bicep CLI i container
Jag tänkte alltså att jag skulle testa Bicep och då behövdes en Bicep CLI-installation, beskriven här [https://github.com/Azure/bicep/blob/main/docs/installing.md](https://github.com/Azure/bicep/blob/main/docs/installing.md). Jag kände ganska omgående att jag inte ville installera något på min utvecklarmaskin. Det var inte så att jag inte litade på installationen, jag bara ville inte. Däremot kände jag att det skulle vara kul att prova att bygga en dockercontainer, starta och ansluta till den och köra `bicep build ...` där och på så sätt generera ARM-template-json-filen med hjälp av CLIn i containern. 

Sagt och gjort, så här gjorde jag:
- Försök hitta en base-image att bygga på => `mcr.microsoft.com/dotnet/runtime:3.1`
- Vald image bygger på Debian, uppdatera den och installera curl vilket behövdes för att nedladdning av Bicep CLI => `RUN apt-get update; apt-get install -y curl`
- Kopiera installationsstegen, Linux-versionen => bicep installeras och OS:et konfigureras
- Bygg imagen => `docker build -t bicep-lab .`
- Kör igång en containerinstans och exekvera/anslut till bash => `> docker run --rm -it bicep-lab /bin/bash`
- Prova att köra `bicep --version` ansluten till bash i containern => `Bicep CLI version 0.2.328 (a13b032755)` <br>=> **success**

Dockerfilen som jag byggde imagen med ser alltså ut enligt
```yml
FROM mcr.microsoft.com/dotnet/runtime:3.1

RUN apt-get update; apt-get install -y curl

# Fetch the latest Bicep CLI binary
RUN curl -Lo bicep https://github.com/Azure/bicep/releases/latest/download/bicep-linux-x64
# Mark it as executable
RUN chmod +x ./bicep
# Add bicep to your PATH (requires admin)
RUN mv ./bicep /usr/local/bin/bicep
# Verify you can now access the 'bicep' command
RUN bicep --help
# Done!
```
> Notera att förutom dom två översta raderna, `FROM ...` och `RUN apt-get ...` så är det en rak kopiering av installationsanvisningen för CLIn i Linux.

Det som nu återstod var att kunna komma åt dom bicep-filer som jag byggde i VS Code på min Windows-maskin. Detta gjorde jag genom att starta containern och mounta den aktuella katalogen till `/src/` i containern. 

Se sekvensen av kommandon nedan:
-  före start av containern, 
- mountning av katalogen med bicep-filer, 
- bicep build, 
- exit och 
- kontroll så att den genererade json-filen finns i katalogen:

```
...\lab\bicep-docker\src> ls 


    Directory: D:\develop\lab\bicep-docker\src


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         2/22/2021   8:56 PM            353 appInsights.bicep
-a----         2/22/2021   8:50 PM            283 appService.bicep
-a----         2/23/2021   8:07 AM           1228 functionApp.bicep
-a----         2/23/2021   8:12 AM            519 storageAccount.bicep


 ...\lab\bicep-docker\src> docker run --rm -it -v ${pwd}:/src bicep-lab /bin/bash
 root@0762625e4278:/# cd src
 root@0762625e4278:/src# ls
appInsights.bicep  appService.bicep  functionApp.bicep  storageAccount.bicep
root@2c1dbf538229:/src# bicep build functionApp.bicep
root@0762625e4278:/src# ls
appInsights.bicep  appService.bicep  functionApp.bicep  functionApp.json  storageAccount.bicep
root@0762625e4278:/src# exit
...\lab\bicep-docker\src> ls


    Directory: D:\develop\lab\bicep-docker\src


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         2/22/2021   8:56 PM            353 appInsights.bicep
-a----         2/22/2021   8:50 PM            283 appService.bicep
-a----         2/23/2021   8:07 AM           1228 functionApp.bicep
-a----         2/27/2021   4:26 PM           6157 functionApp.json
-a----         2/23/2021   8:12 AM            519 storageAccount.bicep


...\lab\bicep-docker\src>
```

> ${pwd} returnerar aktuell katalog i PowerShell.


Start-kommandot av containerinstansen innehåller, förutom mountningen, två saker att notera:
- `'-it'`-växeln => kör containern interaktivt, det vill säga "anslut till den startade instansen"
- `'/bin/bash'` => exekvera /bin/bash i containern

Det här gör att man kommer att hamna i bash-skalet i containern och man kan alltså exekvera vidare andra kommandon där, precis som om man anslutit till vilken linux-instans som helst.

# Lite extra
Ovan verkade funka fint! Varför inte låta containern göra jobbet lite mer automatiskt i bakgrunden? Man borde väl kunna låta den reagera på förändringar i `/src/`-katalogen och då atuomatiskt köra `bicep build ...bicep`?

Modifierade dockerfile enligt nedan ger det önskade beteendet:
```yml
FROM mcr.microsoft.com/dotnet/runtime:3.1

RUN apt-get update; apt-get install -y curl

# Fetch the latest Bicep CLI binary
RUN curl -Lo bicep https://github.com/Azure/bicep/releases/latest/download/bicep-linux-x64
# Mark it as executable
RUN chmod +x ./bicep
# Add bicep to your PATH (requires admin)
RUN mv ./bicep /usr/local/bin/bicep
# Verify you can now access the 'bicep' command
RUN bicep --help
# Done!

RUN apt-get install -y npm 
RUN npm i -g watch-cli 

WORKDIR /src
ENTRYPOINT [ "watch", "-p *.bicep", "-c bicep build main.bicep" ]
```
Det som har kommit till är installation av npm genom `RUN apt-get install -y npm ` samt installation av npm-paketet watch genom `RUN npm i -g watch-cli`. `WORKDIR`och `ENTRYPOINT` ser till att watch körs i `/src/`-katalogen och triggas när en `.bicep`-fil förändras eller skapas. Då körs `bicep build ...`.
Jag byggden ny image med från ovan dockerfile `docker build -t bicep-watch .` och när den körs ser det ut enligt
```
...\lab\bicep-docker\src> docker run -v ${pwd}:/src --rm -it bicep-watch
Watching started
-----> Här sparade jag om main.bicep (omdöpt functionApp.json från tidigare exempel)
/src/main.bicep changed
Running  bicep build main.bicep
Finished  bicep build main.bicep
Watching started
```
En listning av katalogen på min Windows-maskin ser nu ut enligt:
```
...\lab\bicep-docker\src> ls


    Directory: D:\develop\lab\bicep-docker\src


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         2/22/2021   8:56 PM            353 appInsights.bicep
-a----         2/22/2021   8:50 PM            283 appService.bicep
-a----          3/1/2021   6:32 AM           1230 main.bicep
-a----          3/1/2021   6:32 AM           6157 main.json
-a----         2/23/2021   8:12 AM            519 storageAccount.bicep


...\lab\bicep-docker\src>
```
Där syns den nybyggda `main.json`.

Nu tänkte jag inte ta det här längre. Hoppas att jag i och med detta har lockat fler till att försöka använda docker som ett utvecklingsverktyg, även för sådant som inte är server-mjukvara.

