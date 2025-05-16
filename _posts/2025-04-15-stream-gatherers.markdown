---
layout: post
title:  "Einblick in Javas Stream Gatherers"
date:   2025-04-15 17:50:42 +0100
categories: jekyll update
tags: [java, stream, gatherer, jdk, java24]
---

## Neues _finalisiertes_ Feature in Java 24

Bereits in Java 22 und Java 23 sind die Stream-Gatherers enthalten, allerdings noch als Preview.
In der neuesten **Java Version 24**, die am 18.03.2025, während der gleichzeitig stattfindenden JavaOne 2025, 
released wurde, sind Stream Gatherers nun als finalisiertes Feature enthalten.

Stream Gatherers kamen finalisiert mit [JEP 485](https://openjdk.org/jeps/485) in Java 24. 
Es gab keine Änderungen bei dem Stream Gatherers zum Preview in Java 23.

Stream Gatherers sind eine **neue intermediate Stream-Operation**. Intermediate, da die Methode `gather(Gatherer<? super T, ?, R> gatherer)`
selbst wieder ein `Stream<R>` zurückgibt. Der Typ-Parameter `R` wird für diese Methode speziell gesetzt, daher kann der `Gatherer`
auch den Typen der Streams verändern.

## Warum braucht es diese neue Stream-Operation auf dem Stream-Interface?

Das Problem aller vorher bestehenden intermediate Stream-Operationen ist, dass diese keinen eigens-definierbaren State innerhalb der Stream-Operation haben.
Es gibt bereits ìntermediate Stream Operationen, die einen internen State haben, beispielsweise `distinct`. Hierbei werden duplikate Elemente anhand der 
`Object#equals(Object)`-Methode herausgefiltert. Allerdings ist hier ganz genau vorgegeben welcher State gespeichert wird und `distinct` ist nur sehr limitiert einsetzbar.

Für die zweite Art von Stream-Operationen, den terminal Operationen, gibt es bereits die generisch einsetzbare `collect(Collector<? super T, A, R> collector)` Methode,
welche einen `Collector`-Objekt als Parameter erwartet. Beispielsweise mithilfe der accumulator-Funktion eines Collectors kann ein State berücksichtigt werden.
Der Accumulator ist wie folgt auf dem Collector-Interface definiert `BiConsumer<A, T> accumulator()` und muss da es eine abstrakte Methode ist, zwingend implementiert werden.
Innerhalb der Implementierung des Accumulator werden den groups (Parameter 1) jeweils ein neues Element entweder hinzugefügt oder nicht. Der accumulator wird für jedes Element aufgerufen
und hierbei kann anhand der bereits zu den groups hinzugefügten Elementen entschieden werden, ob ein weiteres hinzugefügt wird oder nicht. 

## Beispiel eines selbsterstellen Stream Gatherers

Stream Gatherers können granular selbst erstellt werden mittels Implementierung des neuen `java.util.stream.Gatherer`-Interfaces.
Interessanterweise ist dieses Interface ein funktionales Interface, obwohl die `@FunctionalInterface` Annotation nicht gesetzt wurde.

Ein Gatherer besteht im Kern aus 4 Bestandteilen. Diese sind ein Initializer, ein Integrator, ein Combiner und ein Finisher.
  - Initializer: `Supplier<A> initializer()`, welcher den initialen Zustand erzeugen kann. Im Default wird kein initialer Zustand erzeugt und `null` zurückgegeben.
  - Integrator: `Integrator<A, T, R> integrator()`
    - einzig abstrakte Methode auf dem `Gatherer`-Interface
    - `Integrator` selbst ist auch ein funktionales Interface
    - `boolean integrate(A state, T element, Downstream<? super R> downstream)` muss implementiert werden
  - Combiner: `BinaryOperator<A> combiner()`
    - nimmt zwei zwischenzeitliche Streams und fasst sie in einen zusammen
    - muss implementiert sein für parallel ausgeführte gatherer-Stream operationen
    - im default wird eine `UnsupportedOperation`-Exception geworfen, daher kann hier der Gatherer nur sequentiell durchlaufen werden
  - Finisher: `BiConsumer<A, Downstream<? super R>> finisher()`
    - finale Aktion am Ende der Stream-operation.
    - im default ein no-op

Auf dem Gatherer-Interface gibt es mehrere ofSequential- (zwingend sequentiell) und of (parallelisierbar)-Factory Methoden, um Gatherer-Objekte zu erstellen.  
Sehr simpel ist beispielsweise ofSequential auf dem Gatherer-Interface
```java 
  static <T, R> Gatherer<T, Void, R> ofSequential(
          Integrator<Void, T, R> integrator) {
      return of(
              defaultInitializer(),
              integrator,
              defaultCombiner(),
              defaultFinisher()
      );
  }
```
Es wird ein Gatherer zurückgegeben, der sequentiell und stateless ist und allein durch den ``integrator`` an die jeweiligen Bedürfnisse angepasst wird. 


## Vordefinierte Stream Gatherers im JDK

Die `final` und nicht-initiierbare Klasse `java.util.Stream.Gatherers` beinhaltet fünf vordefinierte Gatherers.  
Diese sind die folgenden:
- **fold**
  - führt eine geordnete, reduction-like Transformation durch.
  - stateful many-to-one gatherer
  - erwartet 2 Argumente: `Supplier` für den initialen Wert, `BiFunction` als folding operation,   
    - wobei bei der `BiFunction` der erste Parameter das derzeitig bestehende Endresultat ist und
    - der zweite Parameter das jeweilige Element darstellt.
  - nützlich um ein Endresultat zu ermitteln basierend auf mehreren Elementen des Streams
- **scan**
  - führt eine geordnete Transformation durch, wobei für jedes Element ein neues Element in den resultierenden Stream kommt
  - wie bei `fold` erwartet `scan` 2 Argumente: `Supplier` für den initialen Wert und eine `BiFunction`
    - das erste Argument der `BiFunction` ist der derzeitige _state_, der in jeder Iteration verändert werden kann
    - das zweite Argument ist ein Element des Streams, d.h. die BiFunction wird für jedes Element des Streams aufgerufen.
  - anders als bei fold wird nicht ein Stream mit einem singulären Element zurückgegeben, sondern die Anzahl der Elemente des Streams ändern sich _nicht_
  - stateful one-to-one gatherer
- **mapConcurrent**
  - führt mitgegebene `Function` (Parameter 2) nebenläufig aus mithilfe von virtual threads
  - Anzahl der virtual threads wird als `int` mit Parameter 1 festgelegt
  - one-to-one gatherer, **ohne state innerhalb des Gatherers**
- **windowFixed**
  - es gibt 2 window methoden in der Gatherer Klasse
  - die erste ist `windowFixed`
  - diese erwartet nur 1 Parameter. Ein int, der die window-size angibt.
  - die Elemente des Streams werden dann in Listen aufgeteilt entsprechend der window-size
  - Jedes Element ist auch in den Sub-Listen nur insgesamt ein mal vorhanden
  - Beispielsweise:
    ```java 
    List<List<Integer>> windows =
           Stream.of(1,2,3,4,5,6,7,8).gather(Gatherers.windowFixed(3)).toList();
    // will contain: [[1, 2, 3], [4, 5, 6], [7, 8]]
    ```
  - many-to-many gatherer, stateful um derzeitige Anzahl Elemente je Liste _mitzuzählen_ 
- **windowSliding**
  - die zweite window methode in der Gatherer Klasse
  - auch hier wird nur 1 Parameter erwartet, welcher wieder die windowSize angibt
  - hier werden die _windows_ jeweils auch mit den Elementen des vorherigen _windows_ (Liste) erstellt, nur das _älteste_ Element fällt jeweils raus
  - Beispielsweise:
    ```java
     List<List<Integer>> windows6 =
           Stream.of(1,2,3,4,5,6,7,8).gather(Gatherers.windowSliding(6)).toList();
     // will contain: [[1, 2, 3, 4, 5, 6], [2, 3, 4, 5, 6, 7], [3, 4, 5, 6, 7, 8]]
    ```
  - many-to-many gatherer, stateful um derzeitige Anzahl Elemente je Liste _mitzuzählen_
  - beide window-Gatherer erstellen jeweils unmodifiable Listen

## _Fazit & Meine Meinung_

Es ist die erste neue Stream-Operation seit der Einführung von der intermediate operation `mapMulti` und der terminal operation `toList` in Java 16.
Daher ist es eine durchaus bedeutende Änderung, insbesondere da das **Stream-Interface** eines der meistgenutzten modernen Java-Features ist.
Neue Features werden zu Java eher spärlich und nach intensiver Abwägung hinzugefügt. Mittels des generischen Stream-Gatherers können nun Entwickler selbst diverse 
intermediate Stream-Operationen kreieren.
  
Auch ich hatte schon mehrmals die Erfahrung gemacht im Job, dass ich eine Stream-Pipeline ohne einen vorhandenen State innerhalb des Streams nicht schreiben konnte.
Vor Java 24 gab es hier mehrere Möglichkeiten. Es konnte der Stream ganz terminiert werden, um die zustandsbehaftete Operation dann ausserhalb der Stream-API durchzuführen.
Andere Optionen sind die schon existierende zustandsbehaftete Stream-Operation `distinct`, wobei dazu teilweise die `equals` und `hashCode` Methoden des jeweiligen Elemententypes
entsprechend den Anforderungen unschön angepasst werden muss. Eine weitere Option, die ich auch bereits verwendet hatte, ist einen Collector dafür zu nutzen.
Mittels `Collector.of()` kann anhand des `BiConsumer`-Accumulators entschieden werden ob Elemente des Streams in die neu erstellte Kollektion kommen anhand der schon vorhandenen Elemente im neuen Stream.
Hier kann also anhand des States (bereits hinzugefügte Elemente) eine Entscheidung getroffen werden.

Diese Optionen sind aber alle keine sauberen Lösungen und sind eher Hacks. Es fehlte eine generische intermediate Stream-Operation, die auch zustandsbehaftet sein kann.
Mit den neuen Stream-Gatherers ist dies möglich. So kann noch mehr imperativer Code im funktionalen, deklarativen Stil innerhalb Streams geschrieben werden.



_Hinweis: Dieser Blogbeitrag wurde ohne Nutzung von KI geschrieben._
