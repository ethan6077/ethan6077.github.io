---
layout: post
title: No Class in Scala
categories: [scala, tech]
---

## What shall we use in stead without class & new

```scala

object MyObj {
  def apply(a: A, b: B): F = {
	...
  }
}

```

## Why


Why Scala Prefers object.apply() Over new Class()
The object-apply pattern in Scala offers several key advantages:

### 1. Flexibility in Return Types

The apply method can return:
- The exact class type
- A subtype or trait implementation
- A wrapped type (like Option, Either, or in your case, Resource[IO, Server])
- Different types based on parameters
- In the code agove , MyOjb.apply() returns F, not an MyOjb instance. With new, you're locked into creating an instance of that specific class.

### 2. Factory Pattern Built-In
 
The apply method acts as a factory method without ceremony. You can:

// Clean syntax - looks like a constructor

```scala
val o = MyObj(a, b)
```

// Instead of verbose factory pattern
```scala
val o = MyObj.create(a, b)
```

### 3. Companion Object Benefits

When object AppServer is a companion to class AppServer, the object can:

- Access private constructors
- Hide implementation details
- Provide alternative constructors with different parameter sets
- Perform validation or complex initialization logic

### 4. Uniform Access Principle

Users don't need to know if they're calling a constructor, a factory method, or retrieving a cached instance:

```scala
val list = List(1, 2, 3)        // Actually calls List.apply
val server = AppServer(80, app)  // Calls AppServer.apply
```

### 5. Smart Constructors

The apply method can perform validation, resource management, or return errors before construction:

```scala
def apply(port: Int, service: HttpApp[IO]): Resource[IO, Server] = {
  // Can validate port range
  // Can set up resources safely
  // Can return error types like Option or Either
  BlazeServerBuilder[IO]
    .bindHttp(port, "0.0.0.0")
    .withHttpApp(service)
    .resource
}
```

### 6. Better for Immutable Data

For case classes, Scala auto-generates `apply` in the companion object:

```scala
case class Person(name: String, age: Int)
val p = Person("Alice", 30)  // No 'new' needed
```

### 7. Composability

`apply` is just a method, so it can be passed around as a function:

```scala
val configs = List((a, b), (a, b))
val servers = configs.map((AppServer.apply _).tupled)
```


