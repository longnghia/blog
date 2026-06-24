---
date: "2026-06-24T10:37:48+07:00"
draft: false
title: "Why Array.forEach() Doesn't Work with async/await (And What to Use Instead)"
summary: "forEach() doesn't wait for async callbacks because it doesn't handle Promises. Use for...of for sequential execution or Promise.all() for parallel async operations."
categories:
  - Code
tags:
  - javascript
  - async-await
  - promises
---

{{< gpt >}}

`Array.prototype.forEach()` doesn't "respect" `async/await` because **it was never designed to handle promises**.

Consider this code:

```js
const items = [1, 2, 3];

items.forEach(async (item) => {
  await new Promise((resolve) => setTimeout(resolve, 1000));
  console.log(item);
});

console.log("done");
```

Output:

```txt
done
1
2
3
```

Many developers expect:

```txt
1
2
3
done
```

but that's not what happens.

### Why?

`forEach` simply calls your callback for every element and immediately returns:

```js
forEach(callback) {
  for (let i = 0; i < this.length; i++) {
    callback(this[i]);
  }
}
```

When the callback is `async`, it returns a Promise:

```js
async (item) => { ... }
```

So internally it's effectively doing:

```js
callback(1); // returns Promise
callback(2); // returns Promise
callback(3); // returns Promise
return; // forEach finishes immediately
```

`forEach`:

- does not collect the promises
- does not await them
- does not return a promise itself

Therefore `await items.forEach(...)` is also useless:

```js
await items.forEach(async (item) => {
  await doSomething(item);
});
```

because `forEach()` returns `undefined`, not a Promise.

---

## Use `for...of` for sequential async work

```js
for (const item of items) {
  await doSomething(item);
}
```

This waits for each iteration before moving to the next.

---

## Use `Promise.all()` for parallel async work

```js
await Promise.all(
  items.map(async (item) => {
    await doSomething(item);
  }),
);
```

This starts all operations immediately and waits until all finish.

---

## Why wasn't `forEach` changed?

Changing its behavior would break existing JavaScript code.

Historically:

- `forEach` was added in ES5 (2009).
- Promises arrived later.
- `async/await` arrived much later (ES2017).

By then, millions of programs relied on `forEach` being synchronous and returning `undefined`, so JavaScript couldn't safely redefine it to await promises.

### Rule of thumb

- Need **sequential** async processing → `for...of` + `await`
- Need **parallel** async processing → `Promise.all(items.map(...))`
- Avoid `async` callbacks inside `forEach` unless you intentionally don't care when they finish.
