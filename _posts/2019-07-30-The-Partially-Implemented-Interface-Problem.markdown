---
layout: post
title: The Partially-Implemented Interface Problem
date: 2019-07-30 18:00:00 +0100
description: The product of a year's obsession, we explore a minor issue with programming languages and redesign everything to fix it.
img: posts/20190739/sharpshot.png
tags: [Methodical Typing]
---

# Preface

I originally had the idea for this article over a year ago.
At the time, I was struggling with constant `NotImplementedException` in Java's JavaFX GUI library.
I knew there had to be a better way, and have been thinking about how to solve the problem ever since.
The problem is real, but the solution is overkill.
Enjoy the journey, and enjoy thinking about language design in more detail than you have before - I certainly have.

# Introduction

It's a long one, so buckle in.
I think it's all interest, but each section can be read independently, so feel free to skip bits if it's too slow or not interesting.
I'll include a recap so far at the top of each section, so you can just read the bits you're interested in.

This article will introduce a common object-oriented programming anti-pattern, 'Partially-Implemented Interfaces'.
We'll discuss what causes it, the problems that occur as a result.
Then, we'll think about preventing it, first through the use of better patterns, before designing a programming language and type system to encourage the use of those patterns.
We'll write a basic transpiler, and try it out on a project, thinking about where our design went right and what went wrong.
Finally, we'll reflect on the difficulties and compromises required with programming language design.

All code snippets in the article are typescript.
Of the languages that would support what we want to do, it was the most widely used and known.
The issues discussed are not limited to any one language, and affect most object-oriented programming languages.

# Partially-Implemented Interfaces: An OOP Anti-Pattern

> A partially-implemented interface is a class which implements an interface but does not support some of its methods, and instead throws runtime exceptions when they are called

The code below shows an example of this anti-pattern, implementing the class hierarchy in the image.
We'll be referring back to this code as we attempt some solutions, so make sure you understand what's going on.

![Original class ](/assets/img/posts/20190730/bad.svg "Starter class hierarchy")

```typescript
interface Stream {
    isOpen(): boolean;
    close(): void;
    open(): void;
}

class SingleUseStream implements Stream {
    private currentlyOpen: boolean = true;
    isOpen = () => this.currentlyOpen;
    close = () => this.currentlyOpen = false;
    open = () => {
        throw "Unsupported Operation"
    }
}

class MultiUseStream implements Stream {
    private currentlyOpen: boolean = true;
    isOpen = () => this.currentlyOpen;
    close = () => this.currentlyOpen = false;
    open = () => this.currentlyOpen = true;
}

function toggleStream(stream: Stream) {
    stream.isOpen() ?
        stream.close() :
        stream.open()
}

function closeIfOpen(stream: Stream) {
    if (stream.isOpen()) {
        stream.close()
    }
}
```

This example code is simplistic compared to what we see in the wild, but examples of this behaviour show up everwhere.
We define a Stream interface, which defines three methods.
We provide two implementations, and two static functions which operate on a stream.
The snippet below always throws an error, but passes the compiler checks without issue.

```typescript
const stream = new SingleUseStream();
toggleStream(stream);
```

`SingleUseStream` implements `Stream`, meaning that the type checker is happy to pass it to `toggleStream`.
When `toggleStream` calls `Stream.open()`, it will throw an error.

## Issues Caused

//TODO Liskov Substitution Principle

Using `SingleUseStream` successfully is incredibly difficult.
The programmer needs to understand the implementation of every method that they pass it to.
If any of the implementations call `Stream.open()`, the code will crash.
When there are multiple classes that use partially-implemented interfaces, the programmer will learn to not trust the compiler.
At that point, the codebase will have all the downsides of a statically-typed language (verbose, inflexible code), while also having all the downsides of a dynamically-typed language (more intermittent bugs, less readable).
Additionally, a new programmer will have constant issues due to trusting the compiler and running into type compatability issues at runtime.

This problem is incredibly serious, and can slow development massively.
You may think that nobody would be stupid enough to write code like this, but it happens frequently.
As we will see later, there is often no good alternative due to the limitations of the type systems in the language.

## Why would someone do this?

There are two main reasons to partially-implement an interface.

### Work-In-Progress Code

When the codebase is a work-in-progress, it's important to compile often to check over the code that is written.
Since an interface is a specification of how a class should behave, they are written first and completely, before any code is written.
Therefore, when implementing the interface, nothing can be tested until all methods are implemented (at least superficially).
Programmers get around that by using method stubs, short methods that satisfy the compiler but do not actually implement the required behaviour.
C# implements this functionality natively, with its `NotYetImplementedException`.
They often look like this:

