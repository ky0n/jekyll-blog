---
layout: page
title: "Angular Cheat Sheet"
permalink: /cheatsheets/angular/
---
{% raw %}
_Updated for Angular v21 — includes Signals, Zoneless, Signal Forms & more._

## Signals

Angulars reaktive Primitive — stabil seit v20.

```typescript
import { signal, computed, effect, linkedSignal } from '@angular/core';

// Writable Signal
const count = signal(0);
count();                  // lesen → 0
count.set(5);             // setzen
count.update(n => n + 1); // updaten

// Computed Signal (read-only, automatisches Tracking)
const double = computed(() => count() * 2);

// Effect (Side-Effects bei Signal-Änderungen)
effect(() => {
  console.log('Count changed:', count());
});

// LinkedSignal (beschreibbares Computed — stabil seit v20)
const pageSize = signal(10);
const userPageSize = linkedSignal(() => pageSize());
// Reagiert auf pageSize(), kann aber manuell überschrieben werden:
userPageSize.set(25);  // überschreibt den abgeleiteten Wert
// Wenn pageSize() sich ändert → reset auf neuen abgeleiteten Wert
```

### Resource (experimentell)

```typescript
import { resource, httpResource } from '@angular/core';

// Async-Daten im Signal-Graph
const userId = signal(1);

const user = resource({
  request: () => ({ id: userId() }),   // wird getrackt
  loader: async ({ request }) => {
    const res = await fetch(`/api/users/${request.id}`);
    return res.json();
  }
});

user.value();    // Signal<User | undefined>
user.status();   // 'idle' | 'loading' | 'resolved' | 'error'
user.error();    // Fehler-Signal

// httpResource — reaktiver HTTP GET Wrapper (experimentell, seit v19.2)
const userData = httpResource<User>(() => `/api/users/${userId()}`);
// Varianten: httpResource.text(), httpResource.blob()

// Für POST/PUT/DELETE → weiterhin HttpClient verwenden
```

## Control Flow (Template-Syntax)

Stabil seit v18. Ersetzt `*ngIf`, `*ngFor`, `ngSwitch`. Kein `CommonModule` nötig.

```html
<!-- @if / @else if / @else -->
@if (user(); as u) {
  <p>Hallo {{ u.name }}</p>
} @else if (loading()) {
  <spinner />
} @else {
  <p>Nicht angemeldet</p>
}

<!-- @for — track ist Pflicht! -->
@for (item of items(); track item.id) {
  <li>{{ $index + 1 }}. {{ item.name }}</li>
} @empty {
  <li>Keine Einträge</li>
}
<!-- Implizite Variablen: $index, $first, $last, $even, $odd, $count -->

<!-- @switch -->
@switch (status()) {
  @case ('active') { <span class="green">Aktiv</span> }
  @case ('inactive') { <span class="red">Inaktiv</span> }
  @default { <span>Unbekannt</span> }
}
```

### @defer (Lazy Loading im Template)

```html
@defer (on viewport) {
  <heavy-component />
} @placeholder {
  <p>Scrollen zum Laden...</p>
} @loading (minimum 500ms) {
  <spinner />
} @error {
  <p>Laden fehlgeschlagen</p>
}

<!-- Trigger-Optionen -->
@defer (on idle)              <!-- Default: wenn Browser idle -->
@defer (on viewport)          <!-- wenn Element sichtbar wird -->
@defer (on interaction)       <!-- bei Klick/Fokus auf Placeholder -->
@defer (on hover)             <!-- bei Hover über Placeholder -->
@defer (on timer(3000))       <!-- nach 3 Sekunden -->
@defer (when condition())     <!-- wenn Signal/Expression true -->
```

## Components (Signal-basiert)

Standalone ist seit v19 der Default — `standalone: true` ist implizit.

