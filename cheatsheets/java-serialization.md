---
layout: page
title: "Java Serialization Cheat Sheet"
permalink: /cheatsheets/java-serialization/
---

_Updated for Java 25 — includes Record Serialization & Deserialization Filters._

## Klassische Serialization

```java
// Klasse serialisierbar machen
public class User implements Serializable {
    @Serial
    private static final long serialVersionUID = 1L;

    private String name;
    private int age;
    private transient String password; // wird NICHT serialisiert
}
```

### Serialisieren & Deserialisieren

```java
// Schreiben
try (var oos = new ObjectOutputStream(new FileOutputStream("user.ser"))) {
    oos.writeObject(user);
}

// Lesen
try (var ois = new ObjectInputStream(new FileInputStream("user.ser"))) {
    User user = (User) ois.readObject();
}
```

### Serialization anpassen

```java
public class User implements Serializable {
    @Serial
    private static final long serialVersionUID = 1L;

    // Eigene Serialisierungslogik
    @Serial
    private void writeObject(ObjectOutputStream out) throws IOException {
        out.defaultWriteObject();
        out.writeUTF(encrypt(password));   // transient-Felder manuell schreiben
    }

    @Serial
    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
        in.defaultReadObject();
        this.password = decrypt(in.readUTF());
    }

    // Schutz vor gefälschten Streams
    @Serial
    private void readObjectNoData() throws ObjectStreamException {
        throw new InvalidObjectException("Stream data required");
    }

    // Ersatzobjekt bei Serialisierung zurückgeben
    @Serial
    private Object writeReplace() throws ObjectStreamException {
        return new UserProxy(this);
    }

    // Ersatzobjekt bei Deserialisierung zurückgeben
    @Serial
    private Object readResolve() throws ObjectStreamException {
        return INSTANCE; // z.B. für Singletons
    }
}
```

## @Serial Annotation (Java 14+)

Markiert Methoden und Felder, die zur Serialisierung gehören. Der Compiler warnt, wenn die Signatur nicht stimmt.

```java
@Serial private static final long serialVersionUID = 1L;
@Serial private static final ObjectStreamField[] serialPersistentFields = { ... };
@Serial private void writeObject(ObjectOutputStream out) { ... }
@Serial private void readObject(ObjectInputStream in) { ... }
@Serial private void readObjectNoData() { ... }
@Serial private Object writeReplace() { ... }
@Serial private Object readResolve() { ... }
```

## Record Serialization (Java 16+)

Records haben eine eigene, **sichere** Serialisierung — der kanonische Konstruktor wird immer verwendet.

```java
// Einfach Serializable implementieren — fertig!
public record User(String name, int age) implements Serializable {}

// Serialisierung nutzt automatisch:
//   - Component Accessors zum Schreiben (name(), age())
//   - Kanonischen Konstruktor zum Lesen (new User(name, age))
//
// IGNORIERT bei Records:
//   - writeObject, readObject, readObjectNoData
//   - writeExternal, readExternal
//   - serialPersistentFields
```

### Warum Records bevorzugen?

```java
// Klassisch: Deserialisierung umgeht den Konstruktor → Sicherheitsrisiko!
public class User implements Serializable {
    private final String name;
    public User(String name) {
        Objects.requireNonNull(name);  // wird bei Deserialisierung NICHT aufgerufen!
        this.name = name;
    }
}

// Record: Konstruktor wird IMMER aufgerufen → Validierung greift
public record User(String name) implements Serializable {
    public User {
        Objects.requireNonNull(name);  // wird auch bei Deserialisierung aufgerufen!
    }
}
```

## Externalizable

Volle Kontrolle über das Format — aber mehr Aufwand.

```java
public class User implements Externalizable {
    private String name;
    private int age;

    public User() {}   // public No-Arg-Konstruktor PFLICHT!

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeUTF(name);
        out.writeInt(age);
    }

    @Override
    public void readExternal(ObjectInput in) throws IOException {
        this.name = in.readUTF();
        this.age = in.readInt();
    }
}
```

## Deserialization Filters (Java 9+ / Java 17+)

Schutz vor Deserialization-Angriffen.

### Stream-spezifischer Filter

```java
var ois = new ObjectInputStream(inputStream);
ois.setObjectInputFilter(filterInfo -> {
    if (filterInfo.serialClass() != null) {
        // Nur bestimmte Klassen erlauben
        if (filterInfo.serialClass() == User.class) {
            return ObjectInputFilter.Status.ALLOWED;
        }
        return ObjectInputFilter.Status.REJECTED;
    }
    // Limits prüfen
    if (filterInfo.depth() > 5) return ObjectInputFilter.Status.REJECTED;
    if (filterInfo.references() > 100) return ObjectInputFilter.Status.REJECTED;
    return ObjectInputFilter.Status.UNDECIDED;
});
```

### Pattern-basierter Filter

```java
// Erlaubt nur bestimmte Klassen, begrenzt Tiefe und Referenzen
var filter = ObjectInputFilter.Config.createFilter(
    "com.myapp.model.*;!*;maxdepth=5;maxrefs=100"
);
ois.setObjectInputFilter(filter);

// Syntax:
//   com.myapp.*   → Paket erlauben
//   !*            → alles andere ablehnen
//   maxdepth=N    → max. Verschachtelungstiefe
//   maxrefs=N     → max. Anzahl Referenzen
//   maxbytes=N    → max. Bytes
//   maxarray=N    → max. Array-Länge
```

### JVM-weite Filter-Factory (Java 17+, JEP 415)

```java
// In main() oder per System Property setzen
ObjectInputFilter.Config.setSerialFilterFactory((current, next) -> {
    // Kombiniere globalen Filter mit stream-spezifischem
    var baseFilter = ObjectInputFilter.Config.createFilter(
        "com.myapp.**;!*"
    );
    return ObjectInputFilter.merge(next, baseFilter);
});

// Oder per JVM Property:
// -Djdk.serialFilter=com.myapp.**;!*
// -Djdk.serialFilterFactory=com.myapp.MyFilterFactory
```

## serialVersionUID

```java
// Explizit setzen → Kompatibilität kontrollieren
@Serial
private static final long serialVersionUID = 1L;

// Wenn nicht gesetzt: JVM berechnet automatisch basierend auf Klassenstruktur
// → Jede Änderung an der Klasse bricht Deserialisierung!

// Tipp: Immer explizit setzen und bei inkompatiblen Änderungen hochzählen
```

## Kompatible vs. inkompatible Änderungen

| Kompatibel | Inkompatibel |
|---|---|
| Felder hinzufügen | Felder entfernen/umbenennen |
| Zugriffsmodifier ändern | Feldtyp ändern |
| `transient` hinzufügen/entfernen | Klassenhierarchie ändern |
| Methoden ändern | `Serializable` entfernen |
| | `static` zu/von Feld ändern |

## Best Practices

```
1. Records bevorzugen — sichere Deserialisierung über Konstruktor
2. @Serial verwenden — Compiler fängt Signatur-Fehler ab
3. serialVersionUID immer setzen — explizite Versionskontrolle
4. Deserialization Filter nutzen — Schutz vor Angriffen
5. transient für sensible Daten — Passwörter, Tokens, etc.
6. readResolve für Singletons — verhindert Duplikate
7. Alternativen prüfen — JSON (Jackson/Gson) oft besser geeignet
8. Defensive Deserialisierung — Eingaben im readObject validieren
```
