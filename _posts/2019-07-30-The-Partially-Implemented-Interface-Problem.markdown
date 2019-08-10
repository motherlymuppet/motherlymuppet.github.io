---
layout: post
title: The Partially-Implemented Interface Problem
date: 2019-07-22 18:00:00 +0100
description: What's wrong with a partially-implemented interface, and how could we it better?
img: posts/20190722/sharpshot.png
tags: [Green-Field Thinking]
---

I originally had the idea for this blog post around a year ago.
An awful lot of thought went into it, as it's really not based on much other work.
A lot of what I discussed here is stuff that I have discovered and reasoned through from scratch.
That doesn't mean that I'm saying nobody has done this before, just that I haven't seen it.

This post will be a slow and methodical journey through a serious of discussions:

1. Why we use interfaces
1. A discussion of the use of partially implemented interfaces
	a. Why they are used
	a. Their issues
1. Introducing methodical type systems - my suggestion for fixing the problem

# Why we use interfaces

If you asked a programmer, you would get one of many answers:

* To hide implementation details
* To centralise decisions about protocols
* 
* 

At the core, interfaces allow us to make *more readable* code.
That probably sounds like a strange thing to say, as interfaces are a fairly low-level, machine-oriented concept.
To suggest that their main benefit is human-oriented is jarring!
 
However, consider the answers above.
We hide implementation details so that *when reading* the code, you don't have to worry about the implementation.
This may already be the case for some code, but by using an interface rather than a concrete type, the programmer explicitly says that you don't need to worry about the implementation details. 

## Less information means less to worry about

Consider the following comparison

```java
public void confusingMethod(Closeable confusingParam){
```
This code tries hard to give us no hint as to what it will do.
However, we can look at the type of `confusingParam` and immediately know what will happen.
The only method on `Closeable` is `close()`.
The programmer has gone to the effort of making this code generic and indicated that this code will not do anything other than closing the stream.

```java
public void confusingMethod(URLClassLoader confusingParam){
```
Compared to the previous code, there is more to think about here.
URLClassLoader is a complex class with 40+ methods available.
You expect some of them to be used, otherwise why did the programmer use such a specific type?

Reading the first code block, I expect it to call `confusingParam.close()`.
In the second code block, I expect it to be some reflection-based black magic.
Which means I need to jump into `confusingMethod`, seeing the source code:

```java
public void confusingMethod(URLClassLoader confusingParam){
	confusingParam.close();
}
```

Interfaces are simpler than concrete classes.
They can do less, so we know what to expect from the code.
When we know what to expect, we can correctly make assumptions about the code.
So readability is improved.

## Centralised protocols are simpler

Another Comparison!
```java
public void confusingMethod(Closeable confusingParam){
```

Again, this is very obvious.

```java
public void confusingMethod(MyCloseable confusingParam){
```
Alarm bells are ringing.
The programmer is doing something strange, and there must be a reason that they aren't using the standard interface.
They may use a different protocol to the standard library.
All my assumptions have disappeared:

* If I close this, does it stay closed?
* Can I close it again?
* Can I open it again?
* What else does it do?

I wouldn't be comfortable touching `confusingParam` until I had looked through the entire class.
When I jump into the class, I see the source code:

```java
public class MyCloseable implements Closeable{}
```

Allowing the reader to make assumptions is good.
Making that unnecessary is better.

## Explicit Similarity allows Code Reuse

When there is similarity between multiple methods, we extract the similar part into a helper method.
When classes are similar, we extract that part into an interface.

By extracting similar code, the reader already knows how it works


 

# Partially-Implemented Interfaces

> A partially-implemented interface is an interface where the concrete implementation does not implement all the methods defined.

Since the compiler enforces all methods to be implemented, often method stubs are generated which throw a runtime exception if called.
The code below shows an example of a partially-implemented interface.
We will refer back to it throughout the rest of this post.

```java
interface Stream {
    void open();
    void close();
    boolean isOpen();
}

class SingleUseStream implements Stream {
    private boolean currentlyOpen = true;
    //This line is the important one!
    public void open() { throw new UnsupportedOperationException(); }
    public void close() { currentlyOpen = false; }
    public boolean isOpen() { return currentlyOpen; }
}

class MultiUseStream implements Stream {
    private boolean currentlyOpen = true;
    public void open() { currentlyOpen = true; }
    public void close() { currentlyOpen = false; }
    public boolean isOpen() { return currentlyOpen; }
}

class StreamUtil {
    public static void toggleStream(Stream stream) {
        if (stream.isOpen()) {
            stream.close();
        } else {
            stream.open();
        }
    }

    public static void closeIfOpen(Stream stream){
        if(stream.isOpen()){
            stream.close();
        }
    }
}
```

