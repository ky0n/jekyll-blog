---
layout: page
title: "Jakarta EE 11 Cheat Sheet"
permalink: /cheatsheets/jakarta-ee/
---
{% raw %}
_Jakarta EE 11 (Juni 2025) — CDI, REST, JPA, Data, Security, Concurrency & mehr._

## Was ist neu in Jakarta EE 11?

```
Neu:        Jakarta Data 1.0 (Repository Pattern)
Minimum:    Java SE 17, empfohlen Java SE 21 (Virtual Threads, Records)
Entfernt:   JAXB, JAX-WS, SOAP, Managed Beans, CORBA
Deprecated: @Context in REST (→ @Inject), java.util.Date in JPA (→ java.time)

16 aktualisierte Specs:
  Persistence 3.2, REST 4.0, CDI 4.1, Concurrency 3.1,
  Security 4.0, Servlet 6.1, Faces 4.1, Validation 3.1,
  JSON-B 3.0, JSON-P 2.1, Expression Language 6.0
```

## CDI (Contexts & Dependency Injection) 4.1

### Scopes

```java
@ApplicationScoped    // Ein Exemplar pro Applikation (Singleton-artig)
@RequestScoped        // Ein Exemplar pro HTTP-Request
@SessionScoped        // Ein Exemplar pro HTTP-Session
@ConversationScoped   // Über mehrere Requests hinweg
@Dependent            // Default: neue Instanz pro Injection Point
```

### Injection

```java
// Field Injection
@Inject
private UserService userService;

// Constructor Injection (empfohlen)
@Inject
public OrderController(OrderService orderService, UserService userService) {
    this.orderService = orderService;
    this.userService = userService;
}

// Qualifier für Disambiguierung
@Inject @Premium
private PricingService pricing;

// Named Bean (z.B. für Faces/EL)
@Named("orderBean")
@RequestScoped
public class OrderBean { ... }
```

### Producer

```java
@Produces
@ApplicationScoped
public DataSource createDataSource() {
    // Factory für Nicht-CDI-Objekte
    return dataSourceFromConfig();
}

@Produces @RequestScoped @Named("currentUser")
public User getCurrentUser() {
    return securityContext.getCallerPrincipal();
}
```

### Interceptors

```java
// 1. Binding Annotation
@InterceptorBinding
@Retention(RUNTIME) @Target({TYPE, METHOD})
public @interface Logged {}

// 2. Interceptor
@Logged @Interceptor @Priority(Interceptor.Priority.APPLICATION)
public class LoggingInterceptor {
    @AroundInvoke
    public Object log(InvocationContext ctx) throws Exception {
        System.out.println("→ " + ctx.getMethod().getName());
        return ctx.proceed();
    }
}

// 3. Anwenden
@Logged
public class OrderService { ... }
```

### Events

```java
// Feuern
@Inject Event<Order> orderEvent;
orderEvent.fire(new Order(...));            // synchron
orderEvent.fireAsync(new Order(...));       // asynchron

// Beobachten
void onOrder(@Observes Order order) { ... }
void onOrderAsync(@ObservesAsync Order order) { ... }
void afterCommit(@Observes(during = AFTER_SUCCESS) Order o) { ... }
```

## Jakarta REST 4.0

### Resource-Klasse

```java
@Path("/products")
@ApplicationScoped
public class ProductResource {

    @Inject ProductService service;        // NEU: @Inject statt @Context

    @GET
    @Produces(MediaType.APPLICATION_JSON)
    public List<Product> getAll() {
        return service.findAll();
    }

    @GET @Path("/{id}")
    public Product getById(@PathParam("id") long id) {
        return service.findById(id)
            .orElseThrow(NotFoundException::new);
    }

    @POST
    @Consumes(MediaType.APPLICATION_JSON)
    public Response create(@Valid Product p) {
        service.save(p);
        return Response.created(URI.create("/products/" + p.getId()))
                       .entity(p).build();
    }

    @PUT @Path("/{id}")
    @Consumes(MediaType.APPLICATION_JSON)
    public Product update(@PathParam("id") long id, Product p) {
        return service.update(id, p);
    }

    @DELETE @Path("/{id}")
    public void delete(@PathParam("id") long id) {
        service.delete(id);
    }
}
```

