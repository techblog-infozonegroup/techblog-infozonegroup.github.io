---
title: "Lås flera samtidiga anrop på ett nyckelord"
date: 2021-07-15
author: Erik Lenells, systemutvecklare
tagline: "Kontrollera samtidiga anrop med minimal prestandapåverkan"
header:
  overlay_image: https://raw.githubusercontent.com/techblog-infozonegroup/resources.techblog-infozonegroup/main/tuples-might-be-good/pexels-markus-spiske-1089438.jpg
categories:
  - blog
tags:
  - C# 
  - lock  
  - concurrency
  - keyword
---

Tänk dig ett scenario där flera användare kan påverka samma databasvärde men att de ändå inte får skriva över varandra, alltså ett ”först till kvarn”-scenario som t.ex. att reservera en specifik sittplats i en biograf. Vi tittar närmare på begreppet lock i C# och framförallt hur det kan nyttjas för att låsa en funktion på ett Id eller annat nyckelord så att funktionen kan köras samtidigt så länge parametrarna skiljer sig. 

## Lock
Nyckelordet låser ett kodblock för ett givet objekt och när kodblocket är färdigexekverat lyfts låset. Så om funktionen anropas två gånger samtidigt kommer det ena anropet att pausas tills dess att det andra anropet är fädig med kodblocket. 

Mer detaljer finns här:
https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/lock-statement

Följande kodexempel visar hur vi kan låsa reservationen av en sittplats så att om två anrop med samma _seatId_ sker samtidigt kommer det ena att få igenom sin reservation medan det andra får felet _SeatNotAvailableException_.

```csharp
readonly static object _reservationLockObject = new object();

[...]

lock(_reservationLockObject)
{
    var isSeatFree = IsThisSeatFree(seatId);  
    
    if(isSeatFree)
    {
        ReserveSeat(userId, seatId);
    }
    else 
    {
        throw SeatNotAvailableException();
    }
}
```

Problemet är bara att om två personer försöker boka __olika__ sittplatser samtidigt så kan den ena behöva vänta på att den andras reservation slutförs. Det blir en onödig väntetid och med högt tryck kan svarstiderna bli långa.

Om vi tar bort _lock_ och två personer försöker boka __samma__ sittplats samtidigt är det troligt att koden rapporterar _IsThisSeatFree(seatId)_ som _true_ för båda anropen och även om _ReserveSeat(userId, seatId)_ förmodligen resulterar i att bara en person kommer få bokningen så kommer efterföljande kod bete sig som att två personer bokat samma plats tills dess att en ny koll mot datalagret gjorts.

Så för att lösa dessa två problem kan vi skriva om koden såhär:

```csharp
private readonly static ConcurrentDictionary<string, object> _lockList = new ConcurrentDictionary<string, object>();

[...]

var lockObject = _lockList.GetOrAdd(seatId, new object());
lock(lockObject)
{
    try
    {
        var isSeatFree = IsThisSeatFree(seatId);
        
        if(isSeatFree)
        {
            ReserveSeat(userId, seatId);
        }
        else 
        {
            throw SeatNotAvailableException();
        }
    }
    finally
    {
        _lockList.TryRemove(keyword, out _);
    }
}
```
Här försöker vi hämta ett låsobjekt från den __statiska__ listan __lockList_ med _seatId_ som nyckel. Om det inte finns så skapas ett nytt objekt som kopplas till _seatId_ och används för låsningen. 

_Try Finally_-blocket ser till att seatId tas bort från __lockList_ så fort _ReserveSeat_ är färdig.

__lockList_ är en statisk och trådsäker _dictionary_ vilket garanterar att alla anrop till funktionen kommer åt samma värden och att en nyckel bara kan finnas en gång. 

__Viktigt att notera__ är att detta inte kommer fungera i en lastbalanserad lösning eftersom att statiska variabler lever i processen som kör en applikation. I sådant fall behöver dessa låsobjekt istället hanteras i en distribuerad cache eller databas.

## Wrapper
Vi kan göra wrapper-funktioner som tar en Func eller Action som parameter och ger dom ett lås för angivet nyckelord:

```csharp
public static class KeywordLocker
{
	private readonly static ConcurrentDictionary<string, object> _lockList = new ConcurrentDictionary<string, object>();
	public static TResult WrapInLock<TResult>(Func<TResult> function, string keyword)
	{
		var lockObject = _lockList.GetOrAdd(keyword, new object());
		lock (lockObject)
		{
			try
			{
				return function();
			}
			finally
			{
				_lockList.TryRemove(keyword, out _);
			}
		}
	}

	public static void WrapInLock(Action function, string keyword)
	{
		var lockObject = _lockList.GetOrAdd(keyword, new object());
		lock (lockObject)
		{
			try
			{
				function();
			}
			finally
			{
				_lockList.TryRemove(keyword, out _);
			}
		}
	}
}
```

## Demo

En konsollapp som med hjälp av _async await_ sätter igång flera samtidiga anrop mot en funktion som sparar ner en inparameter i en lista om den inte redan finns där. Först visas resultatet utan en låsning och sen visas resultatet när vår KeywordLocker används:

![Concurrency Result](https://raw.githubusercontent.com/lenellsarn/techblog-infozonegroup.github.io/lockobjects/assets/images/lockobjects-demo-concurrency-result.PNG)

Som syns ovan får vi ett kaotiskt resultat om vi inte låser funktionen när samtidiga anrop sker. Om vi däremot använder vår _KeywordLocker.WrapInLock()_ så ser vi att det bara sparats ett resultat per biljett-id samtidigt som reservationen av TicketId1 inte blockerat en samtidig reservation av TicketId2 vilket hade varit fallet om vi bara använt _lock_.

Koden finns här:
https://github.com/lenellsarn/blog.lockobjectsdemo

## Relaterad läsning
Om du ute efter att "throttla" dina anrop, det vill säga begränsa antalet anrop som kan köras parallellt rekommenderar jag att kolla närmare på Semaphore Slim, väl sammanfattat här: https://techblogg.infozone.se/blog/throttling-using-semaphore-slim/