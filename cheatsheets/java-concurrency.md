---
layout: page
title: "Java Concurrency Cheat Sheet"
permalink: /cheatsheets/java-concurrency/
---

_Updated for Java 25 — includes Virtual Threads, Scoped Values (JEP 506) & Structured Concurrency (JEP 505, Preview)._

## Virtual Threads (Java 21+, JEP 444)

Leichtgewichtige Threads, die von der JVM verwaltet werden — Millionen gleichzeitig möglich.

```java
// Thread starten
Thread.ofVirtual().start(() -> {
    System.out.println("Hello from virtual thread!");
});

// Mit Name
Thread.ofVirtual().name("worker-", 0).start(task);

// Mit Executor (empfohlener Weg)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 10_000).forEach(i ->
        executor.submit(() -> fetchUrl("https://example.com/" + i))
    );
}
// executor.close() wartet auf alle Tasks — try-with-resources!

// Prüfen ob Virtual Thread
Thread.currentThread().isVirtual()  // true/false

// Factory für Virtual Threads
ThreadFactory factory = Thread.ofVirtual().name("pool-", 0).factory();
```

### Wann Virtual Threads verwenden?

```
Gut geeignet:                      NICHT geeignet:
  I/O-bound Tasks (HTTP, DB)         CPU-intensive Berechnungen
  Viele gleichzeitige Verbindungen   Wenige lang laufende Tasks
  Request-per-Thread Server          Synchronized-heavy Code (vor Java 24)
```

### Pinning vermieden (Java 24+, JEP 491)

```java
// Vor Java 24: synchronized blockierte den Carrier-Thread
// Ab Java 24: Virtual Threads können auch in synchronized unmounten
synchronized (lock) {
    // I/O-Operation — ab Java 24 kein Pinning mehr!
    var data = httpClient.send(request, bodyHandler);
}
```

## Scoped Values (Java 25, JEP 506)

Sichere, performante Alternative zu `ThreadLocal` — besonders für Virtual Threads.

```java
// Deklarieren (immer static final)
private static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();

// Binden und ausführen
ScopedValue.where(CURRENT_USER, user).run(() -> {
    handleRequest();    // und alle aufgerufenen Methoden haben Zugriff
});

// Wert lesen (in handleRequest oder tieferem Aufruf)
User user = CURRENT_USER.get();           // wirft NoSuchElementException wenn ungebunden
User user = CURRENT_USER.orElse(null);    // default wenn ungebunden
boolean bound = CURRENT_USER.isBound();   // prüfen

// Mit Rückgabewert
String result = ScopedValue.where(CURRENT_USER, user).call(() -> {
    return processRequest();
});

// Mehrere Werte binden
ScopedValue.where(CURRENT_USER, user)
           .where(TRACE_ID, traceId)
           .run(() -> handleRequest());

// Rebinding in verschachteltem Scope
ScopedValue.where(CURRENT_USER, user).run(() -> {
    // CURRENT_USER.get() → user
    ScopedValue.where(CURRENT_USER, adminUser).run(() -> {
        // CURRENT_USER.get() → adminUser
    });
    // CURRENT_USER.get() → user (wiederhergestellt)
});
```

### ScopedValue vs. ThreadLocal

```
ScopedValue                         ThreadLocal
  Immutable innerhalb Scope           Jederzeit mutierbar
  Automatisches Cleanup               Manuelles remove() nötig (Memory Leak!)
  Vererbung in StructuredTaskScope    InheritableThreadLocal (kopiert Daten)
  Sehr performant mit VTs             Overhead bei vielen VTs
  Klar begrenzter Scope               Unbegrenzter Lebenszyklus
```

## Structured Concurrency (Java 25, JEP 505 — Preview)

Macht nebenläufige Tasks strukturiert — wie verschachtelte Codeblöcke.