```java
public void doSomething(){
	// TODO implement this
	throw new RuntimeException("Not Yet Implemented")
}
```

This is probably the least harmful use of a partially-implemented interface.
It is temporary, and the programmer already understands the codebase, so will know when the method gets called.
However, in classic `TODO` fashion, this will never actually be implemented.
Then, this code gets published as part of an external API, and used by people who get annoyed that it randomly throws exceptions.

### Attempting to provide a 'sane' default

In java, `AbstractList` throws an `UnsupportedOperationException` whenever you call any of the following methods:

* `set`
* `add`
* `remove`

I can only assume that this is to make it less intimidating to extend `AbstractList`.
When doing so, the only method that you must implement is `get`.
These default 'implementations' (sarcastic air quotes) mean that the class will compile with just that implementation.

However, this should not be encouraged.
Any programmer implementing AbstractList will assume that the methods with default implementations are actually useful.
There is no reason, until it starts throwing exceptions, to look at the `AbstractList` source.

### Overly Broad Interfaces

In external APIs, this is the more common scenario.
When writing the interface, the developer decided 'Everything should be able to do *x*', and that turned out to be completely unnecessary.
By the time people realised it was unnecessary, it was too late.

Java's `ConcurrentHashMap.Node` implements `Map.Entry`.
That means it is forced to implement `setValue`, which could lead to race conditions when used concurrently.
To avoid this, it does not implement the method and instead throws an `UnsupportedOperationException`.

The `Map.Entry` interface is too broad, they did not consider that some entries may be read-only.
Of course, this could be solved by providing a more precise inheritance hierarchy.
We could separate maps into read-only and read-write maps.
Kotlin, another JVM language, does that in its standard library:

![Kotlin Map and MutableMap class hierarchy](/assets/img/posts/20190730/map.svg "Kotlin Map and MutableMap class hierarchy")

If we consider it in more depth, the problem of overly broad interfaces becomes more apparent.
We don't just care about times where an external API has a method stub that throws an interface.
Every time a function takes a parameter and doesn't use every single method provided by that type, it is an overly broad interface.
Consider the following code:

```java
public static Integer first(List<Integer> list){
	return list.get(0);
}
```

The `List` interface is overly broad here.
The only method used is `get`, but if we wanted to use this method with a custom class, we need to implement every method defined on `List`.
That's 29 methods!
It's quite tempting to just implement `get` and add method stubs for the other 28.
Overly broad interfaces like this encourage programmers to partially-implement interfaces!

### Because of bad programmers?

You're probably thinking that all of these issues are caused by bad design choices.
You're right about that, and you're right that by thinking about the architecture more, there's never a need for a partially-implemented interface.
However, that doesn't mean that the programmer who does it is bad.
We make design tradeoffs constantly, and can't expect the programmer to know the future.

So if partially-implemented interfaces can be avoided by being more careful, why would we design a language to avoid it?
Frankly, most language design decisions revolve around encouraging best practices without having to think as much.
This is the case with partially-implemented interfaces too.
We *could* just explain how to avoid it, or we could design our languages to make it unnecessary.


# What are interfaces for?

At the core, interfaces allow us to make *more readable* code.
That's a strange suggestion, since interfaces are an architectural, machine-oriented concept.
To suggest that their main benefit is human-oriented is jarring!
However, the main benefits of interfaces come from *hiding information*, and *explicit similarity*.

* They provide a specification so that *when reading* the code, you can deal with that specification and not an implementation.
* They make similar behaviour between classes explicit, to encourage consistent architectural styles.
* They allow similar classes to be handled identically

# A Solution?

Wouldn't it be much better if we could just specify everywhere which methods we support?

Currently, an interface is a protocol consisting of multiple methods.
What if we instead say that each interface should only be for one method?

This works great, like we saw earlier with `Closeable`.
It's immediately obvious that any `Closeable` parameter is given that type because we need the `close()` method.

This is a fundamental shift in how we view types.
Instead of saying *x should be y*, we say *x should do a and b and c*.
This is known as *Structural Typing*.

This approach to Structural Typing is rare because it is *method-based*, rather than *field-based*.
Most structural typing only allows you to specify an object in terms of its fields, not its methods.
Additionally, we don't actually need to specify what methods we are implementing because we can infer it.

