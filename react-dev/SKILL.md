---
name: react-development
description: React development best practices for functional components, hooks, state management, and performance optimization. Use when asked for "React help", "component creation", "refactoring components", "state management advice", or "fixing re-render issues". Optimized for senior developers building scalable applications.
license: MIT
metadata:
  author: jacobporci
  version: "1.0.0"
---

# React Development

Structured guidance for functional React development with Hooks. Covers dependency arrays, state lifting decisions, custom hooks patterns, and performance optimization trade-offs.

## When to Apply

- Building or refactoring functional components with Hooks
- Deciding between local state, lifted state, or Context API
- Optimizing components for performance
- Debugging unnecessary re-renders
- Creating reusable logic with custom hooks
- Implementing lazy loading or code splitting strategies

## Rule Categories by Priority

| Priority | Category | Impact | Focus |
| -------- | -------- | ------ | ----- |
| 1 | Dependency Arrays | HIGH | Prevent stale closures and infinite loops |
| 2 | State Lifting | HIGH | Choose correct state owner and minimize prop drilling |
| 3 | Custom Hooks | HIGH | Extract stateful logic for reusability |
| 4 | Context Usage | MEDIUM | Balance Context depth vs component coupling |
| 5 | Callback Optimization | MEDIUM | useCallback trade-offs vs re-render costs |
| 6 | Memoization Strategy | MEDIUM | React.memo / useMemo effectiveness |
| 7 | Lazy Loading | MEDIUM | Code splitting and dynamic imports |

## Quick Reference

### 1. Dependency Arrays (HIGH)

- `hooks-dependency-array-completeness` - Include all values that change
- `hooks-avoid-stale-closures` - Prevent stale state in event handlers
- `hooks-avoid-infinite-loops` - Detect circular dependencies

### 2. State Lifting (HIGH)

- `state-lift-shared-state` - Move state to lowest common parent
- `state-context-decision` - When to use Context vs prop drilling

### 3. Custom Hooks (HIGH)

- `hooks-extract-stateful-logic` - Create custom hooks for reusable logic

### 4. Optimization (MEDIUM)

- `perf-memoization-trade-offs` - When React.memo is worth the cost
- `perf-lazy-load-components` - Code splitting patterns

---

## Dependency Arrays (HIGH Impact)

### Incomplete Dependency Array

**Explanation:** Missing dependencies in useEffect cause stale closures where the effect runs with outdated variable values. This leads to bugs that only surface when specific variable changes don't trigger expected updates.

**Incorrect:**

```tsx
function Timer({ delay }) {
  const [count, setCount] = useState(0);

  useEffect(() => {
    // delay changes, but effect doesn't re-run → timer uses stale delay
    const timer = setInterval(() => setCount(c => c + 1), delay);
    return () => clearInterval(timer);
  }, []); // Missing 'delay' — will always use initial value

  return <div>{count}</div>;
}
```

**Correct:**

```tsx
function Timer({ delay }) {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const timer = setInterval(() => setCount(c => c + 1), delay);
    return () => clearInterval(timer);
  }, [delay]); // Re-run when delay changes

  return <div>{count}</div>;
}
```

**Decision:** Include every variable from the component scope that is referenced inside useEffect. Use ESLint `exhaustive-deps` rule.

---

## State Lifting Decision (HIGH Impact)

**Explanation:** Senior teams must distinguish between lifting state one level (to avoid prop drilling) vs. jumping to Context API. Over-lifting creates tight coupling; under-lifting creates deep prop chains. The decision depends on consumer count and intermediate component count.

**Incorrect:**

```tsx
// Over-lifting to Context too early (fragile to component tree changes)
const UserContext = createContext();

export function Layout() {
  const [user, setUser] = useState(null);
  return (
    <UserContext.Provider value={{ user, setUser }}>
      <Header />
      <Content />
      <Footer />
    </UserContext.Provider>
  );
}

function Content() {
  return <Page />; // Intermediate component unaware of context
}

function Page() {
  const { user } = useContext(UserContext); // Fine until design changes
}
```

**Correct:**

```tsx
// Lift state to parent, pass down explicitly to consumers
export function Layout() {
  const [user, setUser] = useState(null);
  return (
    <>
      <Header user={user} />
      <Content user={user} setUser={setUser} />
      <Footer user={user} />
    </>
  );
}

// Only use Context when 3+ levels of intermediates or 5+ consumers
const UserContext = createContext();
export function UserProvider({ children }) {
  const [user, setUser] = useState(null);
  return (
    <UserContext.Provider value={{ user, setUser }}>
      {children}
    </UserContext.Provider>
  );
}
```

**Decision Rule:** If ≤2 intermediate components and <5 consumers, lift state. If 3+ intermediates OR 5+ consumers, use Context.

---

## Custom Hooks for Logic Reuse (HIGH Impact)

**Explanation:** Senior developers avoid copy-pasting hooks logic across components. Extract to custom hooks when the same state + effects pattern repeats. Custom hooks improve testability, reduce component complexity, and create a reusable vocabulary.

**Incorrect:**

