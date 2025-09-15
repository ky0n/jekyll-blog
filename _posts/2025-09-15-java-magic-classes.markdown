---
layout: post
title:  "Magie in Java - Klassen mit besonderer Funktionalität"
date:   2025-09-15 18:06:20 +0200
categories: jekyll update
tags: [java, stream, magics, jdk, java24, try-with, deutsch, iterable, java 25, classloader]
---
# "Magische" Klassen und Interfaces in Java

In der Programmiersprache Java gibt es Klassen, die besonders sind. Eine definierte Anzahl von Klassen im JDK sind nicht nur einfache Java-Klassen, die auch jeder User anlegen kann, sondern jene Klassen haben darüber hinaus eine spezielle Interaktion mit dem JDK. Diese Klassen können entweder in bestimmten Statements von Java verwendet werden und sind dabei fest in der Syntax verbunden oder haben eine feste Bedeutung, die mit Komponenten des Betriebssystems zusammenhängt oder es werden bereits beim Start des Java-Programms, zur Laufzeit, Variablen initialisiert. Ich unterteile die _magischen_ Klassen in 3 Kategorien. Es gibt Klassen die:
1. Eine eigene Rolle in der Syntax von Java haben
    - d. h. sie können mit speziellen Syntax-Features on Java verwendet werden, wie z.b. der for-each-Schleife oder dem try-with-resources-Statement
2. Während der Laufzeit statisch-initialisierte Variablen haben
    - Hier ist insbesondere die `System`-Klasse zu erwähnen,
3. Symbolisch für _mehr_ stehen, also z.b. für eine Funktion des Betriebssystems.
    - z.B. die `Thread`-Klasse startet einen eigenen Thread während der Laufzeit, der vom Betriebssystem gemanaged wird, solange es ein Platform-Thread ist (kein virtual-thread)