Mention **OCaml**

I call this type system *Methodical Typing*.
Really, it's *Static Inferred Method-Based Structural Typing*.
But that's just not as catchy.

## Methodical Typing
Let's have a look at methodical typing, and how we get there through incremental improvement of the above example.

### Java

Most developers would immediately see the solution to the above example's runtime type-checking.
We need to divide the large `Stream` interface into multiple smaller interfaces.

![The Stream interface has been split into Stream and ReopenableStream](/assets/img/posts/20190730/java.svg "The updated class hierarchy")

If we implement the class hierarchy shown above, you'd get something like this:

```typescript
interface Stream{
    isOpen(): boolean;
    close(): void;
}

interface ReusableStream extends Stream{
    open(): void
}

class SingleUseStream implements Stream {
    private currentlyOpen: boolean = true;
    isOpen = () => this.currentlyOpen;
    close = () => this.currentlyOpen = false;
}

class MultiUseStream implements ReusableStream {
    private currentlyOpen: boolean = true;
    isOpen = () => this.currentlyOpen;
    close = () => this.currentlyOpen = false;
    open = () => this.currentlyOpen = true;
}

function toggleStream(stream: ReusableStream) {
    stream.isOpen() ?
        stream.close() :
        stream.open()
}

function closeIfOpen(stream: Stream) {
    if(stream.isOpen()){
        stream.close()
    }
}
```

Now, if we try the code that caused us issues, it gets picked up immediately by the type checker:

```java
SingleUseStream stream = new SingleUseStream();
StreamUtil.toggleStream(stream);
```

`togleStream(ReopenableStream) cannot be applied to (SingleUseStream)`

Meanwhile, `StreamUtil.closeIfOpen(stream)` works fine because the type is satisfied.
So, we have successfully fixed the issue with runtime type-checking!
Is this the solution promised in that heading?
Is this methodical typing?

No.
While this is much improved over the first example, it has some problems.
In this example, we only have 3 methods and 2 implementations.
It looks fine, because the complexity is exponential.

`Stream` and `ReopenableStream` need to exist because a parameter or return type must be a single interface.
These 'combination' types might need to exist for each combination of these single-method interfaces.
Whenever a class implements a combination type, it also needs to implement all combination types made from the components of the overall combination type.

An example is necessary.
Here is what it looks like with four methods and an exhaustative list of combination types:

```typescript
interface A{a(): void}
interface B{b(): void}
interface C{c(): void}
interface D{d(): void}

interface AB extends A,B{}
interface AC extends A,C{}
interface AD extends A,D{}
interface BC extends B,C{}
interface BD extends B,D{}
interface CD extends C,D{}

interface ABC extends A,B,C{}
interface ABD extends A,B,D{}
interface ACD extends A,C,D{}
interface BCD extends B,C,D{}

interface ABCD extends A,B,C,D{}

class ABCDImpl implements
    A,B,C,D,
    AB,AC,AD,BC,BD,CD,
    ABC,ABD,ACD,BCD,
    ABCD{/* ... */}
```

![Comprehensive class hierarchy for 4 methods](/assets/img/posts/20190730/extreme.svg "Class hierarchy with 4 methods")

If `ABCDImpl` does not implement `ABC`, then it could not be returned by a method promising to return `ABC`, even though it implements all the same methods.
For those of you with some maths background, you'll see the issue.
The number of interfaces needed is equal to the number of subsets of the set of methods.
If there are n methods, you need 2^n interfaces.
For 4 methods, we have 16 interfaces.
If you want 10 methods, you will need 1024 interfaces.
And don't forget, that implementation at the bottom must implement all combination interfaces, equal to 2^n - n.
When was the last time one of your classes implemented 1014 interfaces?

In fact, there's a hard limit in the JVM that a class can only implement 65535 interfaces.
Under this system, you hit that limit by implementing 16 methods.
And that's not even mentioning the fact you'd have to write it out!

There must be a better way, and it turns out that there is a concept that eliminates the need for these combination interfaces.

### Union and Intersect types
Union and intersect types allow for on-demand creation of new types by combining other types.

A 'Union Type' is a type is satisfied by a type that satisfies either of its components.
That means that a variable with the union type `String | Int` can be set to either strings or ints.
The methods available on that variable are just those available on both `String` and `Int`.
For example, you could call `toString()` on a `String|Int` object, or `length()` on a `String|Array` object.

