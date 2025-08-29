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
    - d. h. sie können mit speziellen Syntax-Features on Java verwendet werden, wie z.b. der for-each-Schleife oder dem try-with-resource-Statement
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


### try-with-resource Statement und AutoCloseable-Interface

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

## Kategorie 2: Klassen, deren Variablen (teilweise) beim Programmstart initialisiert werden


## Kategorie 3: Klassen, die fest mit Betriebssystem-Funktionalität verknüpft sind und besondere Privilegien in der JRE haben

Zu dieser Kategorie zählt zu erst die `Thread`-Klasse. Mit ihr werden, zumindest mit nicht-virtuellen Thread, d.h. Plattform-Threads, auch jeweils ein nativer Thread des Betriebsystems verbunden. Die JVM sorgt lediglich für die Vewaltung des Threads auf Java-Ebene und die Java `Thread`-Klasse ist ein Wrapper um den nativen Betriebssystem-Thread.   