```typescript
@Component({
  selector: 'app-user-card',
  template: `
    <h2>{{ name() }}</h2>
    <button (click)="clicked.emit()">Click</button>
    <input [value]="search()" (input)="search.set($event.target.value)" />
  `
})
export class UserCardComponent {
  // Input als Signal (ersetzt @Input)
  name = input.required<string>();           // Pflicht-Input
  age = input(0);                            // Optional mit Default
  label = input<string>();                   // Optional, undefined möglich

  // Output (ersetzt @Output)
  clicked = output<void>();                  // Emit: this.clicked.emit()
  selected = output<User>();                 // Emit: this.selected.emit(user)

  // Two-Way Binding mit model()
  search = model('');                        // Eltern: <app [(search)]="query" />

  // View Queries als Signals (ersetzt @ViewChild/@ViewChildren)
  myDiv = viewChild<ElementRef>('myDiv');              // Signal<ElementRef | undefined>
  myDiv = viewChild.required<ElementRef>('myDiv');     // Signal<ElementRef>
  items = viewChildren(ItemComponent);                 // Signal<readonly ItemComponent[]>

  // Content Queries
  header = contentChild<TemplateRef<any>>('header');
  tabs = contentChildren(TabComponent);
}
```

### Lifecycle

```typescript
// Klassisch (weiterhin verfügbar)
ngOnInit()        ngOnDestroy()
ngOnChanges()     ngAfterViewInit()
ngDoCheck()       ngAfterContentInit()

// Neu: afterRender / afterNextRender (seit v17)
constructor() {
  afterRender(() => {
    // Läuft nach jedem Render-Zyklus
    console.log('DOM updated');
  });

  afterNextRender(() => {
    // Läuft einmal nach dem nächsten Render
    this.chart = new Chart(this.canvas().nativeElement);
  });
}
```

## Dependency Injection

```typescript
// inject() Funktion (empfohlen — ersetzt Constructor Injection)
export class UserService {
  private http = inject(HttpClient);
  private router = inject(Router);
  private config = inject(APP_CONFIG, { optional: true });
}

// InjectionToken
export const APP_CONFIG = new InjectionToken<AppConfig>('app.config');

// Provider registrieren
bootstrapApplication(AppComponent, {
  providers: [
    { provide: APP_CONFIG, useValue: { apiUrl: '/api' } },
    { provide: UserService, useClass: UserService },
    { provide: Logger, useFactory: () => new Logger(inject(Config)) },
  ]
});
```

## Routing

```typescript
// app.routes.ts
export const routes: Routes = [
  { path: '', component: HomeComponent },

  // Lazy Loading
  {
    path: 'dashboard',
    loadComponent: () => import('./dashboard.component')
      .then(m => m.DashboardComponent),
    canActivate: [authGuard]
  },

  // Lazy-loaded Child Routes
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.routes')
      .then(m => m.ADMIN_ROUTES),
    canMatch: [adminGuard]   // verhindert sogar Chunk-Download
  },

  // Redirect & Wildcard
  { path: 'home', redirectTo: '', pathMatch: 'full' },
  { path: '**', component: NotFoundComponent }
];

// App-Setup
provideRouter(
  routes,
  withComponentInputBinding(),    // Route-Params → input() Signals
  withViewTransitions(),          // Native View Transitions API
  withPreloading(PreloadAllModules)
)
```

### Functional Guards

```typescript
// Klassenbasierte Guards sind deprecated — funktionale verwenden!
export const authGuard: CanActivateFn = (route, state) => {
  const auth = inject(AuthService);
  if (auth.isLoggedIn()) return true;
  return inject(Router).createUrlTree(['/login']);
};

// Resolve
export const userResolver: ResolveFn<User> = (route) => {
  return inject(UserService).getUser(route.params['id']);
};

// CanDeactivate (z.B. ungespeicherte Änderungen)
export const unsavedGuard: CanDeactivateFn<EditComponent> = (component) => {
  return component.hasUnsavedChanges()
    ? confirm('Ungespeicherte Änderungen verwerfen?')
    : true;
};
```

## HttpClient

### Setup

```typescript
provideHttpClient(
  withInterceptors([authInterceptor, loggingInterceptor]),
  withFetch(),   // fetch API statt XMLHttpRequest
  withXsrfConfiguration({ cookieName: 'XSRF-TOKEN' })
)
```

### Requests

```typescript
export class ApiService {
  private http = inject(HttpClient);

  // GET
  getUsers() {
    return this.http.get<User[]>('/api/users');
  }

  // POST
  createUser(user: User) {
    return this.http.post<User>('/api/users', user);
  }

  // PUT
  updateUser(id: number, user: User) {
    return this.http.put<User>(`/api/users/${id}`, user);
  }

  // DELETE
  deleteUser(id: number) {
    return this.http.delete(`/api/users/${id}`);
  }

  // Mit Optionen
  search(query: string) {
    return this.http.get<Result[]>('/api/search', {
      params: { q: query },
      headers: { 'X-Custom': 'value' },
      observe: 'response'   // vollständige HttpResponse
    });
  }
}
```

