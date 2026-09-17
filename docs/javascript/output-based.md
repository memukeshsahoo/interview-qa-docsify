# JavaScript — Output-based questions

Say the output first, then why, then where it showed up in a UI.

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

Sync, then microtask (`then`), then timeout.

**Real-world example:** You `setTimeout(0)` to “run after render” but a `Promise.then` still runs first and reads a DOM node that is not painted yet. Use `requestAnimationFrame` for paint.

**Cross-question:** Two `then`s vs one timeout?

**Cross-answer:** Both `then`s before `b`.

---

## Q2. Closure + `var`

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
```

**Answer (output):** `3` `3` `3`

**Real-world example:** Three download buttons all downloaded the last file. Use `let` or close over `id`.

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

(in modules / strict; arrow `this` is outer, not `obj`)

**Real-world example:** Angular/TS class field arrows are bound to the instance — the opposite of this object-literal arrow. Know which you used.

**Cross-question:** `const f = obj.a; f();`

**Cross-answer:** `undefined` (`this` lost).

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

**Real-world example:** `pageSize == 0` vs `'0'` from an input. Use `===` and `Number(...)`.

**Cross-question:** `[] == false`?

**Cross-answer:** `true` (coercion). Another reason to avoid `==`.

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

`var` hoists as undefined. `let` is TDZ.

**Real-world example:** Using a `let` service above its line in a script crashed the page. `var` would have been `undefined` and failed later, quieter.

**Cross-question:** `function foo(){}` above the line?

**Cross-answer:** Callable. Function declarations hoist fully.

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

**Real-world example:** Edit-profile form changed the header city before Save. Copy nested objects or clone on edit.

**Cross-question:** `copy.name = 'B'`?

**Cross-answer:** `user.name` stays `A`. Top level is copied.

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

**Output:** `done` **before** saves finish.

**Real-world example:** “Close all” logged done while API calls were in flight. Use `for...of` + `await`.

**Cross-question:** `map` + `Promise.all`?

**Cross-answer:** Parallel, and you can wait.

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

`0` is falsy. Use `size ?? 10`.

**Real-world example:** Page size 0 from a bug became 10 and hid “empty page” tests.

**Cross-question:** `??` with `''`?

**Cross-answer:** `''` is kept (`??` only null/undefined).

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

`await` yields. `3` is still on the current stack.

**Real-world example:** `ngOnInit` called `await load()` and the next line in the caller still ran if the caller did not await. The spinner hid too early.

**Cross-question:** If `go()` is awaited by the caller?

**Cross-answer:** Caller also waits; order depends on that caller.
