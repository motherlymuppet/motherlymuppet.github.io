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

> A Partially-Implemented Interface is a class which implements an interface but does not support some of its methods, and instead throws runtime exceptions when they are called

Let's just get stuck in and have a look at an example.
The code snippet below implements the class hierarchy shown.
The `SingleUseStream` class is a partially-implemented interface.
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

This example code is simplistic compared to what we see in the wild, but similar code shows up everwhere.
We define a Stream interface, which declares three methods.
It has two implementations, and there are two static functions which accept a stream parameter.
The `SingleUseStream` class is a partially-implemented interface.

It declares that it implements `Stream`, then does not impelment the `open()` method.
For example, this snippet will always throw an error, but has no compile-time warnings or errors.

```typescript
const stream = new SingleUseStream();
toggleStream(stream);
```

## What's Wrong With That?

This anti-pattern is one example of violating the *Liskov Substitution Principle* (LSP), one of the five SOLID principles.
The principle states that all implementations of an interface should be interchangeable.
In our case, `SingleUseStream` and `MultiUseStream` are not interchangeable.#
Violating the LSP results in code that is hard to debug and hard to reason about.

Using `SingleUseStream` successfully is very fiddly.
The programmer needs to understand the implementation of every method that they pass it to.
If any of the implementations call `open()`, the code will crash.
In fact, if they ever pass a `SingleUseStream` object to a library function, then updating that library could lead to breaking changes.
The library creators would not think twice about calling `open()` on your object, and they definitely wouldn't report it in the changelog!

When this anti-pattern is used frequently throughout a codebase, you will learn to not trust the compiler when it says there are no issues.
This is because you have replaced the compile-time type checks with run-time checks.
At that point, you have all the boilerplate code and inflexibility of a statically-typed language, while also having the intermittent bugs and worse readability of a dynamically-typed language!

At this point, I hope I've sold you on the idea that you should avoid partially-implemented interfaces.
So why are they so common?
Do programmers just not know that it's bad?
Do they not care?

In fact, there is often no good alternative due to the limitations of the language and the pre-existing code base.

## It's Sometimes Necessary?

Nobody likes writing `throw 'Not Supported'`.
It feels hacky, and like it will come back to bite you.
It's not laziness though, it's a lack of other options.

### Work-In-Progress Code

When creating a new feature, you want to compile often to check that the code is right.
The interface is a specification for a class, so you want to write it first.
Now you can't compile, since you haven't implemented the entire interface!
There's only two options, you can implement everything and then compile, or you can just create method stubs that throw an error when called.
Of course, you do the latter, and honestly I agree with that!

This is probably the least harmful use of a partially-implemented interface.
It is temporary, and the programmer already understands the codebase, so will know when the method gets called.
However, in classic `TODO` fashion, some will never actually be implemented.
Years later, a new programmer looks at the interface, reads the docs, and calls a method that isn't actually implemented.
Even worse, just one branch of an `if` statement was forgotten about, and the code works most of the time, and throws an error on an obscure edge case that wasn't tested.

### Attempting to provide a 'sane' default

In java, `AbstractList` throws an `UnsupportedOperationException` whenever you call any of:

* `set`
* `add`
* `remove`

I can only assume that this is to make it less intimidating to extend `AbstractList`.
When doing so, the only method that you must implement is `get`.
These default 'implementations' (sarcastic air quotes) allow it to compile with just that.

Frankly, this is horrendous.
Any programmer implementing AbstractList will assume that the methods with default implementations are actually useful.
There is no reason, until it starts throwing exceptions, to look at the `AbstractList` source.

### Overly Broad Interfaces

This is the broadest, and most common scenario.
Sadly, it's also the hardest to fix.

If an interface is too broad, it is usually because the original developer did not consider a use case where an implementation would not need one of the methods.
This means that the interface declares a method which future implementations do not, or cannot implement.
By the time anyone realises that the interface is too broad for their use case, it is entrenched.
Let's consider a real-life example.

Java's `ConcurrentHashMap.Node` implements `Map.Entry`, which declares `setValue`.
To compile, it must implement the method, but allowing values to be changed would result in race conditions when used concurrently.
The developers had three choices:

