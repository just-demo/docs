# TypeScript vs modern JavaScript syntax

> If it disappears when you rename the file from `.ts` to `.js`, it's TypeScript. Otherwise, it's probably modern JavaScript.

Some common examples:

| Feature                              | JavaScript    | TypeScript |
| ------------------------------------ | ------------- | ---------- |
| `async/await`                        | ✅             | ✅          |
| Arrow functions `() => {}`           | ✅             | ✅          |
| Classes                              | ✅             | ✅          |
| Modules (`import/export`)            | ✅             | ✅          |
| Optional chaining `obj?.x`           | ✅             | ✅          |
| Nullish coalescing `??`              | ✅             | ✅          |
| Destructuring                        | ✅             | ✅          |
| Template strings `` `Hello ${x}` ``  | ✅             | ✅          |
| Type annotations `x: string`         | ❌             | ✅          |
| Interfaces                           | ❌             | ✅          |
| Enums                                | ❌             | ✅          |
| Generics `Array<T>`                  | ❌ (syntax)    | ✅          |
| `public/private/protected` on fields | ❌ (TS syntax) | ✅          |
| `type User = ...`                    | ❌             | ✅          |

For example:

```ts
async function hello(name: string): Promise<void> {
    console.log(`Hello ${name}`);
}
```

Split into:

**JavaScript:**

```js
async function hello(name) {
    console.log(`Hello ${name}`);
}
```

**TypeScript-only parts:**

```ts
name: string
Promise<void>
```

---

A practical trick when reading code:

1. Ask yourself: "Is this about runtime behavior or types?"
2. If it's about types, it's almost certainly TypeScript.
3. If it's about how the code executes, it's usually JavaScript.

Examples:

```ts
user?.address?.city
```

This changes runtime behavior → JavaScript.

```ts
interface User {
    name: string;
}
```

Exists only for type checking and disappears after compilation → TypeScript.

---

Here are some modern JavaScript features:

* `let` / `const`
* Arrow functions (`=>`)
* `class`
* `import` / `export`
* `async` / `await`
* Destructuring (`const {name} = user`)
* Spread operator (`...`)
* Optional chaining (`?.`)
* Nullish coalescing (`??`)
* `Map` and `Set`
* Promises

Those are all JavaScript features, not TypeScript.