An 'Intersect Type' is the opposite - it is satisfied only by something that satisfies both of its components.
That means that a variable of the intersect type `String & Int` can only be set to an object that is both a string and an int.
On a `String & Int` object, you can call any method from `String` or `Int`.

The one important to us is Intersect Types.
It is important that these types can be used as parameter and return types.
One language supporting Union and Intersect types is 'Ceylon'.

Here is an example of the above program rewritten in Ceylon:

```typescript
interface M_isOpen{ isOpen(): boolean }
interface M_close{ close(): void }
interface M_open{ open(): void }

class SingleUseStream implements M_isOpen, M_close{
    private currentlyOpen: boolean = true;
    isOpen = () => this.currentlyOpen;
    close = () => this.currentlyOpen = false;
}

class MultiUseStream implements M_isOpen, M_close, M_open {
    private currentlyOpen: boolean = true;
    isOpen = () => this.currentlyOpen;
    close = () => this.currentlyOpen = false;
    open = () => this.currentlyOpen = true;
}

function toggleStream(stream: M_isOpen & M_open & M_close) {
    stream.isOpen() ?
        stream.close() :
        stream.open()
}

function closeIfOpen(stream: M_isOpen & M_close) {
    if(stream.isOpen()){
        stream.close()
    }
}
```

Now, we only need to define one interface per method.
The combination interfaces can be defined on-demand as intersect types, as seen in `streamUtil`.

### Designing a methodically-typed language from the ground-up

If we started from scratch, creating a language with a methodical type system, what would it look like?
My favourite language design so far looks like this:

```typescript
method isOpen(): boolean;
method close(): void;
method open(): void;

class SingleUseStream {
	private currentlyOpen: boolean = true;
	isOpen = () => this.currentlyOpen;
	close = () => this.currentlyOpen = false;
}

class MultiUseStream {
	private currentlyOpen: boolean = true;
	isOpen = () => this.currentlyOpen;
	close = () => this.currentlyOpen = false;
	open = () => this.currentlyOpen = true;
}

function toggleStream(stream: <isOpen, close, open>) {
	stream.isOpen() ?
		stream.close() :
		stream.open()
}

function closeIfOpen(stream: <isOpen, close>) {
	if(stream.isOpen()){
		stream.close()
	}
}
```

Let's have a look through some of the interesting design points of this language:

* Methods get defined like interfaces, using the `method` keyword.
	* Unless specified, methods return `void`.	
	* Parameter and return types are written as method lists. `[Open, Close, IsOpen]` is syntactic sugar for `Open & Close & IsOpen`.
	* Interfaces can still be used, and work like a type alias for intersect types that are used frequently:
	
	```
	interface FullyFeaturedStream: [open, close, isOpen]
	```
	
	* A type is any number of methods and/or interfaces in a method list.
		
* Class signatures do not have to list the methods they implement like they would with interfaces in java.
	* We do not allow classes to implement a method that has not been defined. This means we know which methods a class implements from its body.
	* To implement a method defined in another package, use a qualified method name in the method implementation. Like in java, this method can be called using its qualified or unqualified name, assuming that it is not ambiguous.
	* Optionally, you can specify a list of methods/interfaces in the class signature. The main use of this is to enable compile-time checks that all of an interface is implemented. 
	
* When implementing a method, its parameter and return types are inferred from the method definition.
	* These should be shown automatically by an IDE.
	* A parameter type can be overriden if the implementation accepts a less specific type.
	* The return type can be overridden if the implementation returns a more specific type.


## Benefits

### WIP work

You don't need to implement the whole interface to get it to compile.
You keep the compile-time checks that you're not trying to run any code that isn't written yet.

### Overly Broad interfaces
Interfaces can be broad because you don't need to implement the whole thing any more.
With our `List` example, you can just implement `get` and then call the `first` method.
With our `Map` example, you're not forced to implement `setValue`.
In both of these cases, you keep the compile-time checks that you're not calling any code that would use those methods.

### Providing a 'sane' default
When you can't provide a sane default (as we saw previously), you don't need to.
The interface won't be intimidating, because the programmer doesn't need to implement all of it.
They can just implement the bit that they want to use.

### Simpler mocking
This is a side-point to overly broad interfaces, as we can now do mocking when unit testing much more easily.
We get compile-time checks that we have implemented the necessary parts.
We don't have to write stubs for the unnecessary parts.
We don't have to keep track in our heads which bits are written and which bits aren't.

## Disadvantages

### Documentation is even more important

### Code is more verbose

---