* Implement the method and accept that it won't work sometimes
* Don't implement the method, and don't inherit from `Map.Entry`
* Use a method stub that throws an error

The first solution is clearly awful, as it would produce obscure and intermittent bugs caused by race conditions.
The second option really isn't much better.
All throughout java, functions take `Map.Entry` as a parameter.
Creating a data structure in the standard library that was incompatible with the rest of the collections API would be a disaster, and you'd end up rewriting the whole API.
We can easily rule out that option too.
That leaves the final option, to create method stubs which throw an error.
Again, I agree that this is the best option!

What makes this an anti-pattern is that it seems like the best option at the time.
It's only later on that you realise the damage it caused.
Like when you accept a `Map` as a parameter and call `setValue` on one of the nodes, only to realise that code elsewhere has changed and your parameter is now a `ConcurrentHashMap` and your code throws errors.

`Map.Entry` is an Overly Broad Interface as they did not consider that some entries may be read-only.
This *could* be solved by providing a more precise inheritance hierarchy - we could separate maps into read-only and read-write maps.
Kotlin, another JVM language, does that in its standard library:

![Kotlin Map and MutableMap class hierarchy](/assets/img/posts/20190730/map.svg "Kotlin Map and MutableMap class hierarchy")

# A Way Forwards

Our first code snippet had serious issues, but a lot of you already have ideas about how to fix it.
Like we saw with Kotlin's immutable map, let's rework our inheritance hierarchy to make it more precise.
To do that, we split our large `Stream` interface into two smaller interfaces, like this:

![The Stream interface has been split into Stream and ReopenableStream](/assets/img/posts/20190730/java.svg "The updated class hierarchy")

Which translates to the following code snippet:

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

Now when we try the code that threw an error before, the compiler throws an error since `toggleStream` does not accept a parameter of type `Stream`.

```typescript
const stream = new SingleUseStream();
toggleStream(stream);
```

Problem solved, smaller interfaces are the way forwards and all interfaces should be tiny!

## All Interfaces Should be Tiny?

That's an interesting proposal.
What happens if we do make all our interfaces tiny?
Currently, an interface is a specification declaring multiple required methods.
What would happen if we limited ourselves to interfaces that only declared one method?
If we did that, how is declaring a that a class implements an interface any different to declaring that a class implements a method?

The obvious difference is that when using single-method interfaces, multiple classes can declare that they implement the same method.
We end up in a situation where each class and function just declares which methods they support.

This is a fundamental shift in how we view types.
Instead of saying *x should **be** y*, we say *x should **do** a and b and c*.
This is known as *Structural* typing, as opposed to *Nominal* typing.

We know that an object can be passed to a function because the types are compatible.
With nominal typing, we know that because we check the declared type of the object and the delcared type of the function.
If they are the same type, or if the object is a sub-type of the function parameter type, they are compatible and the call is allowed.

Comparatively, with structural typing, we instead check that every property on the function parameter type is also present on the object.
This probably sounds familiar to you, even if you don't have a name for it.
Javascript, and many other interpreted languages work like this.
In javascript, when we call a function on an object, the interpreter looks at the object's prototype and finds the function if it exists.
It does not care what the actual type of the object is.

*duck typing goes here* //TODO

With some nominally-typed languages, we can pretend that they are structurally typed.
In fact, I think that this in-between may have benefits over both purely nominal and purely structural type systems.

## Pretending To Be Structural



This approach to Structural Typing is rare because it is *method-based*, rather than *field-based*.
Most structural typing only allows you to specify an object in terms of its fields, not its methods.
Additionally, we don't actually need to specify what methods we are implementing because we can infer it.

I call this type system *Methodical Typing*.
Really, it's *Static Inferred Method-Based Structural Typing*.
But that's just not as catchy.

## Methodical Typing
Let's have a look at methodical typing, and how we get there through incremental improvement of the above example.

### Java

Most developers would immediately see the solution to the above example's runtime type-checking.
We need to divide the large `Stream` interface into multiple smaller interfaces.



If we implement the class hierarchy shown above, you'd get something like this:



Now, if we try the code that caused us issues, it gets picked up immediately by the type checker:



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