```java
// Preview-Feature: --enable-preview nötig

// Einfaches Beispiel: zwei Tasks parallel ausführen
try (var scope = StructuredTaskScope.open()) {
    Subtask<String> user  = scope.fork(() -> fetchUser(id));
    Subtask<String> order = scope.fork(() -> fetchOrder(id));

    scope.join();  // warte auf alle

    return new Response(user.get(), order.get());
}
// scope.close() cancelled alle noch laufenden Tasks automatisch!

// Mit Joiner: Erstes Ergebnis gewinnt (Race-Pattern)
try (var scope = StructuredTaskScope.open(Joiner.anySuccessfulResultOrThrow())) {
    scope.fork(() -> fetchFromMirror1(url));
    scope.fork(() -> fetchFromMirror2(url));
    scope.fork(() -> fetchFromMirror3(url));

    var result = scope.join();  // gibt erstes erfolgreiches Ergebnis zurück
    // alle anderen Tasks werden automatisch abgebrochen
}

// Mit Joiner: Alle müssen erfolgreich sein
try (var scope = StructuredTaskScope.open(Joiner.allSuccessfulOrThrow())) {
    scope.fork(() -> validatePayment(order));
    scope.fork(() -> checkInventory(order));
    scope.fork(() -> verifyAddress(order));

    scope.join();  // wirft bei erstem Fehler, bricht andere ab
}
```

### Subtask States

```java
Subtask<String> task = scope.fork(() -> compute());
scope.join();

task.state()  // UNAVAILABLE → RUNNING → SUCCESS | FAILED

task.get()    // Ergebnis (nur bei SUCCESS, sonst IllegalStateException)
task.exception()  // Exception (nur bei FAILED)
```

## Klassische Thread-Synchronisation

### synchronized & Locks

```java
// synchronized Block
synchronized (lock) {
    sharedState.update();
}

// ReentrantLock — flexibler als synchronized
var lock = new ReentrantLock();
lock.lock();
try {
    sharedState.update();
} finally {
    lock.unlock();
}

// Trylock mit Timeout
if (lock.tryLock(1, TimeUnit.SECONDS)) {
    try { /* ... */ } finally { lock.unlock(); }
}

// ReadWriteLock — mehrere Reader, ein Writer
var rwLock = new ReentrantReadWriteLock();
rwLock.readLock().lock();    // mehrere gleichzeitig
rwLock.writeLock().lock();   // exklusiv

// StampedLock — optimistisches Lesen (Java 8+)
var sl = new StampedLock();
long stamp = sl.tryOptimisticRead();
int value = sharedValue;
if (!sl.validate(stamp)) {
    stamp = sl.readLock();
    try { value = sharedValue; } finally { sl.unlockRead(stamp); }
}
```

### Atomic Classes

```java
AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet();                      // ++count
count.getAndIncrement();                      // count++
count.compareAndSet(expected, newValue);       // CAS
count.updateAndGet(x -> x * 2);              // atomare Transformation
count.accumulateAndGet(5, Integer::sum);      // atomar akkumulieren

// Für High-Contention: LongAdder/LongAccumulator (Java 8+)
LongAdder adder = new LongAdder();
adder.increment();
adder.sum();   // Ergebnis lesen

// AtomicReference für Objekte
AtomicReference<User> ref = new AtomicReference<>(user);
ref.compareAndSet(oldUser, newUser);
```

## Concurrent Collections

```java
// Map
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.compute("key", (k, v) -> v == null ? 1 : v + 1);      // atomar
map.merge("key", 1, Integer::sum);                          // atomar
map.putIfAbsent("key", defaultValue);
map.computeIfAbsent("key", k -> expensiveCompute(k));

// Queue
ConcurrentLinkedQueue<Task> queue = new ConcurrentLinkedQueue<>();
BlockingQueue<Task> bq = new LinkedBlockingQueue<>(capacity);
bq.put(task);            // blockiert wenn voll
bq.take();               // blockiert wenn leer
bq.offer(task, 1, TimeUnit.SECONDS);  // mit Timeout

// List (Copy-on-Write — gut für viel Lesen, wenig Schreiben)
CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();

// Set
ConcurrentHashMap.newKeySet()
CopyOnWriteArraySet<String> set = new CopyOnWriteArraySet<>();
```

