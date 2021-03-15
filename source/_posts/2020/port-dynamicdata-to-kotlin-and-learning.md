---
title: Port DynamicData to Kotlin and Learning
date: 2020-12-30 11:04:00
categories: Tech
tags:
 - Kotlin
---
<div class="banner-img">
    <img src="/images/2020/dynamicdata-banner.png">
</div>

After learning about Scala for a while, I have less resistance to the language of the JVM platform. It seems that not all languages ​​are as rigid and difficult to use as Java. Maybe JetBrains came up with something that works? With this idea, I tried to port DynamicData to Kotlin. However, the reality is that Kotlin reveals the dilapidation of JVM everywhere, and there is still a long distance from small and beautiful.
<!--more-->

Before this, I tried to understand the ecosystem of JVM. Kotlin was still immature at that time, and I was not very interested in the syntax of Java, so I planned to learn Spring. However, the first problem encountered at that time was dependency management. When I created an IntelliJ JVM project, I immediately became at a loss and did not know how to reference SpringBoot packages, because the JVM project did not have a CLI package manager similar to NuGet. Recently Kotlin also released a new stable version of 1.4, so I plan to port DynamicData to Kotlin and learn it by the way.

# What is DynamicData?
For developers in the JVM ecosystem, they are already familiar with RxJava and Reactor. But in the CLR ecosystem, ReactiveX has more mature usage, one of which is reactive UI. It is generally necessary to save instances of all required data behind the UI, instead of persisting them like back-end development. Using `Subject` can make  single-element data reactive, but cannot deal with list-type data, because `Subject` will emit the entire list instead of only the changed part.

C# developers usually use ObservableCollection for this purpose, because it has an event named CollectionChanged, which has the following properties:
```chsarp
public class NotifyCollectionChangedEventArgs : EventArgs 
{
    public IList? NewItems { get; }

    public IList? OldItems { get; }
 
    public int NewStartingIndex { get; }
 
    public int OldStartingIndex { get; }
}
```

> Rx is extremely powerful but out of the box provides nothing to assist with managing collections. In most applications there is a need to update the collections dynamically. Typically a collection is loaded and after the initial load, asynchronous updates are received. The original collection will need to reflect these changes. In simple scenarios the code is simple. However, typical applications are much more complicated and may apply a filter, transform the original dto and apply a sort. Even with these simple every day operations the complexity of the code is quickly magnified. Dynamic data has been developed to remove the tedious code of dynamically maintaining collections. It has grown to become functionally very rich with at least 60 collection based operations which amongst other things enable filtering, sorting, grouping, joining different sources, transforms, binding, pagination, data virtualisation, expiration, disposal management plus more.

But in the process of porting, with the deepening of the understanding of Kotlin, the following problems were gradually discovered. These problems are largely caused by JVM. Some requirements that are easy to achieve in C# are difficult to achieve with Kotlin.

# Type Erasure

## Runtime
I know that type erasure is something that has been mentioned countless times, and it is very troublesome everywhere. For debugging convenience in DynamicData, the ToString() method is rewritten and type information is output (developers familiar with rx know how difficult it is to debug). But in Kotlin, it is impossible to tell developers what the generic type of this `ChangeSet<T>` is. Of course, Kotlin also provides a solution, adding a `KClass` field, passing `<refied T>` at compile time through the inline factory method in the champion object. But this method is contagious. In order not to lose generic information, this parameter needs to be added to all operators' constructor. It is not graceful.

## Overload Methods
In some cases, the fields and methods allowed in Kotlin cannot be implemented due to JVM limitations, such as overload methods. Therefore, the Kotlin compiler introduced an annotation @JvmName to manually specify the function name in the bytecode. 