This code is very simple, but the issue is immediately clear.
Any calls to `SingleUseStream.open()` will always throw an exception and will not be caught during compilation.
Partially-implemeted interfaces are rarely this blatant, but we still see `UnsupportedOperationException` [thrown frequently in the java standard library](https://docs.oracle.com/javase/8/docs/api/java/lang/class-use/UnsupportedOperationException.html).

For clarity's sake, this is the code that will reveal the issue:

```java
SingleUseStream stream = new SingleUseStream();
StreamUtil.toggleStream(stream);
```

`SingleUseStream` implements `Stream`, which means that the type checker believes it can do everything that `toggleStream` asks of it.
In reality, `toggleStream` will call `Stream.open()`, which is not implemented and will throw an exception.
To successfully write code using `SingleUseStream`, the programmer must keep in mind its limitations and check that `Stream.open()` is never called.
The programmer must treat it as though they are writing code in a dynamically-typed language, and keep in mind that the type checker won't find issues.
This has all the downsides that using a dynamically-typed language does in general.
Additionally, a programmer new to the codebase won't expect to have to treat it like this and will trust the compiler when it says there are no issues.

Basically, a partially-implemented interface represents a programmer hitting the limitations of the program's type system.
To get the code to work, they choose to disable the type checks for those methods, and deal with it at runtime instead.
Just like with dynamic type systems, it is easier to make mistakes and there is no way to know about the mistake until a strange edge-case hits the not-implemented method.

## Why does it happen?
WIP code
Overly broad interfaces

# A Solution?
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

I call this type system *Methodical Typing*.
Really, it's *Static Inferred Method-Based Structural Typing*.
But that's just not as catchy.

## Methodical Typing
Let's have a look at methodical typing, and how we get there through incremental improvement of the above example.

### Java

Most developers would immediately see the solution to the above example's runtime type-checking.
We need to divide the large `Stream` interface into multiple smaller interfaces.
The resulting code would look like this:

```java
interface Openable{ void open(); }
interface Closeable { void close(); }
interface OpenCheckable { boolean isOpen(); }

interface Stream extends Closeable, OpenCheckable{}
interface ReopenableStream extends Stream, Openable{}

class SingleUseStream implements Stream {
    private boolean currentlyOpen = true;
    public void close() { currentlyOpen = false; }
    public boolean isOpen() { return currentlyOpen; }
}

class MultiUseStream implements ReopenableStream {
    private boolean currentlyOpen = true;
    public void open() { currentlyOpen = true; }
    public void close() { currentlyOpen = false; }
    public boolean isOpen() { return currentlyOpen; }
}

class StreamUtil {
    public static void toggleStream(ReopenableStream stream) {
        if (stream.isOpen()) {
            stream.close();
        } else {
            stream.open();
        }
    }

    public static void closeIfOpen(Stream stream){
        if(stream.isOpen()){
            stream.close();
        }
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

```java
interface A{void a();}
interface B{void b();}
interface C{void c();}
interface D{void d();}

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

class ABCDImpl implements ABCD, ABC, ABD, ACD, BCD, AB, AC, AD, BC, BD, CD{ ... }
```

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

```ceylon
interface Openable{ shared formal void open(); }
interface Closeable { shared formal void close(); }
interface OpenCheckable { shared formal Boolean isOpen(); }

class SingleUseStream() satisfies Closeable & OpenCheckable {
    variable Boolean currentlyOpen = true;
    shared actual void close() { currentlyOpen = false; }
    shared actual Boolean isOpen() { return currentlyOpen; }
}

class MultiUseStream() satisfies Closeable & Openable & OpenCheckable {
    variable Boolean currentlyOpen = true;
    shared actual Boolean isOpen() { return currentlyOpen; }
    shared actual void close() { currentlyOpen = false; }
    shared actual void open() { currentlyOpen = true; }
}

object streamUtil {
    shared void toggleStream(Closeable & Openable & OpenCheckable stream) {
        if (stream.isOpen()) {
            stream.close();
        } else {
            stream.open();
        }
    }

    shared void closeIfOpen(Closeable & OpenCheckable stream){
        if(stream.isOpen()){
            stream.close();
        }
    }
}
```

Now, we only need to define one interface per method.
The combination interfaces can be defined on-demand as intersect types, as seen in `streamUtil`.

### Designing a methodically-typed language from the ground-up

If we started from scratch, creating a language with a methodical type system, what would it look like?
My favourite language design so far looks like this:

```
method open()
method close()
method isOpen(): Boolean

class SingleUseStream {
    private var currentlyOpen = true
    method close() { currentlyOpen = false }
    method isOpen() { return currentlyOpen }
}

class MultiUseStream {
    private var currentlyOpen = true
    method open() { currentlyOpen = true }
    method close() { currentlyOpen = false }
    method isOpen() { return currentlyOpen }
}

method toggleStream(stream: [Open, Close, IsOpen])
method closeIfOpen(stream: [Close, IsOpen])

object StreamUtil {
    method toggleStream(stream) {
        if (stream.isOpen()) {
            stream.close()
        } else {
            stream.open()
        }
    }

    method closeIfOpen(stream) {
        if (stream.isOpen()) {
            stream.close()
        }
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
	
	* A type can be any of:
		* A single method, interface, or class
		* A list of methods and/or interfaces
		
* Class signatures do not list methods implemented like they would with interfaces
	* We do not allow classes to implement a method that has not been defined. This means we know which methods a class implements from its body.
	* To implement a method defined in another package, use a qualified method name in the method implementation. Like in java, this method can be called using its qualified or unqualified name, assuming that it is not ambiguous.
	
* When implementing a method, its parameter and return types are inferred from the method definition.
	* These should be shown automatically by an IDE.
	* A parameter type can be overriden if the implementation accepts a less specific type.
	* The return type can be overridden if the implementation returns a more specific type.


## Benefits
Compile-time checks for WIP work
Simple mocking

---