### Functional Interceptors

```typescript
// Auth-Token anhängen
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).getToken();
  if (token) {
    req = req.clone({
      setHeaders: { Authorization: `Bearer ${token}` }
    });
  }
  return next(req);
};

// Logging
export const loggingInterceptor: HttpInterceptorFn = (req, next) => {
  const started = Date.now();
  return next(req).pipe(
    tap({ complete: () => console.log(`${req.url}: ${Date.now() - started}ms`) })
  );
};

// Retry bei Fehler
export const retryInterceptor: HttpInterceptorFn = (req, next) => {
  return next(req).pipe(retry({ count: 2, delay: 1000 }));
};
```

## Forms

### Reactive Forms

```typescript
export class MyFormComponent {
  private fb = inject(FormBuilder);

  form = this.fb.group({
    name: ['', [Validators.required, Validators.minLength(3)]],
    email: ['', [Validators.required, Validators.email]],
    age: [null as number | null, [Validators.min(0)]],
    addresses: this.fb.array([this.createAddress()])
  });

  createAddress() {
    return this.fb.group({
      street: [''],
      city: ['', Validators.required]
    });
  }

  get addresses() {
    return this.form.controls.addresses;
  }

  onSubmit() {
    if (this.form.valid) {
      console.log(this.form.value);   // typsicher!
    }
  }
}
```

```html
<form [formGroup]="form" (ngSubmit)="onSubmit()">
  <input formControlName="name" />
  @if (form.controls.name.errors?.['required']) {
    <span>Name ist Pflicht</span>
  }

  <input formControlName="email" type="email" />
  <input formControlName="age" type="number" />

  @for (addr of addresses.controls; track $index) {
    <div [formGroup]="addr">
      <input formControlName="street" />
      <input formControlName="city" />
    </div>
  }
  <button type="button" (click)="addresses.push(createAddress())">
    Adresse hinzufügen
  </button>

  <button type="submit" [disabled]="form.invalid">Speichern</button>
</form>
```

### Signal Forms (experimentell, v21)

```typescript
import { form } from '@angular/forms/signal';

// Signal-basiertes Form — deklarativ und reaktiv
const myForm = form(mySignalModel);
// Automatische Synchronisation zwischen Signal-Daten und UI
// Deklarative Validierung
// API kann sich noch ändern!
```

## Zoneless (Default seit v21)

Neue Apps verwenden kein `zone.js` mehr — Change Detection wird durch Signals getriggert.

```typescript
// Neues Projekt (v21+): automatisch zoneless
bootstrapApplication(AppComponent, {
  providers: [
    provideZonelessChangeDetection(),  // implizit in neuen v21 Projekten
  ]
});

// Was triggert Change Detection ohne Zone.js?
// ✅ Signal-Änderungen (.set(), .update())
// ✅ Template-Event-Bindings ((click), (input), etc.)
// ✅ Async Pipe
// ✅ markForCheck() / detectChanges()
// ❌ setTimeout/setInterval → manuell triggern oder Signals verwenden
// ❌ Promise.then() → Signals oder async pipe verwenden
```

## App-Setup (Standalone, kein NgModule)

```typescript
// main.ts
bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes,
      withComponentInputBinding(),
      withViewTransitions()
    ),
    provideHttpClient(
      withInterceptors([authInterceptor]),
      withFetch()
    ),
    provideAnimationsAsync(),
    provideZonelessChangeDetection(),

    // Legacy NgModule einbinden
    importProvidersFrom(SomeLegacyModule),
  ]
});
```

## Quick Reference

```
Aufgabe                          API
──────────────────────────────── ────────────────────────────
Reaktiver State                  signal()
Abgeleiteter State               computed()
Beschreibbarer abgeleiteter State linkedSignal()
Side-Effect                      effect()
Async-Daten (GET)                httpResource() / resource()
Input-Property                   input() / input.required()
Output-Event                     output()
Two-Way Binding                  model()
DOM-Referenz                     viewChild() / viewChildren()
Service injizieren               inject()
Lazy-Load Template               @defer
Conditional Rendering            @if / @else
Listen rendern                   @for (track!)
Route Guards                     CanActivateFn (funktional)
HTTP Interceptors                HttpInterceptorFn (funktional)
Change Detection                 Zoneless (Signals)
```
{% endraw %}