### Parameter-Annotationen

```java
@PathParam("id")              // /items/{id}
@QueryParam("page")           // ?page=2
@HeaderParam("Authorization")
@CookieParam("session")
@FormParam("name")            // Form POST
@BeanParam                    // Parameter-Objekt
@DefaultValue("1")            // Default-Wert
```

### Filter & Exception Mapper

```java
// Request Filter
@Provider
public class AuthFilter implements ContainerRequestFilter {
    @Override
    public void filter(ContainerRequestContext ctx) {
        if (!isAuthorized(ctx)) {
            ctx.abortWith(Response.status(401).build());
        }
    }
}

// Exception Mapper
@Provider
public class NotFoundMapper implements ExceptionMapper<NotFoundException> {
    @Override
    public Response toResponse(NotFoundException e) {
        return Response.status(404)
            .entity(Map.of("error", e.getMessage())).build();
    }
}
```

### SSE (Server-Sent Events)

```java
@GET @Path("/events")
@Produces(MediaType.SERVER_SENT_EVENTS)
public void stream(@Inject SseEventSink sink, @Inject Sse sse) {
    executor.submit(() -> {
        sink.send(sse.newEvent("update", "data"));
        sink.close();
    });
}
```

### Client API

```java
Client client = ClientBuilder.newClient();
List<Product> products = client
    .target("https://api.example.com/products")
    .queryParam("category", "books")
    .request(MediaType.APPLICATION_JSON)
    .get(new GenericType<List<Product>>() {});
client.close();
```

## Jakarta Persistence 3.2

### EntityManager

```java
@PersistenceContext
EntityManager em;

em.persist(entity);                  // INSERT
em.find(Product.class, id);          // SELECT by PK
em.merge(entity);                    // UPDATE (detached → managed)
em.remove(entity);                   // DELETE
em.flush();                          // SQL sofort ausführen
em.detach(entity);                   // aus Persistence Context lösen
em.refresh(entity);                  // aus DB neu laden
```

### JPQL

```java
// Basis
em.createQuery("SELECT p FROM Product p WHERE p.price > :price", Product.class)
  .setParameter("price", 100.0)
  .getResultList();

// Joins
"SELECT o FROM Order o JOIN o.items i WHERE i.product.name = :name"

// Aggregation
"SELECT p.category, COUNT(p), AVG(p.price) FROM Product p GROUP BY p.category"

// NEU in 3.2: UNION, INTERSECT, EXCEPT
"SELECT p FROM Product p WHERE p.active = true
 UNION
 SELECT p FROM Product p WHERE p.featured = true"

// NEU in 3.2: id() und version() Funktionen
"SELECT id(p), version(p) FROM Product p"

// NEU in 3.2: left(), right(), replace()
"SELECT left(p.name, 3) FROM Product p"
"SELECT replace(p.description, 'old', 'new') FROM Product p"

// NEU in 3.2: || String-Konkatenation
"SELECT p.firstName || ' ' || p.lastName FROM Person p"
```

### Relationships

```java
// One-to-Many / Many-to-One
@OneToMany(mappedBy = "order", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
List<OrderItem> items;

@ManyToOne @JoinColumn(name = "order_id")
Order order;

// Many-to-Many
@ManyToMany
@JoinTable(name = "student_course",
    joinColumns = @JoinColumn(name = "student_id"),
    inverseJoinColumns = @JoinColumn(name = "course_id"))
Set<Course> courses;

// One-to-One
@OneToOne(mappedBy = "employee")
Address address;
```

### Lifecycle Callbacks

```java
@PrePersist    void beforeInsert() { createdAt = Instant.now(); }
@PostPersist   void afterInsert() { ... }
@PreUpdate     void beforeUpdate() { updatedAt = Instant.now(); }
@PostUpdate    void afterUpdate() { ... }
@PreRemove     void beforeDelete() { ... }
@PostLoad      void afterLoad() { ... }
```

### Neu in Persistence 3.2