## CompletableFuture

```java
// Async starten
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> fetchData());

// Verketten
cf.thenApply(String::toUpperCase)          // transformieren
  .thenAccept(System.out::println)         // konsumieren
  .thenRun(() -> log("done"));             // danach ausführen

// Async Verkettung
cf.thenApplyAsync(this::process)           // in anderem Thread
  .thenComposeAsync(this::fetchMore);      // flatMap-Äquivalent

// Mehrere kombinieren
CompletableFuture.allOf(cf1, cf2, cf3).join();    // warte auf alle
CompletableFuture.anyOf(cf1, cf2, cf3).join();    // warte auf ersten

// Fehlerbehandlung
cf.exceptionally(ex -> "fallback")
  .handle((result, ex) -> ex != null ? "error" : result)
  .whenComplete((result, ex) -> log(result, ex));

// Timeout (Java 9+)
cf.orTimeout(5, TimeUnit.SECONDS)
  .completeOnTimeout("default", 5, TimeUnit.SECONDS);
```

## Executors & Thread Pools

```java
// Virtual Threads (Java 21+ — bevorzugt für I/O!)
Executors.newVirtualThreadPerTaskExecutor()

// Klassische Thread Pools
Executors.newFixedThreadPool(nThreads)         // feste Größe
Executors.newCachedThreadPool()                // dynamisch
Executors.newSingleThreadExecutor()            // ein Thread
Executors.newScheduledThreadPool(coreSize)     // zeitgesteuert

// Scheduled Execution
var scheduler = Executors.newScheduledThreadPool(1);
scheduler.schedule(task, 5, TimeUnit.SECONDS);           // einmal nach Delay
scheduler.scheduleAtFixedRate(task, 0, 1, TimeUnit.SECONDS);  // wiederholt
scheduler.scheduleWithFixedDelay(task, 0, 1, TimeUnit.SECONDS);

// Immer shutdown aufrufen!
executor.shutdown();
if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
    executor.shutdownNow();
}
```

## Countdown & Barriers

```java
// CountDownLatch — einmalig, Threads warten auf Countdown
CountDownLatch latch = new CountDownLatch(3);
latch.countDown();        // in Worker-Threads
latch.await();            // wartet bis count == 0

// CyclicBarrier — wiederverwendbar, Threads warten aufeinander
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("Phase done"));
barrier.await();          // jeder Thread wartet bis alle da sind

// Semaphore — begrenzte Anzahl gleichzeitiger Zugriffe
Semaphore semaphore = new Semaphore(5);
semaphore.acquire();
try { accessResource(); } finally { semaphore.release(); }

// Phaser (Java 7+) — flexibler als CyclicBarrier
Phaser phaser = new Phaser(1);
phaser.register();        // Teilnehmer hinzufügen
phaser.arriveAndAwaitAdvance();
phaser.arriveAndDeregister();
```

## Quick-Reference: Was wofür?

```
Aufgabe                          Lösung
──────────────────────────────── ────────────────────────────────
Viele I/O Tasks parallel         Virtual Threads + Executor
Implizite Kontextdaten           ScopedValue (statt ThreadLocal)
Fan-out/Fan-in parallel          StructuredTaskScope (Preview)
Einfacher Shared Counter         AtomicInteger / LongAdder
Thread-sichere Map               ConcurrentHashMap
Producer-Consumer                BlockingQueue
Async Pipeline                   CompletableFuture
Exklusiver Zugriff               ReentrantLock / synchronized
Viele Reader, wenig Writer       ReadWriteLock / StampedLock
N Tasks abwarten                 CountDownLatch
Threads synchronisieren          CyclicBarrier
Zugriff begrenzen                Semaphore
```
