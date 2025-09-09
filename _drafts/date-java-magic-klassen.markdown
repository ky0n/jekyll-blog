---
layout: post
title:  "Magie in Java - Klassen mit besonderer Funktionalität"
date:   2025-08-14 09:23:65 +0200
categories: jekyll update
tags: [java, stream, magics, jdk, java24, try-with, deutsch]
---
_diesmal ein Blogeintrag auf deutsch.._  
_schreibe gerne ein Kommentar unten im Kommentarbereich oder hinterlasse eine Reaktion_
# Magische Klassen und Interaces in Java

In der Programmiersprache Java gibt es Klassen, die besonders sind. Eine definierte Anzahl von Klassen im JDK sind nicht nur einfache Java-Klassen, die auch jeder User anlegen kann, sondern jene Klassen haben darüber hinaus eine spezielle Interaktion mit dem JDK. Diese Klassen können entweder in bestimmten Statements von Java verwendet werden oder haben eine feste Bedeutung, die mit Komponenten des Betriebssystems zusammenhängt oder in der schon bei dem Start des Java-Programms, zur Laufzeit, Variablen initialisiert werden. Ich unterteile die _magischen_ Klassen in 3 Kategorien. Es gibt Klassen die:
1. Eine eigene Rolle in der Syntax von Java haben
    - d. h. sie können mit speziellen Syntax-Features on Java verwendet werden, wie z.b. der for-each-Schleife oder dem try-with-resources-Statement
2. Während der Laufzeit statisch-initialisierte Variablen haben
    - Hier ist inbsesondere die `System`-Klasse zu erwähnen,
3. Symbolisch für _mehr_ stehen, also z.b. für eine Funktion des Betriebssystems.
    - z.B. die `Thread`-Klasse startet einen eigenen Thread während der Laufzeit, der vom Betriebssystem gemanaged wird, solange es ein Platform-Thread ist (kein virtual-thread)

## Kategorie 1: Klassen mit eigener Bedeutung in der Syntax von Java
Interfaces, die in speziellen Syntax-Konstrukten in Java verwendbar sind, ermöglichen eine "Growability" der Sprache. Bibliotheken-Maintainer und auch im eigenen Projekt können Implementierungen der Interfaces schreiben, die dann in den speziellen Java-Statements verwendet werden. Der Chief-Java-Architect Brian Götz hat darüber zufällig vor kurzem ein Talk gehalten beim JVM Language Summit ([Youtube-Link](https://www.youtube.com/watch?v=Gz7Or9C0TpM)). 

### Iterable-Interface für for-each-Schleife
Klasse, die das Iterable-Interface implementieren können in einer for-each-Schleife verwendet werden. 
Standardmäßig fällt hierunter alle Collection-Klassen im JDK, da das `Collection`-Interface `Iterable` extended. Also die Implementierungen von `List`, `Set`, aber nicht `Map`. Eine Map kann nach Umwandlung in ein `Set<Map.Entry<K, V>>` umgewandelt werden durch Aufruf von `Map::entrySet` und somit in einer for-each-Schleife verwendet werden. Als die for-each-Schleife mit Java 7 eingeführt wurde, wurde 
```java
// Beispiel Iteration eine Map mittels for-each-Schleife
var beispielMap = Map.of(1, "Wert 1", 2, "Wert 2", 3, "Wert 3");
for (var entry : beispielMap.entrySet())
    System.out.println(entry);
```


### Statement try-with-resources und AutoCloseable-Interface
Das try-with-resources Statement 
<!-- 
- try-with, autoclose implementierung
- iterable implementieren -> nutzung in for-each schleife
- Exception (checked/unchecked exceptions)
- Object-Klasse
- Literale (String (non-primitive Literal), byte, long, double, float, short, autoboxing)
- main methode (aber logisch..)

- thread klasse
- System klasse

-->
### Checked und unchecked Exception
Es gibt in Java Exceptions, die zwingend behandelt werden müssen. D.h. dass diese Exceptions entweder gefangen werden müssen oder per throws-clause in der Methoden-Kopfzeile weitergeworfen werden müssen.

Demgegenüber gibt es _Unchecked Exceptions_, welche diese Bedingung nicht haben und nicht behandelt werden müssen.
Dazu gehören Runtime-Exceptions und Exceptions, die von `RuntimeException` erben. Zudem gehören dazu `Error`-Klassen und Exceptions, die von `Error` erben, beispielsweise `OutOfMemoryError` oder `StackOverflowError`. 

Die Magie liegt hierbei daran, dass Abhängig vom Supertyp einer Exceptions eine Behandlung dieser zwingend notwendig wird und ohne diese die Kompilierung fehlschlägt. Ausserdem ist die Throwable-Klasse, die Klasse, von der alle Exception erben, auch _magisch_, da nur sie das Werfen mittels `throws` überhaupt ermöglicht.


### Object Klasse
Die `Object` Klasse ist daher _magisch_, da alle Klassen in Java von ihr erben und dies implizit. Es ist möglich dies explizit in die Kopfzeile einer Klasse (ohne andere Eltern-Klassen) zu schreiben, aber dies ist überflüssig und wird bei IntelliJ auch entsprechend als "unnecessary" und mit grauer Schrift markiert.
```java
// Bei dieser minimalen Klassen-Deklaration wird "extends Object" als unnecessary markiert, da Klasse A auch implizit von Object erbt.
class A extends Object {}
```
Aus diesem Grund kann auch jede Klasse in Java public/protected Methoden von `Object`, wie `hashcode`, `equals`, `clone` implementieren.


## Kategorie 2: Klassen, deren Variablen (teilweise) beim Programmstart initialisiert werden
Zu dieser Kategorie zählen Klassen, die statische Variablen haben, die beim Start des Programmes initialisiert werden. Dazu zählt beispielsweise die Klasse 
`System`. Durch `System.in` und `System.out` wird der Zugang zum Input/Output im Terminal gewährt. 


## Kategorie 3: Klassen, die fest mit Betriebssystem-Funktionalität verknüpft sind und besondere Privilegien in der JRE haben

Zu dieser Kategorie zählt zu erst die `Thread`-Klasse. Mit ihr werden, zumindest mit nicht-virtuellen Thread, d.h. Plattform-Threads, auch jeweils ein nativer Thread des Betriebsystems verbunden. Die JVM sorgt lediglich für die Verwaltung des Threads auf Java-Ebene und die Java `Thread`-Klasse ist ein Wrapper um den nativen Betriebssystem-Thread. Auch User könnten theoretisch über native Methode mit JNI oder Panama Threads erstellen, aber auch dieser Code hat kein Zugriff auf die nativen VM-Hooks, welche nur im JDK existieren. 

Über die statisch native Methode `Thread.currentThread()` wird der Thread ermittelt, der diese Methode aufruft und als `Thread`-Objekt zurückgegeben. Diese Methode ist tief verankert im JVM und so nicht nachzubauen in einer durch User erstellt Klasse.