The following is an example. Due to type erasure, different typed lambda parameters actually have the same type after compilation, so they cannot have the same method name in the JVM. If these codes are called in Java, due to different method names, it destroys the meaning of function overload.
```kotlin
@JvmName("transformWithIndex")
fun <T, R> Observable<IChangeSet<T>>.transform(
    transformFactory: (T, Int) -> R,
    transformOnRefresh: Boolean = false
): Observable<IChangeSet<R>> =
    this.transform(
        { t, _, idx -> transformFactory(t, idx) },
        transformOnRefresh
    )

@JvmName("transformWithOptional")
fun <T, R> Observable<IChangeSet<T>>.transform(
    transformFactory: (T, Optional<R>) -> R,
    transformOnRefresh: Boolean = false
): Observable<IChangeSet<R>> =
    this.transform(
        { t, prev, _ -> transformFactory(t, prev) },
        transformOnRefresh
    )
```

## toList()
When using nested generic methods such as `Iterator<A<T>>`, `toList()` cannot infer the current generic type T and it assumes receiver type is `Iterator<T>`, so kotlin complains that "None of the following candidates is applicable because of receiver type mismatch". But if the variable is converted to `Sequence<A<T>>` first, due to that Sequence defines an instance method toList(), we finally get `List<A<T>>`.
```kotlin
fun <T> Reverser<T>.reverse(items: ChangeSet<T>): Iterator<Change<T>>

fun <T> Observable<ChangeSet<T>>.reverse(): Observable<ChangeSet<T>> =
    Reverser<T>().let {
        this.map { x ->
            val list = it.reverse(x)
            // compile error
            // ChangeSet(list.toList())
            ChangeSet(list.asSequence().toList())
        }
    }
```

## Covariance
I don't know the design of kotlin generic type, but it's indeed hardly to use when return generic interface. As same as in C#, it doesn't allow covariance in return value, but C# allow sub-interface when return type is parent interface.
```kotlin
interface PageChangeSet<T>: ChangeSet<T>

fun run(): Observable<ChangeSet<T>> =
    Observable.create<PageChangeSet<T>> { emitter ->
        TODO()
    }
    // cannot compile unless unsafe type cast
    as Observable<ChangeSet<T>>
```

## Constructor
If you must put type erasure, constructor at the same place, JVM will complain these 'ctor' methods have the same signture and `JvmName` cannot be is not applied to target 'constructor'.
```kotlin
constructor(reTransformer: Observable<Unit>
constructor(reTransformer: Observable<(Person) -> Boolean>)
```

However constructor overload is very common in C#, maybe someone doesn't prefer to use factory methods. So, to pretend to have a constructor, a function with the special name `invoke` in companion object is useful, of course, a private constructor with all needed arguments is necessary.
```kotlin
private class TransformStub(
    val source: SourceCache<String, Person>,
    val result: ChangeSetAggregator<String, PersonWithGender>
) : Disposable {
    companion object {
        operator fun invoke(): TransformStub {
            val source = SourceCache<String, Person> { it.name }
            return TransformStub(
                source,
                ChangeSetAggregator(source.connect().transform(transformFactory))
            )
        }

        @JvmName("invokeWithUnit")
        operator fun invoke(reTransformer: Observable<Unit>): TransformStub {
            val source = SourceCache<String, Person> { it.name }
            return TransformStub(
                source,
                ChangeSetAggregator(source.connect().transform(transformFactory, reTransformer))
            )
        }

        operator fun invoke(reTransformer: Observable<(Person) -> Boolean>): TransformStub {
            val source = SourceCache<String, Person> { it.name }
            return TransformStub(
                source,
                ChangeSetAggregator(source.connect().transform(transformFactory, reTransformer))
            )
        }
    }
```

# Lambda With Receiver
Lambda parameters in Scala do not need to be written in parentheses, which really improves the readability of the code. Kotlin also has this syntax, and uses return@label to return within the nest. In most cases, this works fine, but when assigning a lambda variable, because the variable is not a function, it can only use implicit return, which leads to potential bugs. If you must use return, you must create a label.

Not work:
```kotlin
val variable = { x: Int ->
    return x + 1
}
```

