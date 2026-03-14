---
layout: page
title: "Java Streams Cheat Sheet"
permalink: /cheatsheets/java-streams/
---

_Updated for Java 25 — includes Gatherers (JEP 485)._

## Stream erzeugen

```java
// Aus Collection
list.stream()
list.parallelStream()

// Aus Werten
Stream.of("a", "b", "c")
Stream.empty()

// Aus Array
Arrays.stream(array)
Arrays.stream(array, startInclusive, endExclusive)

// Generieren
Stream.iterate(0, n -> n + 2)               // 0, 2, 4, 6, ...
Stream.iterate(0, n -> n < 100, n -> n + 2) // mit Abbruchbedingung (Java 9+)
Stream.generate(Math::random)               // unendlicher Stream

// Primitive Streams
IntStream.range(0, 10)        // 0..9
IntStream.rangeClosed(1, 10)  // 1..10
LongStream.of(1L, 2L, 3L)

// Aus String
"hello".chars()               // IntStream

// Aus Datei
Files.lines(Path.of("file.txt"))
```

## Intermediate Operations

```java
.filter(x -> x > 5)              // Filtern
.map(String::toUpperCase)         // Transformieren
.mapToInt(String::length)         // zu IntStream
.flatMap(list -> list.stream())   // 1:n Transformation, flatten

// mapMulti (Java 16+) — performanter als flatMap bei wenigen Elementen
.<String>mapMulti((element, consumer) -> {
    if (element.startsWith("A")) {
        consumer.accept(element.toUpperCase());
        consumer.accept(element.toLowerCase());
    }
})

.distinct()                       // Duplikate entfernen
.sorted()                         // natürliche Sortierung
.sorted(Comparator.reverseOrder())
.sorted(Comparator.comparing(Person::age))

.peek(System.out::println)        // Debugging (nicht für Side-Effects!)
.limit(10)                        // Erste n Elemente
.skip(5)                          // Erste n überspringen

.takeWhile(x -> x < 100)         // Nimmt solange Bedingung wahr (Java 9+)
.dropWhile(x -> x < 10)          // Überspringt solange Bedingung wahr (Java 9+)
```

## Terminal Operations

```java
// Sammeln
.toList()                              // Unmodifiable List (Java 16+)
.toArray(String[]::new)
.collect(Collectors.toList())          // Mutable List
.collect(Collectors.toSet())
.collect(Collectors.toCollection(TreeSet::new))

// Reduzieren
.count()
.min(Comparator.naturalOrder())        // Optional<T>
.max(Comparator.naturalOrder())        // Optional<T>
.reduce(0, Integer::sum)
.reduce((a, b) -> a + b)              // Optional<T>

// Suchen
.findFirst()                           // Optional<T>
.findAny()                             // Optional<T> (parallel-freundlich)

// Prüfen
.anyMatch(x -> x > 5)                 // boolean
.allMatch(x -> x > 0)
.noneMatch(x -> x < 0)

// Ausgeben
.forEach(System.out::println)
.forEachOrdered(System.out::println)   // Reihenfolge garantiert
```

## Collectors

```java
// Strings zusammenfügen
Collectors.joining(", ")
Collectors.joining(", ", "[", "]")     // mit Prefix/Suffix

// Gruppieren
Collectors.groupingBy(Person::city)                         // Map<City, List<Person>>
Collectors.groupingBy(Person::city, Collectors.counting())  // Map<City, Long>
Collectors.groupingBy(Person::city,
    Collectors.mapping(Person::name, Collectors.toList()))   // Map<City, List<String>>

// Partitionieren (boolean-Gruppen)
Collectors.partitioningBy(n -> n % 2 == 0)                  // Map<Boolean, List<T>>

// Statistiken
Collectors.summarizingInt(Person::age)                      // IntSummaryStatistics

// Zu Map
Collectors.toMap(Person::id, Person::name)
Collectors.toMap(Person::id, Person::name, (a, b) -> a)     // Merge bei Duplikaten
Collectors.toUnmodifiableMap(Person::id, Person::name)

// Downstream-Collectors verschachteln
Collectors.groupingBy(Person::dept,
    Collectors.collectingAndThen(
        Collectors.maxBy(Comparator.comparing(Person::salary)),
        Optional::get))

// Teeing — zwei Collectors parallel, Ergebnisse zusammenführen (Java 12+)
Collectors.teeing(
    Collectors.minBy(Comparator.naturalOrder()),
    Collectors.maxBy(Comparator.naturalOrder()),
    (min, max) -> new Range(min.get(), max.get()))
```

## Stream Gatherers (Java 24+, JEP 485)

Gatherers sind benutzerdefinierte Intermediate Operations — das Gegenstück zu Collectors.

```java
// Built-in Gatherers
import java.util.stream.Gatherers;

// Feste Fenster: [1,2,3,4,5] → [[1,2],[3,4],[5]]
stream.gather(Gatherers.windowFixed(2))

// Gleitende Fenster: [1,2,3,4] → [[1,2],[2,3],[3,4]]
stream.gather(Gatherers.windowSliding(2))

// Fold — alle Elemente zu einem Ergebnis (wie reduce, aber mit Initialwert und anderem Typ)
stream.gather(Gatherers.fold(() -> "", (str, el) -> str + el))

// Scan — wie fold, gibt aber jeden Zwischenzustand aus
stream.gather(Gatherers.scan(() -> 0, Integer::sum))
// [1,2,3] → [1, 3, 6]

// mapConcurrent — parallele Verarbeitung mit Virtual Threads
stream.gather(Gatherers.mapConcurrent(10, this::fetchFromApi))
```

### Eigenen Gatherer schreiben

```java
// Beispiel: Nur Elemente durchlassen, die sich vom Vorgänger unterscheiden
Gatherer<String, ?, String> dedupConsecutive = Gatherer.ofSequential(
    () -> new Object() { String last = null; },             // Initializer (State)
    (state, element, downstream) -> {                        // Integrator
        if (!element.equals(state.last)) {
            state.last = element;
            return downstream.push(element);                 // true = weitermachen
        }
        return true;
    }
);

stream.gather(dedupConsecutive).toList();
// ["a","a","b","b","a"] → ["a","b","a"]

// Mit Finisher (wird am Ende aufgerufen)
Gatherer.ofSequential(initializer, integrator, finisher)

// Parallel-fähig (mit Combiner)
Gatherer.of(initializer, integrator, combiner, finisher)
```

## Primitive Streams

```java
IntStream / LongStream / DoubleStream

.sum()
.average()    // OptionalDouble
.summaryStatistics()

// Boxing
intStream.boxed()         // Stream<Integer>
intStream.asLongStream()  // LongStream
intStream.asDoubleStream()

// Objekt-Stream → Primitiv
stream.mapToInt(String::length)
stream.flatMapToInt(...)
```

## Tipps & Fallstricke

```java
// Stream kann nur einmal konsumiert werden!
Stream<String> s = list.stream();
s.count();
s.count(); // IllegalStateException!

// toList() ist unmodifiable — kein add/remove möglich
var list = stream.toList(); // UnsupportedOperationException bei Mutation

// parallel() nicht blindlings verwenden
// Gut bei: CPU-intensiv, große Datenmengen, unabhängige Operationen
// Schlecht bei: I/O, kleine Datenmengen, gemeinsamer Zustand

// Reihenfolge bei parallel Streams
parallelStream.forEachOrdered(...)  // erzwingt Reihenfolge
parallelStream.forEach(...)         // beliebige Reihenfolge

// Optional richtig verwenden
stream.findFirst()
    .map(String::toUpperCase)
    .orElse("default");
```