```java
// Programmatische Konfiguration (ohne persistence.xml)
EntityManagerFactory emf = new PersistenceConfiguration("myPU")
    .jtaDataSource("java:global/jdbc/MyDS")
    .managedClass(Customer.class)
    .property(PersistenceConfiguration.LOCK_TIMEOUT, 5000)
    .createEntityManagerFactory();

// Transaction Helpers
emf.runInTransaction(em -> {
    em.persist(new Customer("Alice"));
});

String name = emf.callInTransaction(em -> {
    return em.find(Customer.class, 1L).getName();
});

// Java Records als Embeddable
@Embeddable
public record Address(String street, String city, String zip) {}

// java.time Support
@Entity
public class Event {
    private Instant createdAt;         // Instant direkt unterstützt
    private Year releaseYear;          // Year direkt unterstützt
}
```

## Jakarta Data 1.0 (NEU)

Repository Pattern — Provider-agnostisch (JPA, NoSQL).

### Repository definieren

```java
@Repository
public interface ProductRepository extends CrudRepository<Product, Long> {

    // Derived Query (aus Methodenname)
    List<Product> findByCategory(String category);
    List<Product> findByPriceLessThan(double maxPrice);
    Optional<Product> findByName(String name);
    long countByCategory(String category);
    boolean existsByName(String name);

    // Mit Sortierung
    List<Product> findByCategory(String category, Sort<?>... sorts);

    // Mit Paginierung
    @Find @OrderBy("name")
    Page<Product> byCategory(String category, PageRequest pageRequest);

    // JDQL Query
    @Query("WHERE name LIKE :pattern AND active = true")
    List<Product> search(String pattern, Limit max);

    // JPQL Query (bei JPA-Backend)
    @Query("SELECT p FROM Product p WHERE p.price BETWEEN :min AND :max ORDER BY p.price")
    List<Product> findInPriceRange(double min, double max);

    // Update Query
    @Query("UPDATE Product SET price = price * :factor WHERE category = :cat")
    int updatePriceByCategory(String cat, double factor);

    // Explizite Lifecycle-Annotationen
    @Insert  Product create(Product p);
    @Insert  List<Product> createAll(List<Product> products);
    @Delete  void remove(Product p);
}
```

### Verwendung

```java
@Inject ProductRepository repo;

// CRUD
Product p = new Product("Widget", 9.99, "tools");
repo.create(p);
Optional<Product> found = repo.findByName("Widget");
repo.remove(p);

// Paginierung
PageRequest page = PageRequest.ofPage(1).size(20);
Page<Product> results = repo.byCategory("tools", page);
long total = results.totalElements();
List<Product> content = results.content();

// Cursor-basierte Paginierung
CursoredPage<Product> cursored = repo.findAll(pageRequest, Order.by(Sort.asc("id")));
PageRequest next = cursored.nextPageRequest();

// Sortierung
repo.findByCategory("books", Sort.asc("price"), Sort.desc("name"));
```

### Repository-Hierarchie

```
DataRepository<T, K>          Marker-Interface
  └─ BasicRepository<T, K>    @Save, @Find, @Delete
       └─ CrudRepository<T, K>  + @Insert, @Update
```

## Jakarta Security 4.0

### Authentication

```java
// Basic Auth
@BasicAuthenticationMechanismDefinition(realmName = "myRealm")

// Form Auth
@FormAuthenticationMechanismDefinition(
    loginToContinue = @LoginToContinue(
        loginPage = "/login.xhtml",
        errorPage = "/error.xhtml"))

// OpenID Connect
@OpenIdAuthenticationMechanismDefinition(
    providerURI = "https://provider.example.com",
    clientId = "myClient",
    clientSecret = "mySecret",
    redirectURI = "${baseURL}/callback",
    scope = {"openid", "email", "profile"})
```

### SecurityContext

```java
@Inject SecurityContext sec;

Principal caller = sec.getCallerPrincipal();
boolean isAdmin = sec.isCallerInRole("admin");
Set<String> roles = sec.getAllDeclaredCallerRoles();  // NEU in 4.0
```

## Jakarta Validation 3.1