Work:
```kotlin
val variable = site@ { x: Int ->
    return@site x + 1
}
```
or
```kotlin
val variable = { x: Int ->
    x + 1
}
```

# Reflection
In DynamicData, for classes that implement INotifyPropertyChanged interface, AutoFresh operator can be applied to monitor property changes. This operator accepts an `Expression<Func<T, R>>` parameter, and uses reflection to take out the property getter called in the expression, which not only achieves strong consistency but also uses static type intellisense. However, the reflection in the JVM cannot read the expression tree. Fortunately, Kotlin provides the function of Property References. Getters can be saved in `KProperty1<T, R>` and passed.
```kotlin
open class PropertyChangedEvent(
    sender: Any,
    open val propertyName: String
) : EventObject(sender)

val propertyChanged = obj.propertyChanged
    .filter { PropertyChangedEvent.propertyName == accessor.name }
    .map { factory() }
```

# API in RxJava
When I first started using Reactor, I was very curious why the names of the concepts were different, so I switched to RxJava. What is the point of wasting time on the same things? On the RxJava side, the API design is also different from Rx.Net. This is not to say that the names of operators are different, but to be more specific: `Observer`.

## Subscribe
Since RxJava2, observable has been divided into observable and flowable. From the perspective of back pressure, this is understandable. But why should observer be subdivided into observer and emitter? Observer has an `onSubscribe` method, which accepts a `Disposable` parameter in order to save current subscription. Then in the `subscribe` method, it does not return a `Disposable`, and it does not overload to accept an `emitter`. If you want to get `Disposable`, then use the overload method that accepts three delegates. So I wrote an extension method to reduce duplication of code.
```kotlin
internal fun <T> Observable<T>.subscribeBy(observer: Observer<T>): Disposable =
    subscribe(observer::onNext, observer::onError, observer::onComplete)

internal fun <T> Observable<T>.subscribeBy(emitter: ObservableEmitter<T>): Disposable =
    subscribe(emitter::onNext, emitter::onError, emitter::onComplete)
```

## Observable.create
When I use Observable.create to create Observable, things are even stranger, it accepts a delegate but the return value is Unit. Then `ObservableEmitter` has a method called `setDisposable` to set Disposable. So why is it designed like this. Is it not accpetable to change the return value from Unit to Disposable?
```kotlin
fun run(): Observable<T> =
    Observable.create { emitter ->
        val d = CompositeDisposable(...)

        emitter.setDisposable(d)
    }
```

## Scan
Scan operator will use accumulator to calculate each received value and emit the result, it also has an overload accepts a seed parameter. The problem is that seed is generally used to initialize the state, but RxJava will emit seed as the first value to downstream. Therefore I must drop the first value every time.

From the source code of RxJava, the seed indeed be emitted as the first value in `onSubscribe`. I don't know why.
```java
@Override
public void onSubscribe(Disposable d) {
    if (DisposableHelper.validate(this.upstream, d)) {
        this.upstream = d;
        downstream.onSubscribe(this);
        downstream.onNext(value);
    }
}
```

# Unit Test
It may be because I'm not familiar with the Java ecosystem, and I have not found a useful unit test framework. Since Java has no extension methods, almost all frameworks require packaging of test data before they can call related test methods.
```java
assertEquals(2, addMethod(1, 1));
```

But in C# I usually use FluentAssertion, this library wraps xunit as fluent API, which can effectively increase the readability of unit tests.
```csharp
addMethod(1, 1).Should().Be(2);
"ABCDEFGHI".Should().StartWith("AB").And.EndWith("HI").And.Contain("EF").And.HaveLength(9);
```

Kotlin also has a library called Kluent, which provides a fluent API with infix methods. However, it just is a light wrapper of junit and contains a very limited set of DSL API. Due to that it doesn't create a container class, many additional operations cannot be applied such if and else.
```kotlin
addMethod(1, 1) shouldBeEqualTo 2
items.toList() shouldBeEqualTo (6..10).toList()
```
