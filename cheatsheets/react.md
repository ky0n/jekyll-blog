---
layout: page
title: "React Cheat Sheet"
permalink: /cheatsheets/react/
---
{% raw %}
_Updated for React 19 — includes Actions, Server Components, React Compiler & neue Hooks._

## Komponenten

```jsx
// Function Component (Standard)
function Greeting({ name, age }) {
  return <h1>Hello, {name}! Age: {age}</h1>;
}

// Arrow Function
const Greeting = ({ name }) => <h1>Hello, {name}!</h1>;

// Default Props
function Button({ label = "Click me", variant = "primary" }) {
  return <button className={variant}>{label}</button>;
}

// Children
function Card({ title, children }) {
  return (
    <div className="card">
      <h2>{title}</h2>
      {children}
    </div>
  );
}
```

## Hooks

### useState

```jsx
const [count, setCount] = useState(0);
const [user, setUser] = useState({ name: '', age: 0 });

setCount(5);                     // direkt setzen
setCount(prev => prev + 1);      // basierend auf vorherigem Wert

// Objekte immer neu erstellen (immutable!)
setUser(prev => ({ ...prev, name: 'Max' }));
```

### useEffect

```jsx
// Nach jedem Render
useEffect(() => { document.title = `Count: ${count}`; });

// Nur beim Mount
useEffect(() => {
  const subscription = subscribe();
  return () => subscription.unsubscribe();   // Cleanup
}, []);

// Wenn sich count ändert
useEffect(() => {
  fetchData(count);
}, [count]);
```

### useRef

```jsx
const inputRef = useRef(null);
const renderCount = useRef(0);

// DOM-Element referenzieren
<input ref={inputRef} />
inputRef.current.focus();

// Wert speichern ohne Re-Render
renderCount.current++;
```

### useMemo & useCallback

```jsx
// Teuren Wert cachen (React Compiler macht das oft automatisch!)
const sorted = useMemo(() => items.sort(compareFn), [items]);

// Funktion referenziell stabil halten
const handleClick = useCallback(() => {
  doSomething(id);
}, [id]);
```

### useReducer

```jsx
function reducer(state, action) {
  switch (action.type) {
    case 'increment': return { count: state.count + 1 };
    case 'decrement': return { count: state.count - 1 };
    case 'reset':     return { count: 0 };
    default:          throw new Error('Unknown action');
  }
}

const [state, dispatch] = useReducer(reducer, { count: 0 });
dispatch({ type: 'increment' });
```

## React 19: Neue Hooks

### useActionState

```jsx
import { useActionState } from 'react';

// Ersetzt manuelles isLoading/error/success State-Management
async function submitForm(prevState, formData) {
  const name = formData.get('name');
  const result = await saveUser(name);
  return { success: true, message: `Saved ${name}` };
}

function MyForm() {
  const [state, formAction, isPending] = useActionState(submitForm, null);

  return (
    <form action={formAction}>
      <input name="name" />
      <button disabled={isPending}>
        {isPending ? 'Saving...' : 'Save'}
      </button>
      {state?.message && <p>{state.message}</p>}
    </form>
  );
}
```

### useFormStatus

```jsx
import { useFormStatus } from 'react-dom';

// Muss innerhalb eines <form> gerendert werden!
function SubmitButton() {
  const { pending, data } = useFormStatus();
  return (
    <button disabled={pending}>
      {pending ? 'Submitting...' : 'Submit'}
    </button>
  );
}

function MyForm() {
  return (
    <form action={handleSubmit}>
      <input name="email" />
      <SubmitButton />           {/* greift auf Form-Status zu */}
    </form>
  );
}
```

### useOptimistic

```jsx
import { useOptimistic } from 'react';

function TodoList({ todos, addTodo }) {
  const [optimisticTodos, addOptimistic] = useOptimistic(
    todos,
    (currentTodos, newTodo) => [...currentTodos, { ...newTodo, pending: true }]
  );

  async function handleAdd(formData) {
    const newTodo = { text: formData.get('text') };
    addOptimistic(newTodo);        // sofort anzeigen
    await addTodo(newTodo);        // Server-Request
    // Bei Erfolg: todos-Prop aktualisiert → optimistic wird überschrieben
    // Bei Fehler: optimistic wird automatisch zurückgerollt
  }

  return (
    <form action={handleAdd}>
      <input name="text" />
      <button>Add</button>
      {optimisticTodos.map(t => (
        <li style={{ opacity: t.pending ? 0.5 : 1 }}>{t.text}</li>
      ))}
    </form>
  );
}
```

### use()

