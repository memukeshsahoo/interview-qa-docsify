# JavaScript — Output-based questions

Say the output first, then why, then where you saw it in a real UI.

---

## Q1. Event loop

```javascript
console.log('a');
setTimeout(() => console.log('b'), 0);
Promise.resolve().then(() => console.log('c'));
console.log('d');
```

**Answer (output):**

```text
a
d
c
b
```

Sync code runs first, then the microtask (`then`), then the timeout.

**Real-world example:** You use `setTimeout(0)` to “run after render”, but a `Promise.then` still runs first and reads a DOM node that is not painted yet. For paint timing, use `requestAnimationFrame`.

**Cross-question:** Two `then`s vs one timeout?

**Cross-answer:** Both `then`s run before `b`.

---

## Q2. Closure + `var`

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
```

**Answer (output):** `3` `3` `3`

**Real-world example:** We had three download buttons that all downloaded the last file. The fix is `let`, or closing over the `id`.

**Cross-question:** `let i`?

**Cross-answer:** `0` `1` `2`

---

## Q3. `this` and arrow

```javascript
const obj = {
  n: 1,
  a: function () { console.log(this.n); },
  b: () => console.log(this.n),
};
obj.a();
obj.b();
```

**Answer (output):**

```text
1
undefined
```

(in a classic browser script; an arrow function takes `this` from the outer scope, not from `obj`, so `this.n` is `undefined`. In an ES module the outer `this` is itself `undefined`, so `obj.b()` throws a TypeError instead.)

**Real-world example:** Angular/TS class field arrows are bound to the instance — the opposite of this object-literal arrow. Know which you used.

**Cross-question:** `const f = obj.a; f();`

**Cross-answer:** `this` is lost. In a classic (non-strict) script it prints `undefined`, because `this` falls back to the global object. In strict mode or a module, `this` is `undefined`, so it throws a TypeError.

---

## Q4. `==` vs `===`

```javascript
console.log(0 == '0');
console.log(0 === '0');
console.log(null == undefined);
console.log(null === undefined);
```

**Answer (output):**

```text
true
false
true
false
```

**Real-world example:** `pageSize == 0` compared against `'0'` coming from an input. Use `===` and convert with `Number(...)`.

**Cross-question:** `[] == false`?

**Cross-answer:** `true`, because of type coercion. Another reason to avoid `==`.

---

## Q5. Hoisting

```javascript
console.log(a);
var a = 1;
try { console.log(b); } catch (e) { console.log('err'); }
let b = 2;
```

**Answer (output):**

```text
undefined
err
```

`var` is hoisted as undefined. `let` is in the TDZ (temporal dead zone), so touching it early throws.

**Real-world example:** Using a `let` service above its line in a script crashed the page. With `var` it would have been `undefined` and failed later, more quietly.

**Cross-question:** `function foo(){}` above the line?

**Cross-answer:** You can call it. Function declarations are fully hoisted.

---

## Q6. `typeof null` and array

```javascript
console.log(typeof null);
console.log(typeof []);
console.log(Array.isArray([]));
```

**Answer (output):**

```text
object
object
true
```

**Real-world example:** `typeof data === 'object'` was true for both `null` and the incident list. We used `Array.isArray` and `data == null` checks.

**Cross-question:** `typeof function(){}`?

**Cross-answer:** `function`

---

## Q7. Spread shallow copy

```javascript
const user = { name: 'A', addr: { city: 'Pune' } };
const copy = { ...user };
copy.addr.city = 'Goa';
console.log(user.addr.city);
```

**Answer (output):** `Goa`

**Real-world example:** An edit-profile form changed the city in the header before the user clicked Save. Copy nested objects, or clone when editing starts.

**Cross-question:** `copy.name = 'B'`?

**Cross-answer:** `user.name` stays `A`. The top level is copied.

---

## Q8. `forEach` + async

```javascript
async function run() {
  const n = [];
  [1, 2, 3].forEach(async x => { n.push(x); });
  console.log(n.length);
}
run();
```

**Answer (output):** `3` (pushes are sync before first `await`; if the callback `await`ed first, length could be `0`)

Safer example they use:

```javascript
[1,2,3].forEach(async x => { await save(x); });
console.log('done');
```

**Output:** `done` prints **before** the saves finish.

**Real-world example:** A “Close all” action logged done while the API calls were still in flight. Use `for...of` + `await`.

**Cross-question:** `map` + `Promise.all`?

**Cross-answer:** They run in parallel, and you can wait for all of them.

---

## Q9. Default params and `0`

```javascript
function page(size) {
  size = size || 10;
  return size;
}
console.log(page(0));
console.log(page(undefined));
```

**Answer (output):**

```text
10
10
```

`0` is falsy, so `||` replaces it. Use `size ?? 10`.

**Real-world example:** A page size of 0, caused by a bug, silently became 10 and hid our “empty page” tests.

**Cross-question:** `??` with `''`?

**Cross-answer:** `''` is kept, because `??` only replaces null/undefined.

---

## Q10. Promise vs timeout order inside async

```javascript
async function go() {
  console.log(1);
  await Promise.resolve();
  console.log(2);
}
go();
console.log(3);
```

**Answer (output):**

```text
1
3
2
```

`await` pauses `go` and gives control back. `3` is still part of the current stack, so it logs first.

**Real-world example:** `ngOnInit` called `await load()`, but the next line in the caller still ran if the caller did not await. The spinner hid too early.

**Cross-question:** If `go()` is awaited by the caller?

**Cross-answer:** Then the caller also waits, and the order depends on that caller.