```tsx
function SearchUsers() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    if (!query) return;
    setLoading(true);
    fetch(`/api/users?q=${query}`)
      .then(r => r.json())
      .then(data => {
        setResults(data);
        setLoading(false);
      });
  }, [query]);

  return <div>{loading ? "..." : results.map(u => <div key={u.id}>{u.name}</div>)}</div>;
}

// Duplicated in another component
function SearchPosts() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    if (!query) return;
    setLoading(true);
    fetch(`/api/posts?q=${query}`)
      .then(r => r.json())
      .then(data => {
        setResults(data);
        setLoading(false);
      });
  }, [query]);

  return <div>{loading ? "..." : results.map(p => <div key={p.id}>{p.title}</div>)}</div>;
}
```

**Correct:**

```tsx
function useSearch(endpoint) {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    if (!query) return;
    setLoading(true);
    fetch(`/api/${endpoint}?q=${query}`)
      .then(r => r.json())
      .then(data => {
        setResults(data);
        setLoading(false);
      });
  }, [query, endpoint]);

  return { query, setQuery, results, loading };
}

function SearchUsers() {
  const { query, setQuery, results, loading } = useSearch("users");
  return <div>{loading ? "..." : results.map(u => <div key={u.id}>{u.name}</div>)}</div>;
}

function SearchPosts() {
  const { query, setQuery, results, loading } = useSearch("posts");
  return <div>{loading ? "..." : results.map(p => <div key={p.id}>{p.title}</div>)}</div>;
}
```

**Decision:** Extract when same pattern appears in 2+ components or when logic exceeds 10 lines.

---

## Context API Trade-offs (MEDIUM Impact)

**Explanation:** Context is a global state broadcast—every consumer re-renders when any value changes. For fine-grained state (user object, theme), this is acceptable. For high-frequency updates (form input, scroll position), Context causes performance regressions. Use Zustand, Jotai, or reducer for latter.

**Incorrect:**

```tsx
// Context broadcasts full form state on every keystroke
const FormContext = createContext();

export function FormProvider({ children }) {
  const [formData, setFormData] = useState({ name: "", email: "", message: "" });
  const handleChange = (field, value) => setFormData(prev => ({ ...prev, [field]: value }));

  return (
    <FormContext.Provider value={{ formData, handleChange }}>
      {children}
    </FormContext.Provider>
  );
}

function NameInput() {
  const { formData, handleChange } = useContext(FormContext);
  // Re-renders on email or message changes too — wasteful
  return <input value={formData.name} onChange={e => handleChange("name", e.target.value)} />;
}
```

**Correct:**

```tsx
// Use useReducer for form state—batches updates, better for complex forms
function useFormState(initialState) {
  const [formData, dispatch] = useReducer((state, action) => {
    return { ...state, [action.field]: action.value };
  }, initialState);

  const handleChange = (field, value) => dispatch({ field, value });
  return { formData, handleChange };
}

function FormComponent() {
  const { formData, handleChange } = useFormState({ name: "", email: "", message: "" });
  return (
    <>
      <input value={formData.name} onChange={e => handleChange("name", e.target.value)} />
      <input value={formData.email} onChange={e => handleChange("email", e.target.value)} />
    </>
  );
}
```

**Decision:** Use Context for infrequent state (user, theme, auth). Use useState/useReducer for form input; use external store (Zustand/Jotai) for high-frequency updates.

---

## Memoization Cost–Benefit (MEDIUM Impact)

**Explanation:** React.memo and useMemo prevent re-renders but add memory and comparison overhead. Overuse defeats purpose. Profile first; only memoize if component re-renders excessively AND re-render cost exceeds memoization cost.

**Incorrect:**

```tsx
// Memoizing cheap component — comparison overhead > benefit
const SmallButton = React.memo(({ onClick, label }) => {
  return <button onClick={onClick}>{label}</button>;
});

// Every render still creates new function — memo ineffective
function Parent() {
  return <SmallButton onClick={() => console.log("click")} label="Click" />;
}
```

**Correct:**

```tsx
// Only memoize expensive component
const ExpensiveList = React.memo(({ items, onItemClick }) => {
  return items.map(item => (
    <div key={item.id} onClick={() => onItemClick(item.id)}>
      {item.name}
    </div>
  ));
}, (prev, next) => prev.items === next.items && prev.onItemClick === next.onItemClick);

function Parent() {
  const handleItemClick = useCallback(id => console.log(id), []);
  const items = useMemo(() => largeArray, [largeArray]);

  return <ExpensiveList items={items} onItemClick={handleItemClick} />;
}
```

**Decision:** Only memoize if DevTools shows 3+ re-renders per interaction AND component renders in <10ms without memo. Always pair React.memo with useCallback for props.

---

## How to Use

**Step 1: Identify the problem**  
Determine which category applies: dependency arrays (stale state), state lifting (prop drilling), custom hooks (code duplication), Context usage (performance), or memoization (re-renders).

**Step 2: Consult the rule**  
Read the Incorrect example to recognize the anti-pattern. Check the Decision criteria.

**Step 3: Apply the pattern**  
Implement the Correct example, adapting to your codebase. Use ESLint rules to enforce (e.g., `exhaustive-deps`, `no-const-assign`).

**Step 4: Validate**  
Test with React DevTools Profiler to confirm re-render reduction. Measure first, optimize second.

---

## References

- [React Hooks Rules](https://react.dev/reference/rules/rules-of-hooks)
- [useEffect Dependency Array](https://react.dev/reference/react/useEffect#specifying-reactive-dependencies)
- [useContext Performance](https://react.dev/reference/react/useContext#caveats)
- [React.memo](https://react.dev/reference/react/memo)
- [Custom Hooks](https://react.dev/learn/reusing-logic-with-custom-hooks)
