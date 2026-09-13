# TypeScript Learning Log

A running personal collection of TypeScript rules of thumb, good practices, and lessons learned, gathered day after day while working on real projects (but written generically so they're useful anywhere).

Entries are logged chronologically (oldest first). This log will be reorganized into logical chapters once it grows large enough to need them.

## Index

1. [Structural typing: functions don't "know" their type alias](#1-structural-typing-functions-dont-know-their-type-alias)
2. [`import type` and type-file naming conventions](#2-import-type-and-type-file-naming-conventions)

## Entries

### 1. Structural typing: functions don't "know" their type alias

TypeScript uses **structural typing** (a.k.a. duck typing) instead of nominal typing. This means a function is never "declared" as being of a certain type alias — TypeScript simply compares shapes (parameter types and return type) wherever that type is actually *used*, such as when assigning the function to a variable, passing it as an argument, or storing it in an array/object field typed with that alias. A plain function declaration on its own is just inferred as its own signature; nothing links it to a named type unless you put it in a context that expects that type.

Compatibility rules for function shapes follow this practical guideline: a function can have **fewer** parameters or **optional** extra parameters than the target type expects, but not **more required** parameters — because callers using the target type will call it with only the parameters the type declares, and any extra *required* parameter would be left `undefined`.

Declaring a reusable function type alias (like `SupportResponse` below) only pays off once you actually use it somewhere as a contract — e.g. as a parameter type for a function that accepts callbacks, a variable annotation, or an array/object field type. Used that way, it gives you documentation, reusability, and a single place to update if the shape changes, and TypeScript will flag every function that's no longer compatible.

```ts
// Type alias describing the expected shape of a function
export type SupportResponse = (name: string) => string;

// This function is NOT explicitly typed as SupportResponse,
// but its inferred shape (name: string) => string matches it.
export function greetCustomer(name: string) {
    return `Hello ${name}, welcome to Support.ai!`;
}

// The type alias only becomes meaningful when used as a contract,
// e.g. constraining what functions are acceptable here:
function handleRequest(name: string, responder: SupportResponse) {
    return responder(name);
}

handleRequest("Alice", greetCustomer); // ✅ compiles - shape matches

// Adding an extra REQUIRED parameter breaks compatibility:
function greetWithAge(name: string, age: number) {
    return `Hello ${name}, you are ${age}`;
}
// handleRequest("Alice", greetWithAge); // ❌ Error: extra required param

// Making the extra parameter OPTIONAL restores compatibility:
function greetWithOptionalAge(name: string, age?: number) {
    return `Hello ${name}${age ? `, you are ${age}` : ""}`;
}
handleRequest("Alice", greetWithOptionalAge); // ✅ compiles - optional param is fine
```

**Further reading:** [TypeScript Handbook: Type Compatibility](https://www.typescriptlang.org/docs/handbook/type-compatibility.html)

### 2. `import type` and type-file naming conventions

Use `import type { X, Y } from "./module"` when you only need `X`/`Y` for type-checking, not at runtime. Unlike a regular `import`, an `import type` is completely erased during compilation — no JS import statement survives in the output. This matters for several reasons: it guarantees no unnecessary runtime module loading or side effects just to grab a type; it plays well with transpilers that process files in isolation (Babel, esbuild, swc, or the `isolatedModules` TS flag) since those tools can't always tell a type-only import from a value import without full program analysis — `import type` removes the ambiguity explicitly; and it can prevent circular-import breakage, since a type-only reference never actually executes at runtime. You can also mix the two in one import when a module exports both types and values: `import { type User, someFunction } from "./models";`.

There's no single universal rule for naming the file that holds your types, but a few conventions are common and each carries a slightly different connotation:

| File/Folder | Typical Meaning |
|---|---|
| `types.ts` / `*.types.ts` | Pure type/interface definitions, no runtime logic — the most generic, most common choice |
| `models.ts` / `models/` | Domain entities that mirror your data layer (DB schema, API shape) — borrowed from ORM terminology (Rails, Mongoose, Prisma) |
| `interfaces.ts` | Interface-only definitions — a slightly dated but still-seen style |
| `dto.ts` | Data Transfer Objects — request/response shapes in API-heavy codebases |
| `user.types.ts` (colocated) | Feature-scoped types, one file per domain/feature, common in larger apps |

Whatever you choose, keep the **file name** in kebab-case or camelCase consistent with the rest of the project, but always name the **types/interfaces themselves** in PascalCase (`User`, `Post`) — that part *is* a firm, widely-followed convention.

```ts
// models.ts - domain entities, type-only, no runtime code
export interface User {
    id: string;
    name: string;
}

export interface Post {
    id: string;
    authorId: string;
    title: string;
}

// consumer.ts - imported purely for type-checking, erased at compile time
import type { User, Post } from "./models";

function printUser(user: User): void {
    console.log(user.name);
}
```

**Further reading:** [TypeScript Handbook: Modules - Import Type](https://www.typescriptlang.org/docs/handbook/2/modules.html#importing-types)
