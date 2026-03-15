---
layout: page
title: "Java 25 New Features Cheat Sheet"
permalink: /cheatsheets/java25/
---
{% raw %}
_Java 25 (LTS, September 2025) — alle wichtigen Neuerungen auf einen Blick._

## Compact Source Files (JEP 512)

Einfachere Programme ohne Boilerplate — ideal für Einsteiger und Scripting.

```java
// Vorher: Java 21
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}

// Jetzt: Java 25
void main() {
    IO.println("Hello, World!");
}

// Instance main — kein static nötig
// Kein public class nötig
// Automatischer Import von java.base
// Neue IO-Klasse für Konsolen-I/O
```

### IO Klasse

```java
IO.println("Wie heisst du?");
String name = IO.readln();             // Zeile von stdin lesen
IO.println("Hallo, " + name + "!");

int number = Integer.parseInt(IO.readln());
IO.print("Ohne Zeilenumbruch");
```

## Flexible Constructor Bodies (JEP 513)

Code **vor** `super()` oder `this()` — Validierung und Transformation von Argumenten.

```java
// Vorher: Validierung erst NACH super()
public class PositiveRange extends Range {
    public PositiveRange(int start, int end) {
        super(start, end);
        if (start < 0) throw new IllegalArgumentException(); // zu spät!
    }
}

// Jetzt: Validierung VOR super()
public class PositiveRange extends Range {
    public PositiveRange(int start, int end) {
        if (start < 0 || end < 0) {
            throw new IllegalArgumentException("Values must be positive");
        }
        int adjustedEnd = Math.max(start, end);  // Argumente transformieren
        super(start, adjustedEnd);                // danach super()
    }
}

// Auch bei this() möglich
public class User {
    public User(String name) {
        Objects.requireNonNull(name);
        this(name, name.toLowerCase());   // this() nach Validierung
    }
    public User(String name, String slug) { ... }
}
```

**Einschränkung:** Vor `super()`/`this()` darf `this` nicht als Instanz verwendet werden (kein Zugriff auf Felder oder Methoden).

## Module Import Declarations (JEP 511)

Alle Packages eines Moduls mit einem Import.

```java
// Vorher: viele einzelne Imports
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

// Jetzt: ein Import für das gesamte Modul
import module java.base;      // importiert java.util.*, java.io.*, java.nio.* etc.
import module java.sql;       // importiert java.sql.*, javax.sql.*
import module java.net.http;  // importiert java.net.http.*

// Funktioniert auch ohne dass dein Code selbst in einem Modul ist!
```

## Primitive Types in Patterns (JEP 507, Preview)

Pattern Matching erweitert auf primitive Typen.

```java
// instanceof mit Primitiven
Object obj = 42;
if (obj instanceof int i) {
    System.out.println("Int: " + i);
}

// Switch mit primitiven Patterns
int statusCode = getStatusCode();
switch (statusCode) {
    case 200 -> handleOk();
    case 404 -> handleNotFound();
    case int code when code >= 500 -> handleServerError(code);
    case int code -> handleOther(code);
}

// Auch mit long, float, double, boolean
switch (temperature) {
    case double t when t < 0    -> "freezing";
    case double t when t < 20   -> "cold";
    case double t when t < 30   -> "comfortable";
    case double t               -> "hot";
}
```

## Stable Values (JEP 502, Preview)

Lazy-initialisierte Konstanten, die die JVM als `final` optimieren kann.

```java
import java.lang.StableValue;

// Einmal initialisiert, dann wie final behandelt
private final StableValue<Logger> logger = StableValue.of();

public Logger getLogger() {
    // Initialisierung beim ersten Aufruf, danach Konstante
    return logger.orElseSet(() -> Logger.getLogger(getClass().getName()));
}

// Für Listen
StableValue<List<Service>> services = StableValue.of();
services.orElseSet(() -> loadServicesFromConfig());
```

## Compact Object Headers (JEP 519)

Object Header von 128 Bit auf 64 Bit reduziert.

```
// Automatisch aktiv — kein Code nötig!
// Ergebnisse:
//   - ~22% weniger Heap-Verbrauch
//   - ~8% weniger CPU-Zeit
//   - Besonders wirkungsvoll bei vielen kleinen Objekten
```

## Ahead-of-Time (AOT) Verbesserungen

### AOT Cache (JEP 514)

```bash
# Vorher (Java 24): Zwei Schritte nötig
java -XX:AOTMode=record -XX:AOTConfiguration=app.aotconf -cp app.jar MyApp
java -XX:AOTMode=create -XX:AOTConfiguration=app.aotconf -XX:AOTCache=app.aot -cp app.jar

# Jetzt (Java 25): Ein Befehl
java -XX:AOTCache=app.aot -cp app.jar MyApp
# Erstellt und nutzt den Cache automatisch
```

### AOT Method Profiling (JEP 515)

```bash
# Training Run: Profildaten sammeln
java -XX:AOTCache=app.aot -cp app.jar MyApp

# Beim nächsten Start: Hot Methods sofort nativ kompiliert
# → Deutlich schnellere Aufwärmphase
```

## Key Derivation Function API (JEP 510)

```java
import javax.crypto.KDF;
import javax.crypto.SecretKey;

// HKDF für Key-Ableitung
KDF kdf = KDF.getInstance("HKDF-SHA256");

SecretKey derivedKey = kdf.deriveKey("AES",
    KDF.HKDFParameterSpec.ofExtract()
        .addIKM(inputKeyMaterial)
        .addSalt(salt)
        .thenExpand(info, 32)
        .build()
);
```

## JFR Verbesserungen

```bash
# Method Timing (JEP 520) — Laufzeitmessung per JFR
java -XX:StartFlightRecording:jdk.MethodTiming#enabled=true,\
  jdk.MethodTiming#filter='com.myapp.*' -jar app.jar

# CPU-Time Profiling (JEP 509, experimental)
java -XX:+UnlockExperimentalVMOptions \
  -XX:StartFlightRecording:jdk.CPUTimeSample#enabled=true -jar app.jar

# Cooperative Sampling (JEP 518) — stabiler Stack-Sampling
# Automatisch aktiv, keine Konfiguration nötig
```

## Generational Shenandoah GC (JEP 521)

```bash
# Shenandoah mit generationalem Modus (jetzt Production-ready)
java -XX:+UseShenandoahGC -XX:ShenandoahGCMode=generational -jar app.jar

# Kein -XX:+UnlockExperimentalVMOptions mehr nötig!
```

## Quick Reference: Was ist neu in Java 25?

```
Feature                           Status      JEP
────────────────────────────────── ─────────── ────
Compact Source Files               Final       512
Flexible Constructor Bodies        Final       513
Module Import Declarations         Final       511
Scoped Values                      Final       506
Compact Object Headers             Final       519
AOT Command-Line Ergonomics        Final       514
AOT Method Profiling               Final       515
Key Derivation Function API        Final       510
JFR Cooperative Sampling           Final       518
JFR Method Timing & Tracing        Final       520
Generational Shenandoah            Final       521
Primitive Types in Patterns        3rd Preview 507
Stable Values                      Preview     502
PEM Encodings                      Preview     470
Vector API                         Incubator   508
JFR CPU-Time Profiling             Experimental509
```
{% endraw %}
