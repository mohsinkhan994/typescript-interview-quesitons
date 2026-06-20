# Day 1 — TypeScript Deep Dive for React (Interview-Ready Notes)

---

## 1. Interfaces vs Types (The Real Difference)

**Core idea:** Both describe the *shape* of data. In 90% of day-to-day React work they're interchangeable. The differences only matter in specific edge cases — and interviewers love asking about those edge cases.

| Feature | `interface` | `type` |
|---|---|---|
| Extending | `extends` keyword | `&` (intersection) |
| Declaration merging | ✅ Yes (can re-open and add fields) | ❌ No |
| Unions / primitives / tuples | ❌ Cannot represent | ✅ Can represent |
| Mapped types | ❌ No | ✅ Yes |
| Use for objects/classes | ✅ Preferred | ✅ Works |

**Declaration merging — the key real difference:**

```ts
interface User {
  name: string;
}
interface User {
  age: number;
}
// User is now automatically { name: string; age: number }
```

This is **not possible** with `type` — TypeScript will throw a "Duplicate identifier" error.

```ts
type User = { name: string };
type User = { age: number }; // ❌ Error: Duplicate identifier 'User'
```

**Real-life example:** Think of `interface` like a **Google Doc that multiple teams can edit and merge into one final document** — a library can declare a base shape (e.g. Express's `Request` interface), and your app can "extend" it elsewhere in the codebase by re-declaring it. `type` is like a **sealed PDF** — once defined, it's final; you can only build something *new* on top of it (via `&`), not edit the original.

**Practical React example:**
```ts
// type — needed because you're unioning primitives
type ButtonVariant = "primary" | "secondary" | "danger";

// interface — preferred for component props (object shape, can be extended)
interface ButtonProps {
  variant: ButtonVariant;
  label: string;
}

interface IconButtonProps extends ButtonProps {
  icon: React.ReactNode;
}
```

**Rule of thumb (also your interview answer):**
> "I use `interface` for object shapes, especially component props and class contracts, because it supports declaration merging and reads cleanly with `extends`. I use `type` when I need unions, intersections, tuples, or mapped/conditional types — things `interface` simply can't express."

---

## 2. Generics (`<T>`, Generic Components)

**Core idea:** Generics let you write code that works with *any* type while still preserving type safety — instead of writing the same function/component multiple times for different types, or using `any` and losing safety entirely.

**Real-life example:** A generic is like a **shipping container** at a port. The container (your function/component) doesn't care what's inside — laptops, furniture, food — it just needs to know "what's inside" so it can handle it correctly (right size truck, right refrigeration). `<T>` is the label on the container that says "whatever is inside, call it T, and use that information consistently."

**Without generics (bad — loses type info):**
```ts
function getFirstItem(arr: any[]): any {
  return arr[0];
}
const first = getFirstItem([1, 2, 3]); // type is `any` — no autocomplete, no safety
```

**With generics (good — type is preserved):**
```ts
function getFirstItem<T>(arr: T[]): T {
  return arr[0];
}
const first = getFirstItem([1, 2, 3]); // TypeScript infers T = number
const name = getFirstItem(["a", "b"]); // TypeScript infers T = string
```

**Generic React component — exactly what was asked in "Practice":**

```tsx
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
  keyExtractor: (item: T) => string | number;
}

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
  return (
    <ul>
      {items.map((item) => (
        <li key={keyExtractor(item)}>{renderItem(item)}</li>
      ))}
    </ul>
  );
}

// Usage 1: list of users
interface User {
  id: number;
  name: string;
}
<List<User>
  items={users}
  keyExtractor={(u) => u.id}
  renderItem={(u) => <span>{u.name}</span>}
/>

// Usage 2: list of products — SAME component, fully type-safe, zero duplication
interface Product {
  sku: string;
  title: string;
}
<List<Product>
  items={products}
  keyExtractor={(p) => p.sku}
  renderItem={(p) => <span>{p.title}</span>}
/>
```

If you tried this without generics, you'd either need two separate components (`UserList`, `ProductList`) or you'd type `items: any[]` and lose all safety/autocomplete inside `renderItem`.

**Generic constraints** (a common follow-up question):
```ts
// "T must have an id" — constrains the generic instead of allowing anything
function findById<T extends { id: number }>(items: T[], id: number): T | undefined {
  return items.find((item) => item.id === id);
}
```

**Interview-ready answer:**
> "Generics let a single function or component be reusable across multiple types while TypeScript still tracks the *specific* type at each call site. It's the difference between `any` (no safety) and writing 10 duplicate components (no reusability) — generics give you both safety and reusability at once."

---

## 3. Union & Intersection Types

### Union (`|`) — "OR"
A value can be **one of several types**.

```ts
type Status = "loading" | "success" | "error";

function showStatus(status: Status) {
  if (status === "loading") return "Please wait...";
  if (status === "error") return "Something went wrong";
  return "Done!";
}
```

**Real-life example:** A union is like a **payment method selector** at checkout — you can pay with `"card" | "upi" | "cash"`. It's exactly one of those options, never a blend.

### Intersection (`&`) — "AND"
A value must satisfy **all** combined types at once.

```ts
interface WithTimestamps {
  createdAt: Date;
  updatedAt: Date;
}
interface User {
  name: string;
  email: string;
}

type UserWithTimestamps = User & WithTimestamps;
// Must have: name, email, createdAt, updatedAt — ALL of them
```

**Real-life example:** An intersection is like a **job role that requires multiple certifications** — e.g., a "Senior Driver" must have `DrivingLicense & HazmatCertification & FirstAidCertified`. They don't choose one; they need all of them simultaneously.

**Common React use case (props composition):**
```ts
type WithLoading = { isLoading: boolean };
type WithError = { error: string | null };

type DataTableProps = WithLoading & WithError & {
  data: Row[];
};
```

---

## 4. Utility Types: `Partial`, `Pick`, `Omit`, `Record`

These let you **derive new types from existing ones** instead of rewriting them — keeping a single source of truth.

### `Partial<T>` — makes all properties optional
**Real-life example:** Like a **PATCH request** for updating a user profile — you only send the fields that changed, not the entire object.

```ts
interface User {
  id: number;
  name: string;
  email: string;
}

function updateUser(id: number, updates: Partial<User>) {
  // updates can be { name: "New Name" } — no need to pass id/email
}

updateUser(1, { name: "Rahul" }); // ✅ valid, even though email/id are missing
```

### `Pick<T, Keys>` — selects only specific properties
**Real-life example:** Like a **visiting card** — it shows only your name and phone number from your full HR profile, not your salary or address.

```ts
type UserPreview = Pick<User, "id" | "name">;
// { id: number; name: string }  — email is excluded
```

### `Omit<T, Keys>` — removes specific properties (opposite of Pick)
**Real-life example:** Like a **public-facing API response** — you omit `password` or `internalId` before sending user data to the frontend.

```ts
type PublicUser = Omit<User, "email">;
// { id: number; name: string } — email is removed
```

### `Record<Keys, Type>` — builds an object type with specific keys and a consistent value type
**Real-life example:** Like a **price list / menu board** — every dish (key) maps to a price (value), and the structure is consistent for all entries.

```ts
type Role = "admin" | "editor" | "viewer";

const permissions: Record<Role, string[]> = {
  admin: ["read", "write", "delete"],
  editor: ["read", "write"],
  viewer: ["read"],
};
```

**Interview tip:** Mention that these are all built using TypeScript's **mapped types** under the hood (e.g. `Partial<T>` is essentially `{ [K in keyof T]?: T[K] }`), which shows depth beyond memorized usage.

---

## 5. Type Narrowing

**Core idea:** TypeScript can "narrow" a broad type down to a more specific one based on runtime checks — `typeof`, `instanceof`, `in`, equality checks, or custom **type guards**.

**Real-life example:** Like a **security checkpoint with multiple lanes** — everyone starts as "passenger" (broad type), but after checking their ticket (`typeof`/`in` check), they get routed/narrowed into "Economy passenger" or "Business passenger," and from that point on, the airline staff knows exactly which rules apply.

```ts
function printId(id: string | number) {
  if (typeof id === "string") {
    console.log(id.toUpperCase()); // ✅ TS knows id is `string` here
  } else {
    console.log(id.toFixed(2)); // ✅ TS knows id is `number` here
  }
}
```

**`in` operator narrowing:**
```ts
interface Cat { meow: () => void }
interface Dog { bark: () => void }

function makeSound(pet: Cat | Dog) {
  if ("meow" in pet) {
    pet.meow(); // narrowed to Cat
  } else {
    pet.bark(); // narrowed to Dog
  }
}
```

**Custom type guard (common in interviews):**
```ts
interface ApiSuccess { status: "success"; data: User }
interface ApiError { status: "error"; message: string }
type ApiResponse = ApiSuccess | ApiError;

function isSuccess(res: ApiResponse): res is ApiSuccess {
  return res.status === "success";
}

function handleResponse(res: ApiResponse) {
  if (isSuccess(res)) {
    console.log(res.data); // narrowed to ApiSuccess
  } else {
    console.log(res.message); // narrowed to ApiError
  }
}
```

---

## 6. `unknown` vs `any`

This is one of the **most-asked interview questions** because it tests real understanding vs surface-level memorization.

| | `any` | `unknown` |
|---|---|---|
| Type safety | None — disables all checks | Full — forces you to check first |
| Can call methods directly | ✅ Yes (dangerous) | ❌ No, must narrow first |
| Can assign to other types | ✅ Yes, anywhere | ❌ No, until narrowed |
| Use case | Escape hatch (avoid!) | Safe way to handle truly unknown data (e.g. API responses) |

**Real-life example:** `any` is like handing someone a **blank check** — they can do anything, no restrictions, and you have no protection if something goes wrong. `unknown` is like receiving a **locked box** — you know *something* is inside, but you must verify what it is (open it safely / check ID) before you're allowed to use it.

```ts
let valueAny: any = "hello";
valueAny.toUpperCase(); // ✅ compiles, but if it were a number at runtime → crash

let valueUnknown: unknown = "hello";
valueUnknown.toUpperCase(); // ❌ Error: Object is of type 'unknown'

if (typeof valueUnknown === "string") {
  valueUnknown.toUpperCase(); // ✅ now safe, narrowed to string
}
```

**Real React use case:** typing API responses or `catch` blocks.
```ts
try {
  // ...
} catch (error: unknown) {
  if (error instanceof Error) {
    console.log(error.message); // safe
  }
}
```

**Interview-ready answer:**
> "`any` turns off type checking entirely — it's effectively writing JavaScript. `unknown` is type-safe — it accepts any value, but TypeScript forces you to narrow it before you can use it. I always prefer `unknown` for things like API responses or catch blocks, and treat `any` as a last resort."

---

## 7. Type Assertions

**Core idea:** Telling TypeScript "I know more about this value's type than you do" — it does **not** change the runtime value, only how TypeScript treats it at compile time.

**Real-life example:** Like telling airport security **"trust me, this bag has already been checked"** — you're overriding their usual verification process. If you're wrong, there's no safety net; the plane (your app) can still crash at runtime.

```ts
const input = document.getElementById("username") as HTMLInputElement;
input.value = "Rahul"; // works because we asserted it's an input, not a generic Element

// Alternative syntax (not usable in .tsx files because of JSX conflict)
const input2 = <HTMLInputElement>document.getElementById("username");
```

**Dangerous misuse (common interview trap question):**
```ts
const value = "hello" as unknown as number; // ❌ compiles, but it's a lie — value is still a string at runtime
```

**Difference from casting in other languages:** Type assertions are **purely compile-time** — no conversion happens. `as number` does **not** turn a string into a number; use `Number(value)` for that.

**`!` (non-null assertion) — a related, frequently used operator:**
```ts
function getUser(id: number): User | undefined { /* ... */ }
const user = getUser(1)!; // "I promise this isn't undefined" — use sparingly
```

**Interview-ready answer:**
> "A type assertion changes how TypeScript *views* a value at compile time — it does not perform any actual conversion or runtime check. It's useful when TypeScript can't infer something I genuinely know to be true (like a DOM element's type), but it's also a way to silence the compiler incorrectly, so I use it sparingly and prefer type guards when possible."

---

## 8. Strict Mode

**Core idea:** `"strict": true` in `tsconfig.json` is an umbrella flag that turns on a bundle of strictness checks — it's the difference between TypeScript *catching real bugs* and TypeScript just being "JavaScript with some hints."

Key flags bundled inside `strict`:
- `strictNullChecks` — `null`/`undefined` aren't silently assignable to every type
- `noImplicitAny` — variables/params can't silently become `any`
- `strictFunctionTypes` — stricter checking of function parameter types
- `strictPropertyInitialization` — class properties must be initialized
- `alwaysStrict` — emits `"use strict"`, enforces JS strict mode rules

**Real-life example:** Strict mode is like switching a code review process from **"a teammate glances at your PR"** to **"a senior engineer reviews every line and blocks the merge on any ambiguity."** Without it, bugs like "forgot to handle null" slip into production. With it, the compiler stops you before you ship.

```ts
// strictNullChecks: false (bug waiting to happen)
function getLength(str: string) {
  return str.length;
}
let myStr: string = null; // ✅ allowed without strict mode — crashes at runtime later

// strictNullChecks: true
let myStr2: string = null; // ❌ Error: Type 'null' is not assignable to type 'string'
let myStr3: string | null = null; // ✅ must be explicit about nullability
```

```ts
// noImplicitAny catches this:
function add(a, b) { // ❌ Error: Parameter 'a' implicitly has an 'any' type
  return a + b;
}
```

**Interview-ready answer:**
> "Strict mode is a set of compiler flags — `strictNullChecks`, `noImplicitAny`, etc. — that make TypeScript catch entire categories of bugs (like missing null checks) at compile time instead of at runtime. Every production codebase I work in should have `strict: true`; turning it off defeats most of the purpose of using TypeScript."

---

## Practice Section

### 1. Generic `<List<T>>` Component
✅ Already covered in detail in **Section 2** above.

### 2. Reusable `<Button />` with Variant Props

```tsx
import React from "react";

type ButtonVariant = "primary" | "secondary" | "danger";
type ButtonSize = "sm" | "md" | "lg";

interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: ButtonVariant;
  size?: ButtonSize;
  isLoading?: boolean;
}

const variantStyles: Record<ButtonVariant, string> = {
  primary: "bg-blue-600 text-white",
  secondary: "bg-gray-200 text-black",
  danger: "bg-red-600 text-white",
};

const sizeStyles: Record<ButtonSize, string> = {
  sm: "px-2 py-1 text-sm",
  md: "px-4 py-2 text-base",
  lg: "px-6 py-3 text-lg",
};

export function Button({
  variant = "primary",
  size = "md",
  isLoading = false,
  children,
  disabled,
  ...rest
}: ButtonProps) {
  return (
    <button
      className={`${variantStyles[variant]} ${sizeStyles[size]} rounded`}
      disabled={disabled || isLoading}
      {...rest}
    >
      {isLoading ? "Loading..." : children}
    </button>
  );
}

// Usage
<Button variant="danger" size="lg" onClick={() => console.log("clicked")}>
  Delete
</Button>
```

**Key interview point:** Extending `React.ButtonHTMLAttributes<HTMLButtonElement>` means your component automatically supports every native button prop (`onClick`, `disabled`, `type`, `aria-*`, etc.) without manually re-declaring them — this is a very commonly tested pattern.

### 3. Converting a JS Component to TS Properly

**Before (JavaScript):**
```jsx
function UserCard({ user, onSelect }) {
  return (
    <div onClick={() => onSelect(user.id)}>
      <h3>{user.name}</h3>
      <p>{user.email}</p>
    </div>
  );
}
```

**After (TypeScript) — step-by-step thought process:**
```tsx
// Step 1: Define the shape of the data this component depends on
interface User {
  id: number;
  name: string;
  email: string;
}

// Step 2: Define the props interface (including function signatures)
interface UserCardProps {
  user: User;
  onSelect: (id: number) => void; // be specific about callback param/return types
}

// Step 3: Apply types to the component
function UserCard({ user, onSelect }: UserCardProps) {
  return (
    <div onClick={() => onSelect(user.id)}>
      <h3>{user.name}</h3>
      <p>{user.email}</p>
    </div>
  );
}
```

**What changed and why (good interview talking points):**
1. Extracted a `User` interface — avoids inline duplication if `User` is used elsewhere (e.g., in `List<User>` from Section 2).
2. Typed `onSelect` explicitly as `(id: number) => void` instead of leaving it implicit — this prevents someone from accidentally calling `onSelect(user)` (passing the whole object instead of just the id).
3. No `any` used anywhere — every prop has a concrete, meaningful type.
4. If `user` could be missing fields, you'd reach for `Partial<User>`; if this card needed only `name`/`email` for a "preview" variant, you'd use `Pick<User, "name" | "email">`.

---

## Must-Be-Able-to-Answer — Full Interview Answers

### Q1: Why TypeScript in large projects?
> "In large codebases, the biggest cost isn't writing code — it's *changing* code safely months later, often by people who didn't write it originally. TypeScript catches an entire class of bugs at compile time (wrong prop types, typos in object keys, calling a function with the wrong arguments) instead of at runtime in production. It also acts as living documentation — when I open a component, the props interface tells me exactly what it needs without reading the implementation. On a team, this means safer refactors (the compiler flags every place broken by a change), better autocomplete/IntelliSense, and fewer 'undefined is not a function' bugs in production. The tradeoff is some upfront verbosity, but on any project with more than a couple of contributors or a lifespan longer than a few months, that tradeoff is almost always worth it."

### Q2: When to use `interface` vs `type`?
> "I default to `interface` for object shapes — especially component props, API response shapes, and anything that might need to be extended later — because it supports declaration merging and reads naturally with `extends`. I switch to `type` when I need something `interface` can't express: union types (`"success" | "error"`), intersections, tuples, or mapped/conditional types. In practice on a React codebase: props and data models → `interface`; variant strings, combined types, utility-type derivations → `type`."

### Q3: What are generics and why are they powerful?
> "Generics let me write a single function, type, or component that works correctly across multiple types, without losing type safety and without duplicating code. Instead of writing `UserList`, `ProductList`, `OrderList` separately, or writing one `List` component with `items: any[]` and losing all safety, I write one `List<T>` component, and TypeScript automatically figures out and enforces the correct type at every call site — including inside callbacks like `renderItem`. They're powerful because they solve the tension between reusability and type safety, which `any` and code duplication each solve only one side of."

---

### Quick Self-Check Before the Interview
- [ ] Can I explain declaration merging out loud, with an example, in under 30 seconds?
- [ ] Can I write a generic function from scratch on a whiteboard?
- [ ] Can I explain `unknown` vs `any` with the "locked box vs blank check" analogy or my own?
- [ ] Can I name 2 flags inside `strict: true` and what each one catches?
- [ ] Can I justify *why* I chose `interface` over `type` (or vice-versa) for a given prop type, live?
