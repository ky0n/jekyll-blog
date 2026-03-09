---
layout: post
title:  "Book Review - Effective Java"
date:   2026-03-10 00:03:20 +0100
categories: book review
tags: [java, book, review, effective, bloch, joshua, serialization]
---

# Book Review: <u>Effective Java</u> (Author: Joshua Bloch)

# Introduction of the book and the author
Recently I finished reading "Effective Java" (3rd Edition from 2017). A book for intermediate to advanced software engineers with existing knowledge about Java. 
The book covers purely the Java programming language, no frameworks and libraries which are built on top of Java. The first edition of the book was published in 2001, 
but has been updated to around Java 9 and covers many important new features of Java 8 and 9 (Streams, Optionals, Module-System just to name a few). 
The 3rd Edition was published in 2017.

The book is divided into 12 topics consisting in total of 90 items. Each item contains a recommendation on how to use the Java programming language. 
These recommendations are directly stated in the title of the topic and are further explained in the respective item.

The author is Joshua Bloch. He wrote himself code for the JDK, including substantial parts of the `java.math` package and the Collections framework. 
He is mentioned as the author of 103 classes in the JDK and for example the original author of the `Map`, `Set`, and `Throwable` and `RetentionPolicy` Class. 
Most of the code he wrote for Java was in the early days of the Java programming language, mostly in the early 2000s.

## Content of the book
The book covers 90 items which are spread over 12 chapters. The book contains various recommendations, which are often printed in bold, on how to use the Java programming language.  
For example to only use the `Cloneable`-interface in rare cases. The `Cloneable`-interface is flawed, as it doesn't actually contain the `clone`-method, which is part of the `Object`-class.
Even though, if a class implements the `Cloneable`-interface, it's expected, that the class overrides the `clone`-method. It's not easy to implement the `clone`-method, as the newly created object should never have a mutual state with the original object.
Other than that, it's advised to always override the `toString`-method of the `Object`-class and to always override the `hashCode`-method, when the `equals`-method is overridden, otherwise hashed Collections, like the `HashMap`, will not work correctly.
Bloch argues for static factory methods instead of using constructors as they can have meaningful names, they aren't required to always create a new object and they can return any subtype.
He also argues for using the builder pattern, instead of constructors with many parameters for better readability and to avoid the 'telescoping'-problem. 
Another advice is to prefer using the try-with-resources statement, as it can close resources automatically, even in cases of errors and has a much better error handling than try-finally blocks.

# Favor composition over inheritance
The author is famous for arguing for composition over inheritance, as inheritance brings more coupling of the classes. The subclasses rely on the implementation of the parent classes, i.e. it violates encapsulation.
It's difficult to create inheritance and the author of the code has to strictly follow rules, e.g. to not invoke overridable methods in constructors. 
Bloch advises to prohibit creating subclasses, if not explicitly wanted, by either making the class final or by setting the default constructor private. 
He emphasizes that inheritance should only be used in cases where a true "is-a" relationship exists. 
If this is not the case, a wrapper class (also known as the Decorator pattern) is usually the superior design choice. 
Composition makes your code more flexible, as it avoids the "fragile base class" problem where changes to a superclass can inadvertently break subclasses. 
By using composition, you are only depending on the public API of the component, rather than its internal implementation.

# Concurrency
One of the most important takeaways from this chapter is Bloch’s view that `wait` and `notify` are effectively the "concurrency assembly language." While they are foundational, he argues that there is seldom, if ever, a reason to use them in modern Java (Item 81). Instead, we should rely on the higher-level utilities in `java.util.concurrent`.

Bloch emphasizes a few key principles for writing robust concurrent code:

