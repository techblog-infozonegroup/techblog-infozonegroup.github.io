---
title: "Undersökning av deconstructing i C#"
date: 2021-09-10
author: Fredde Johnsson, systemutvecklare
tagline: "I tidigare post har vi tittat på tuples och hur man kan använda dessa för att returnera flera värden samtidigt utan definiera en klass. I den här posten undersöker vi hur man kan nyttja sk deconstructing för att 'veckla ut' typer på ett smidigt sätt."
header:
  overlay_image: https://user-images.githubusercontent.com/460203/129669260-65dc36a5-2f02-444e-b1d2-36065504a8ce.jpg
  teaser: https://raw.githubusercontent.com/techblog-infozonegroup/resources.techblog-infozonegroup/main/tuples-might-be-good/teaser.jpg
categories:
  - blog
tags:
  - systemutveckling
  - c#
---
# Introduktion

# Bakgrund

# Kod
```csharp
static void DeconstructTuples()
{
    var successTuple = (true, new Person(), 0);
    var (isSuccess1, successModel, successStatus) = successTuple;

    var failTuple = (false, default(Person), 123);
    var (isSuccess2, failModel, failStatus) = failTuple;

    // Ignore/discard things
    var (success1, model, _) = (true, new Person(), 0);
    var (success2, _, status) = (false, default(Person), 123);
}
```

```csharp
static void DeconstructResultAsClassesWithClassModel()
{
    var successClass = new ResultAsClass<Person>(true, new Person(), 0);
    var (isSuccess1, successModel, successStatuts) = successClass;

    var failClass = new ResultAsClass<Person>(false, default, 123);
    var (isSuccess2, failModel, failStatus) = failClass;

    // Ignore/discard things
    var (success1, model, _) = new ResultAsClass<Person>(true, new Person(), 0);
    var (success2, _, status) = new ResultAsClass<Person>(false, default, 123);
}

static void DeconstructResultAsClassesWithRecordModel()
{
    var successClass = new ResultAsClass<Human>(true, new Human(1, "Test"), 0);
    var (isSuccess1, successModel, successStatuts) = successClass;

    var failClass = new ResultAsClass<Human>(false, default, 123);
    var (isSuccess2, failModel, failStatus) = failClass;

    // Ignore/discard things
    var (success1, model, _) = new ResultAsClass<Human>(true, new Human(1, "Test"), 0);
    var (success2, _, status) = new ResultAsClass<Human>(false, default, 123);
}
```

```csharp
static void DeconstructResultAsRecordWithClassModel()
{
    var successRecord = new ResultAsRecord<Person>(true, new(), 0);
    var (isSuccess1, successModel, successStatuts) = successRecord;

    var failRecord = new ResultAsRecord<Person>(false, default, 123);
    var (isSuccess2, failModel, failStatus) = failRecord;

    // Ignore/discard things
    var (success1, model, _) = new ResultAsRecord<Person>(true, new(), 0);
    var (success2, _, status) = new ResultAsRecord<Person>(false, default, 123);
}

static void DeconstructResultAsRecordWithRecordModel()
{
    var successRecord = new ResultAsRecord<Human>(true, new(1, "Test"), 0);
    var (isSuccess1, successModel, successStatuts) = successRecord;

    var failRecord = new ResultAsRecord<Human>(false, default, 123);
    var (isSuccess2, failModel, failStatus) = failRecord;

    // Ignore/discard things
    var (success1, model, _) = new ResultAsRecord<Human>(true, new(1, "Test"), 0);
    var (success2, _, status) = new ResultAsRecord<Human>(false, default, 123);
}
```