## Kategorie 1: Klassen mit eigener Bedeutung in der Syntax von Java
Interfaces, die in speziellen Syntax-Konstrukten in Java verwendbar sind, ermöglichen eine "Growability" der Sprache. Bibliotheken-Maintainer und auch im eigenen Projekt können Implementierungen der Interfaces schreiben, die dann in den speziellen Java-Statements verwendet werden. Der Chief-Java-Architect Brian Götz hat darüber zufällig vor kurzem ein Talk gehalten beim JVM Language Summit 2025 ([Youtube-Link](https://www.youtube.com/watch?v=Gz7Or9C0TpM){:target="_blank"}). 

### Iterable-Interface für for-each-Schleife
Klassen, die das Iterable-Interface implementieren können in einer for-each-Schleife verwendet werden. Auch User können das Iterable-Interface implementieren und dann mit selbstgeschriebenen Klassen dieses Syntax-Feature verwenden.
Häufig genutzt werden in diesem Statement alle Collection-Klassen im JDK, da das `Collection`-Interface `Iterable` extended. Also die Implementierungen von `List`, `Set`, aber nicht `Map`. Eine Map kann nach Umwandlung in ein `Set<Map.Entry<K, V>>` umgewandelt werden durch Aufruf von `Map::entrySet` und somit in einer for-each-Schleife verwendet werden.
```java
// Beispiel Iteration eine Map mittels for-each-Schleife
var beispielMap = Map.of(1, "Wert 1", 2, "Wert 2", 3, "Wert 3");
for (var entry : beispielMap.entrySet())
    System.out.println(entry);
```


### Statement try-with-resources und AutoCloseable-Interface
Das try-with-resources Statement kann nur mit Klassen verwendet werden, die das `java.lang.AutoCloseable`-Interface implementieren. Das AutoCloseable-Interface ist ein Functional-Interface, mit der abstrakten Methode `void close() throws Exception`.
```java
package java.lang;
public interface AutoCloseable {
    void close() throws Exception;
} 
```
Die geworfene checked Exception kann unterdrückt werden, indem bei der Implementierung der close Methode keine CheckedException geworfen wird und dadurch auch keine throws-Klausel erforderlich ist. Wenn keine throws-Klausel in der Implementierung verwendet wird, muss das try-with-resources statement auch kein `catch`-Zweig haben.

Der Name AutoCloseable bzw. close ist dabei etwas unglücklich, da das Interface beispielsweise auch sehr gut für Locks verwendet werden kann und ein Lock am Ende der Operation freigegeben wird und nicht geschlossen wird.  
  
Das Try-with-resources Statement wird in verschiedensten Szenarien genutzt. Die Resourcen werden in jedem Fall freigegeben, d.h. am Ende des Try-Blocks wird die `close`-Implementierung aufgerufen des Objektes in den Klammern des Trys.
```java
try (Connection conn = DriverManager.getConnection(url, user, pw);
    Statement stmt = conn.createStatement;
    ResultSet rs = stmt.executeQuery("SELECT id, name FROM users")) {
    /// ...
} catch (SQLException e) {
    // ...
}
```
Wenn es mehrere Resourcen gibt, werden die close-Methoden in umgekehrter Reihenfolge aufgerufen. Das Try-with-resources-Statement kann auch genutzt werden mit selbstgeschriebenen Implementierungen des AutoCloseable-Interfaces. Es ist quasi hardgecoded in den Java-Compiler, dass Try-with-resource-Statement nur mit Implementierungen des AutoCloseable-Interfaces möglich ist.

```java
try (AutoCloseable _ = () -> System.out.println("Ende des Blocks")) {
    System.out.println("Innerhalb des Blocks");
}
```
Diese weniger sinnvolle anonyme Implementierung des AutoCloseable-Intefaces ermöglicht es schon das try-with-resources-Statement zu verwenden. Zuerst wird "Innerhalb des Blocks" in die Konsole geschrieben, danach "Ende des Blocks".


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
Dazu gehören Runtime-Exceptions und Exceptions, die von `RuntimeException` erben. Zudem gehören dazu `Error`-Klassen und Exceptions, die von `Error` erben, beispielsweise `OutOfMemoryError` oder `StackOverflowError`. Diese Fehler sind meist sehr kritisch und können nicht mehr behandelt werden.

Die Magie liegt hierbei daran, dass Abhängig vom Supertyp einer Exceptions eine Behandlung dieser zwingend notwendig wird und ohne diese die Kompilierung fehlschlägt. Ausserdem ist die Throwable-Klasse, die Klasse, von der alle Exception erben, auch _magisch_, da nur sie das Werfen mittels `throws` überhaupt ermöglicht.


### Object Klasse
Die `Object` Klasse ist daher _magisch_, da alle Klassen in Java von ihr erben und dies implizit. Es ist möglich dies explizit in die Kopfzeile einer Klasse (ohne andere Eltern-Klassen) zu schreiben, aber dies ist überflüssig und wird bei IntelliJ auch entsprechend als Warning markiert mit dem Hinweis "explicitly extends `java.lang.Object`".
```java
// explizites (unnötiges) extends Object
class A extends Object {}
```
Aus diesem Grund kann auch jede Klasse in Java public/protected Methoden von `Object`, wie `hashcode`, `equals`, `clone` implementieren.


## Kategorie 2: Klassen, deren Variablen (teilweise) beim Programmstart initialisiert werden und Objekte, die von der JVM erzeugt werden
Zu dieser Kategorie zählen Klassen, die statische Variablen haben, die beim Start des Programmes initialisiert werden, bzw. beim Laden des Klasse durch die JVM. Dazu zählt beispielsweise die Klasse 
`System`. Durch `System.in`,`System.out` und `System.err` wird der Zugang zum Input/Output im Terminal gewährt. 

Zu jeder geladene Java-Klasse wird ein `Class`-Objekt mit Metainformationen zu dieser Klasse erzeugt. Dies ist aufrufbar über `"Klasse".class`. Hier ein Beispiel: 
`Class<String> s = String.class;` Darüber können während der Laufzeit des Programmes beispielsweise Annotationen einer Klasse ausgelesen werden.

Die Bootstrap-Classloader, Plattform-Classloader und System-Classloader sind auch magische Klassen. Sie implementieren jeweils die abstrakte Klasse java.lang.ClassLoader.
Der Plattform-Classloader lädt die die Standard-Java-Library. Er hat privilegierte Rechte um alle _Plattform-Klassen_ lesen zu können.

## Kategorie 3: Klassen, die fest mit Betriebssystem-Funktionalität verknüpft sind und besondere Privilegien in der JRE haben

Zu dieser Kategorie zählt zuerst die `Thread`-Klasse. Mit ihr werden, zumindest mit nicht-virtuellen Thread, d.h. Plattform-Threads, auch jeweils ein nativer Thread des Betriebsystems verbunden. Die JVM sorgt lediglich für die Verwaltung des Threads auf Java-Ebene und die Java `Thread`-Klasse ist ein Wrapper um den nativen Betriebssystem-Thread. Auch User könnten theoretisch über native Methode mit JNI oder Panama Threads erstellen, aber auch dieser Code hat kein Zugriff auf die nativen VM-Hooks, welche nur im JDK existieren. 

Über die statisch native Methode `Thread.currentThread()` wird der Thread ermittelt, der diese Methode aufruft und als `Thread`-Objekt zurückgegeben. Diese Methode ist tief verankert im JVM und so nicht nachzubauen in einer durch User erstellt Klasse.

In diese Kategorie gehört auch die Runtime Klasse, wobei diese ebenfalls zu Kategorie 2 gehört. Die `java.lang.Runtime`-Klasse gibt über die statische Methode `Runtime.getRuntime()` ein `Runtime`-Objekt zurück, dass mit dem laufenden Java-Programm zusammenhängt. Hierdurch kann beispielsweise mittels `gc()` der Garbage-Collector der JVM angestossen werden. Allerdings ist diese Methode keine Garantie, dass bestimmte Objekte durch den Garbage-Collector recycled wurden. Weitere Methoden in dieser Klasse sind `maxMemory` sowie weitere Methoden im Zusammenhang mit dem Speicher der JVM. Diese Methoden sind `native` und daher direkt in C++ oder C geschrieben in der JVM.

_diesmal ein Blogeintrag auf deutsch._  
_schreibe gerne ein Kommentar unten im Kommentarbereich oder hinterlasse eine Reaktion_