```jsx
import { use, Suspense } from 'react';

// Promise lesen — React suspends automatisch
function UserProfile({ userPromise }) {
  const user = use(userPromise);     // kein await nötig!
  return <h1>{user.name}</h1>;
}

// Verwendung mit Suspense
<Suspense fallback={<Spinner />}>
  <UserProfile userPromise={fetchUser(id)} />
</Suspense>

// Context lesen (ersetzt useContext) — kann conditional aufgerufen werden!
function Theme({ isAdmin }) {
  if (isAdmin) {
    const theme = use(ThemeContext);    // conditional erlaubt!
    return <AdminPanel theme={theme} />;
  }
  return <UserPanel />;
}
```

## React 19: Weitere Neuerungen

### ref als normaler Prop (kein forwardRef mehr!)

```jsx
// Vorher
const Input = forwardRef((props, ref) => <input ref={ref} {...props} />);

// Jetzt: ref ist ein ganz normaler Prop
function Input({ ref, ...props }) {
  return <input ref={ref} {...props} />;
}

// Ref Cleanup Function
<div ref={(node) => {
  // Setup
  node.addEventListener('click', handler);
  // Cleanup (neu!)
  return () => node.removeEventListener('click', handler);
}} />
```

### Context ohne .Provider

```jsx
// Vorher
<ThemeContext.Provider value={theme}>
  <App />
</ThemeContext.Provider>

// Jetzt
<ThemeContext value={theme}>
  <App />
</ThemeContext>
```

### Document Metadata

```jsx
// title, meta, link direkt in Komponenten — React hoisted automatisch zu <head>
function BlogPost({ post }) {
  return (
    <article>
      <title>{post.title}</title>
      <meta name="description" content={post.summary} />
      <link rel="canonical" href={post.url} />
      <h1>{post.title}</h1>
      <p>{post.body}</p>
    </article>
  );
}
// Kein react-helmet mehr nötig!
```

## React Compiler (v1.0)

```bash
# Installation
npm install -D babel-plugin-react-compiler

# babel.config.js
module.exports = {
  plugins: [
    ['babel-plugin-react-compiler'],    // MUSS als erstes Plugin!
    // ... andere Plugins
  ],
};
```

```jsx
// Der Compiler macht automatisch:
// ✅ useMemo für teure Berechnungen
// ✅ useCallback für Event-Handler
// ✅ React.memo für Komponenten

// Du schreibst einfach:
function TodoList({ todos, filter }) {
  const filtered = todos.filter(t => t.status === filter);   // auto-memoized!
  return filtered.map(t => <TodoItem key={t.id} todo={t} />);
}
```

## Conditional Rendering

```jsx
// &&
{isLoggedIn && <Dashboard />}

// Ternary
{isLoggedIn ? <Dashboard /> : <Login />}

// Early Return
function Page({ user }) {
  if (!user) return <Login />;
  return <Dashboard user={user} />;
}
```

## Listen rendern

```jsx
{items.map(item => (
  <li key={item.id}>{item.name}</li>
))}

// Mit Index (nur wenn keine stabile ID vorhanden)
{items.map((item, index) => (
  <li key={index}>{item}</li>
))}

// Fragments
{items.map(item => (
  <Fragment key={item.id}>
    <dt>{item.term}</dt>
    <dd>{item.definition}</dd>
  </Fragment>
))}
```

## Event Handling

```jsx
<button onClick={() => handleClick(id)}>Click</button>
<input onChange={(e) => setName(e.target.value)} />
<form onSubmit={(e) => { e.preventDefault(); save(); }}>

// Event-Typen (TypeScript)
onClick:    React.MouseEvent<HTMLButtonElement>
onChange:   React.ChangeEvent<HTMLInputElement>
onSubmit:   React.FormEvent<HTMLFormElement>
onKeyDown:  React.KeyboardEvent<HTMLInputElement>
```

## Suspense & Error Boundaries

```jsx
import { Suspense } from 'react';
import { ErrorBoundary } from 'react-error-boundary';

<ErrorBoundary fallback={<p>Something went wrong.</p>}>
  <Suspense fallback={<Spinner />}>
    <AsyncComponent />
  </Suspense>
</ErrorBoundary>
```

## Quick Reference

```
Hook / API                 Zweck
────────────────────────── ──────────────────────────────
useState                   Lokaler State
useEffect                  Side-Effects, Subscriptions
useRef                     DOM-Refs, mutable Values
useMemo                    Gecachte Berechnung
useCallback                Stabile Funktionsreferenz
useReducer                 Komplexer State mit Actions
useContext / use(Context)  Context lesen
useActionState        ⭐   Form-Action State (pending/result)
useFormStatus         ⭐   Form-Status ohne Prop-Drilling
useOptimistic         ⭐   Optimistic UI Updates
use(Promise)          ⭐   Async Daten in Render lesen
React Compiler        ⭐   Auto-Memoization (Build-Time)

⭐ = Neu in React 19
```
{% endraw %}
