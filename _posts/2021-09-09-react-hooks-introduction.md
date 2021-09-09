---
title: "Varför du ska börja använda React Hooks idag"
date: 2021-09-09
author: Willie Björnbom, systemutvecklare
tagline: "Hooks är ett ganska nytt begrepp inom react som håller på att förändra sättet för hur vi bygger applikationer. Jag vill med detta blogginlägg visa hur lätt det är att komma igång och börja jobba med React hooks."
header:
  overlay_image: https://raw.githubusercontent.com/techblog-infozonegroup/resources.techblog-infozonegroup/main/react-hooks-introduction/hooks-header.png
  teaser: https://raw.githubusercontent.com/techblog-infozonegroup/resources.techblog-infozonegroup/main/react-hooks-introduction/hooks-teaser.png
categories:
  - blog
tags:
  - systemutveckling
  - react
  - hooks
---

# Introduktion

I denna bloggpost ska vi kolla på hur man kan använda funktionella komponenter med react hooks och varför alla borde börja använda det redan idag.

Vi kommer titta närmare på greppen `useState` och `useEffect` som är två av de mest använda inom React hooks, jag kommer illustrera hooks med snippets från hur man gjorde på det **Gamla sättet** vs hur man gör med **Hooks**.

# Vad är hooks?

I början av 2019 släpptes [version 16.8](https://reactjs.org/blog/2019/02/06/react-v16.8.0.html) av React med ett helt nytt koncept, kallat hooks. Hooks ger oss möjlighet att skriva funktionella komponenter som är lika kraftfulla som de traditionella klasskomponenterna. Begreppet “hooks” kommer från just detta, att kunna “hooka fast” states och livscykel-metoder på funktionella komponenter.

En hook kännetecknas genom att alla metoder börjar med nyckelordet `use`. Som ett illustrerande exempel kommer jag att nedan visa skillnaden mellan en enkel komponent skriven med hooks vs det traditionella sättet.

**Gamla sättet**

```javascript
import React from "react";

class Example extends React.Component {
  constructor() {
    this.state = {
      count: 0,
    };
  }

  render() {
    return (
      <div>
        <p>{this.state.count}</p>
        <button onClick={() => this.setState({ count: this.state.count + 1 })}>
          Increase counter
        </button>
      </div>
    );
  }
}
```

**Hooks**

```javascript
import React, { useState } from "react";

function Example() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setCount(count + 1)}>Increase counter</button>
    </div>
  );
}
```

## Varför du bör använda hooks

Förutsatt att du har åtminstone v16.8 av React så är du redo att börja använda hooks redan idag. Du behöver inte refaktorera nuvarande klasskomponeter eller liknande då hooks är bakåtkompatibel. React hooks använder färre koncept/termer vilket gör inlärningskurvan betydligt kortare och eftersom funktionella komponenter varken har en konstruktor eller behöver använda en renderingsmetod så blir koden både kortare och mer lättläst.

## State management - useState

React hooks använder hooken `useState` för att “hooka på” state management i funktionella komponenter.

När vi deklarerar en statevariabel med useState-hooken skickar vi med ett argument som blir statets initiala värde och sedan returnerar hooken ett par - en array med två föremål. Det första föremålet i arrayen är statets "nuvarande" värde och det andra är en funktion som uppdaterar statets värde. Vi skulle kunna få tillgång till dessa värden genom att använda arrayens index `[0]` och `[1]` men det är lite förvirrande eftersom det finns en mening med att namnsätta dessa värden. Det är därför vi använder [array destructing](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment) istället, vilket ger oss möjlighet att namnsätta föremålen i arrayen baserat på dess position.

**Hooks**

```javascript
function ExampleWithOneState() {
  const [count, setCount] = useState(0);
}
```

```javascript
function ExampleWithTwoStates() {
  const [age, setAge] = useState(0);
  const [fruit, setFruit] = useState(‘apples’);
  const [todos, setTodos] = useState([{text: ‘Learn Hooks’, done: false}]);
}
```

Som vi ser ovan kan states formas i alla olika typer.

**Hooks**

```javascript
function ExampleWithSet() {
  const [fruit, setFruit] = useState(‘apples’);

  function handleOrangeClick() {
    // Similar to this.setState({ fruit: 'orange' })
    setFruit('orange');
  }
}
```

Ovan ser vi hur vi använder set-funktionen för att uppdatera statets värde. Värt att tänka på är att setfunktionerna med hooks alltid ersätter hela statet med det nya värdet, istället för att slå samman statets gamla värde med det nya som de klassbaserade staten gjorde.

## Side effects - useEffect

Livscykelmetoderna `componentDidMount`, `componentDidUpdate` och `componentWillUnmount` är oundvikligt när du jobbar med klasskomponenter, de används för att hantera dynamiska sidoeffekter när du t.ex. hanterar asynkrona funktioner såsom att fetcha data eller registrera subscriptions. Hooken `useEffect` ersätter alla livscykelmetoder och gör att du kan använda sidoeffekter i funktionella komponenter.

**Hooks**

```javascript
function ExampleEffect() {
  useEffect(() => {
    // Do something
  }, [dep1, dep2]);
}
```

Som vi ser i exemplet ovan så tar useEffect-metoden emot två parametrar:

1. En funktion som exekveras vid effekten.
2. En optional array av beroenden som kan styra när effekten ska triggas (kan vara states eller props). Eftersom denna är optional så kan denna utelämnas, det som då händer är att effekten kommer köras vid varje rendering.

**Gamla sättet**

```javascript
import React, { useState } from "react";

class Example extends React.Component {
  constructor() {
    this.state = { count: 0 };
  }

  componentDidMount() {
    // componentDidMount
  }

  componentDidUpdate(prevProps) {
    if (this.props.count !== prevProps.count) {
      // componentDidUpdate
    }
  }

  componentWillUnmount() {
    // componentWillUnmount
  }
}
```

**Hooks**

```javascript
import React, { useState } from "react";

function Example() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    // Same as componentDidMount
  }, []);

  useEffect(() => {
    // Same as componentDidUpdate
  }, [count]);

  useEffect(() => {
    return () => {
      // Same as componentWillUnmount
    };
  }, []);
}
```

Ovan ser vi hur vi traditionellt hanterar livscykelmetoder med klasser jämfört med hur det hanteras i hooks. I detta exempel skiljer sig de inte så mycket mot varandra där jag skulle säga att den klassbaserade implementationen är lite mer läsvänlig. Grejen med hooks är att dess funktioner är väldigt dynamiska och i detta falla kan vi lösa alla dessa livscykelmetoder i en och samma effekt, se nedan.

**Hooks**

```javascript
import React, { useState } from "react";

function Example() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    /* componentDidMount code + componentDidUpdate code */
    return () => {
      /* componentWillUnmount code */
    };
  }, [count]);
}
```

## Props

Eftersom vi numera inte använder klasser så kommer vi inte heller att ha någon användning av `this`. Detta gör att vi på ett snyggare sätt kan hantera användningen av props. Tänk dig att vi har en komponent likt Example nedan.

```javascript
<Example text="Example" />
```

I en klasskomponent är vi tvugna att hämta propertyn från this.props men det är ingenting vi behöver tänka på med funktionella komponenter.

**Gamla sättet**

```javascript
class Example extends React.Component {
  render() {
    const { text } = this.props;
    return <h1>{text}</h1>;
  }
}
```

**Hooks**

```javascript
function Example({ text }) {
  return <h1>{text}</h1>;
}
```

# Sammanfattning

Detta var en övergripande bild av två av de stora förändringarna useState och useEffect för React v16.8 och det finns väldigt många fler områden att beröra när det kommer till funktionella komponenter och hooks. Med denna bloggpost så hoppas jag att fler börjar implementera hooks för det är lite roligare att skriva samt att det bidrar till en betydligt mer lättläslig kod.

Happy coding!💜

# Resurser

- **[React version 16.8](https://reactjs.org/blog/2019/02/06/react-v16.8.0.html)**
- **[React hooks](https://reactjs.org/docs/hooks-intro.html)**
