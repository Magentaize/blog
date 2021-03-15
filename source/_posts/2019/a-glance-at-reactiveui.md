---
title: A Glance at ReactiveUI
date: 2019-03-23 05:55:41
categories:
 - Tech
tags:
 - ReactiveX
---

<div class="banner-img">
    <img src="/images/2020/reactiveui-banner.png">
</div>

When we are programming an application that contains plenty of streams of data, it's awful to deal with the combinations of them. Especially in GUI applications, we also need to handle all input events and then response for them. To be honest, I'm bored with these boring jobs. But, ReactiveUI has changed everything and the past had passed.
<!--more-->

Think about those jobs, what you have to do isn't to write event handler but to organize the logic of how to deal with the event. Fortunately, ReactiveUI has made perfect work to abstract the logic, and you do not need to handle again and again.

## Scenario：EventBus ##
Many applications need a centralization configuration to provide global configs, typically we use EventBus or EventAggregator to implement this pattern. When use them, we need to add a event handler and write some codes, but if switch to ReactiveUI, every event can be expressed by a `Subject<T>` which behaves like a infinite sequence and we can simply push a new item into a `Subject<T>` at the same time every listener will receive this item. Due to reactive fluent api, the logic looks more clear than before.

```csharp
ISubject<Person> NextCustom = new Subject<Person>();

NextCustom.Subscribe(x => ImplServiceFor(x));

NextCustom.OnNext(new Person());
```

## Scenario：Data Source Collection Changed ##