```java
// Auf Feldern
@NotNull                          // nicht null
@NotEmpty                         // nicht null und nicht leer
@NotBlank                         // nicht null und nicht nur Whitespace
@Size(min = 2, max = 100)        // Länge/Größe
@Min(0) @Max(999)                // Zahlenbereiche
@Positive @PositiveOrZero
@Email                            // E-Mail-Format
@Pattern(regexp = "^[A-Z].*")    // Regex
@Past @Future                     // Datum
@Digits(integer = 5, fraction = 2)

// Auf Methoden-Parametern
public @NotNull Order place(@NotNull @Valid OrderRequest req) { ... }

// Custom Validator
@Constraint(validatedBy = PhoneValidator.class)
@Target({FIELD, PARAMETER}) @Retention(RUNTIME)
public @interface ValidPhone {
    String message() default "Invalid phone";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class PhoneValidator implements ConstraintValidator<ValidPhone, String> {
    public boolean isValid(String val, ConstraintValidatorContext ctx) {
        return val != null && val.matches("\\+?[0-9]{10,15}");
    }
}

// NEU: Java Records werden unterstützt
```

## Jakarta JSON-B 3.0

```java
Jsonb jsonb = JsonbBuilder.create();

// Serialisieren / Deserialisieren
String json = jsonb.toJson(product);
Product p = jsonb.fromJson(json, Product.class);

// Konfiguration
JsonbConfig config = new JsonbConfig()
    .withFormatting(true)
    .withNullValues(false)
    .withPropertyNamingStrategy(PropertyNamingStrategy.LOWER_CASE_WITH_UNDERSCORES);
Jsonb jsonb = JsonbBuilder.create(config);
```

### Annotationen

```java
@JsonbProperty("full_name")           // JSON-Feldname
@JsonbTransient                        // ausschließen
@JsonbDateFormat("yyyy-MM-dd")        // Datumsformat
@JsonbNumberFormat("#0.00")           // Zahlenformat
@JsonbNillable                         // null-Werte einschließen
@JsonbPropertyOrder({"id", "name"})   // Reihenfolge
@JsonbCreator                          // Custom Deserialisierung
```

### NEU: Polymorphie

```java
@JsonbTypeInfo(key = "@type", value = {
    @JsonbSubtype(alias = "dog", type = Dog.class),
    @JsonbSubtype(alias = "cat", type = Cat.class)
})
public abstract class Animal { ... }
// {"@type":"dog","name":"Rex"} → Dog-Instanz
```

## Jakarta Concurrency 3.1

### Virtual Threads

```java
// Executor mit Virtual Threads definieren
@ManagedExecutorDefinition(
    name = "java:app/concurrent/virtualExecutor",
    maxAsync = 10,
    virtual = true)           // Virtual Threads auf Java 21+

@ManagedScheduledExecutorDefinition(
    name = "java:app/concurrent/scheduler",
    virtual = true)
```

### Async & Scheduling

```java
// CDI-basiertes @Asynchronous (ersetzt EJB)
@Asynchronous
public CompletableFuture<Report> generateReport(Long id) {
    Report r = buildReport(id);
    return Asynchronous.Result.complete(r);
}

// Scheduling (ersetzt EJB @Schedule)
@Asynchronous(runAt = @Schedule(cron = "0 8 * * MON-FRI"))
public void morningJob() {
    // Läuft Mo-Fr um 08:00
}
```

## Quick Reference

```
Spec                      Version  Highlights
───────────────────────── ──────── ─────────────────────────────
Jakarta Data               1.0     Repository Pattern (NEU)
Jakarta Persistence        3.2     UNION, Records, ohne persistence.xml
Jakarta REST               4.0     @Context → @Inject, JSON Merge Patch
Jakarta CDI                4.1     CDI Lite, Verbesserungen
Jakarta Concurrency        3.1     Virtual Threads, @Schedule
Jakarta Security           4.0     OIDC, Multiple Auth Mechanisms
Jakarta Servlet            6.1     ByteBuffer, Virtual Threads
Jakarta Validation         3.1     Records Support
Jakarta JSON-B             3.0     Polymorphie
Jakarta Faces              4.1     UUIDConverter, CDI Alignment
```
{% endraw %}