*   **Prefer Executors, Tasks, and Streams to Threads (Item 80):** In the early days of Java, a `Thread` served as both the unit of work and the execution mechanism. Bloch argues for separating these: the *Task* (Runnable or Callable) is the unit of work, and the *Executor Service* is the mechanism. This decoupling gives you the flexibility to change execution policies (like thread pool sizes) without touching the business logic.
*   **Don't depend on the Thread Scheduler (Item 84):** A well-written program should not rely on the thread scheduler for correctness or performance. If your program only works when you tweak thread priorities or use `Thread.yield`, it is likely non-portable and fragile. The best approach is to ensure that the number of runnable threads is not significantly greater than the number of processors.
*   **Synchronization is about Visibility, not just Mutual Exclusion (Item 78):** Many developers think synchronization is only about stopping two threads from hitting the same code at once. Bloch reminds us that without proper synchronization (or `volatile`), one thread’s changes might never be visible to another thread due to compiler optimizations like "hoisting."
*   **Avoid Excessive Synchronization (Item 79):** He warns against "alien methods"—methods provided by clients or designed to be overridden—being called from within a synchronized block. This is a classic recipe for deadlocks or data corruption. The advice is simple: do as little work as possible inside synchronized regions.

By moving away from "concurrency assembly language" and using modern synchronizers like `CountDownLatch` or `ConcurrentHashMap`, we can write code that is not only safer but also much easier to maintain.


# Serialization
Serialization as Java's Framework for encoding objects as byte streams (serializing) and reconstructing objects from their encodings (deserializing).[1]
There are many disadvantages for using serialization in Java. The book generally advises to _not_ use serialization in new programs as there are other possibilities to use equal functionality with other frameworks and those being safer and easier to use than serialization. 
One option is to just use JSON or Protocol Buffers, also known as Protobuf. Serialization introduced significant security risks.
  
One of the main security problems of serialization is deserializing untrusted objects, which can cause remote code execution (RCE) or Denial of Service (DoS).

One big disadvantage of using serialization is that you're giving up flexibility and have to keep the classes backward compatible to previously serialized objects.
Testing correct serialization is also hard and time-costing to do. It's most of the time appropriate to provide a `readObject` method, as the default serialization has flaws, including the possible causing of stack overflows and slow performance.

Classes with custom serialization must "override" `writeObject` (for serialization) and `readObject` (for deserialization), even though there does not exist a default implementation of these two methods which is overridden.
Neither the class `Object` nor the interface `Serializable` contains the `writeObject` and `readObject`-methods. They are called through reflection by the `ObjectOutputStream` (`readObject` -> deserialization) and `ObjectInputStream` (`writeObject` -> serialization).

```java
// custom serialization (writeObject) and deserialization (readObject)
class Person implements Serializable {

    private String name;
    private transient int age; // normally not serialized
    private long id;

    private void writeObject(ObjectOutputStream out) throws IOException {
        out.defaultWriteObject();   // serialize normal fields
        out.writeInt(age);          // manually serialize transient field
    }

    private void readObject(ObjectInputStream in)
            throws IOException, ClassNotFoundException {
        in.defaultReadObject();     // read normal fields
        age = in.readInt();         // read custom field
        id = 50L;                   // id will always be 50L after deserialization
    }
}
```

Another method of the serialization framework in Java is `readResolve`, which can be overridden and is called during deserialization, just after an object has been deserialized. One use-case is to enforce that a Singleton class has only one instance, by returning that one instance in the `readResolve`-method.
```java
class Singleton implements Serializable {

    private static final Singleton INSTANCE = new Singleton();

    private Singleton() {}

    public static Singleton getInstance() {
        return INSTANCE;
    }

    private Object readResolve() {
        return INSTANCE;
    }
}
```
The `readResolve`-method is also called via reflection by the serialization mechanism of Java, which isn't a good design in my opinion.

## Summary
Josh Bloch, who authored a lot of code for the JDK itself, especially in the early 2000s, shares his deep knowledge of Java and how to/how to not use effectively the Java programming language.
I can recommend the book for non-beginner to experienced Java-Developers, who want to dig deeper into how to write clean, basic Java code and what should be avoided.
The book is iconic and still relevant today, even though it doesn't cover the most modern features of Java.
---

[1]: Bloch, J. (2018). Effective Java (3rd ed.). Addison-Wesley Professional.
