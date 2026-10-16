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

Sync code runs first, then the microtask (`then`), then the timeout. Step by step:

1. `console.log('a')` runs right away and prints `a`.
2. `setTimeout` does not run the callback now. It puts `b` in the task queue, the "later" line, even with a delay of `0`.
3. `Promise.resolve().then(...)` puts `c` in the microtask queue, the "as soon as the current code is done" line.
4. `console.log('d')` runs right away and prints `d`.
5. The main code is finished. The engine empties the microtask queue first, so `c` prints.
6. Only then does it take the next task, the timer, so `b` prints last.

Think of it as finishing the customer at the counter (sync code), then the quick "one more thing" requests (microtasks), and only then calling the next person in the queue (tasks).

**Real-world example:** You use `setTimeout(0)` to “run after render”, but a `Promise.then` still runs first and reads a DOM node that is not painted yet. For paint timing, use `requestAnimationFrame`, which runs right before the browser draws the next frame.

**Cross-question:** Two `then`s vs one timeout?

**Cross-answer:** Both `then`s run before `b`. They both sit in the microtask queue, and the engine empties that whole queue before it moves on to the timer.

---

## Q2. Closure + `var`

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
```

**Answer (output):** `3` `3` `3`

Why: `var` is not scoped to the loop body, so there is only one `i`, shared by every turn of the loop. The three arrow functions do not save the value of `i` when they are created. They remember the variable itself (that is what a closure is). The timeouts only run after the loop has completely finished, and by then the loop has pushed `i` up to `3`, which is what stopped it. So all three read the same `i` and print `3`.

**Real-world example:** We had three download buttons that all downloaded the last file. The fix is `let`, or closing over the `id`.

**Cross-question:** `let i`?

**Cross-answer:** `0` `1` `2`. With `let`, each turn of the loop gets its own fresh `i`, so each callback remembers its own value.

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

Step by step: `obj.a()` calls a normal function with `obj` before the dot, so `this` is `obj` and `this.n` is `1`. `obj.b()` calls an arrow function. An arrow does not get its own `this`, whatever is before the dot. It uses the `this` from where the object was written, which at the top of a classic script is the global `window`. `window` has no `n`, so it prints `undefined`.

**Real-world example:** Angular/TS class field arrows are bound to the instance — the opposite of this object-literal arrow. Inside a class, the "outer `this`" for a field is the instance being created, so the arrow keeps pointing to it. In an object literal there is no such thing, so the arrow gets the outside world's `this`. Know which you used.

**Cross-question:** `const f = obj.a; f();`

**Cross-answer:** `this` is lost. Copying the function into `f` and calling `f()` means there is nothing before the dot, so the function no longer knows about `obj`. In a classic (non-strict) script it prints `undefined`, because `this` falls back to the global object. In strict mode or a module, `this` is `undefined`, so it throws a TypeError.

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

Why, line by line:

- `0 == '0'`: `==` converts the types before comparing. The string `'0'` becomes the number `0`, so it is `true`.
- `0 === '0'`: `===` does no converting. A number and a string are different types, so it is `false` straight away.
- `null == undefined`: this is a special rule. With `==`, `null` and `undefined` count as equal to each other (and to nothing else), so `true`.
- `null === undefined`: they are two different types, so `false`.

**Real-world example:** `pageSize == 0` compared against `'0'` coming from an input. Form inputs always give strings, so `==` quietly converts them and hides the real type. Use `===` and convert with `Number(...)`, so the conversion is visible in the code.

**Cross-question:** `[] == false`?

**Cross-answer:** `true`, because of type coercion. Coercion means JavaScript converting a value to another type on its own. Here both sides end up as numbers: `false` becomes `0`, and the empty array becomes the empty string `''`, which then becomes `0`. So it compares `0 == 0`. That is surprising, because `[]` on its own counts as truthy in an `if`. Another reason to avoid `==`.

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

Hoisting means JavaScript sets up all the variables of a scope before running any of its lines. A `var` is created and set to `undefined` at that point, and it gets its real value only when its line runs. So `console.log(a)` finds `a` already there but empty, and prints `undefined`.

A `let` is also known about from the start, but it is locked until its line runs. That locked period is the temporal dead zone. Reading `b` before `let b = 2` throws a ReferenceError, the `catch` catches it, and it prints `err`.

**Real-world example:** Using a `let` service above its line in a script crashed the page. With `var` it would have been `undefined` and failed later, more quietly. A loud error at the right line is easier to fix than a silent `undefined` that breaks something further down.

**Cross-question:** `function foo(){}` above the line?

**Cross-answer:** You can call it. Function declarations are fully hoisted. The whole function, not just its name, is ready before any line runs, so you can call `foo()` even above where it is written.

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

Why: `typeof null` gives `object`. This is an old bug from the very first version of JavaScript, and it was never fixed because fixing it would break existing websites. `typeof []` also gives `object`, because arrays are a kind of object in JavaScript, and `typeof` does not tell them apart. To really check for an array, use `Array.isArray`, which gives `true`.

**Real-world example:** `typeof data === 'object'` was true for both `null` and the incident list. We used `Array.isArray` and `data == null` checks. `data == null` is one of the few good uses of `==`, because it is true for both `null` and `undefined`.

**Cross-question:** `typeof function(){}`?

**Cross-answer:** `function`. Functions are objects too, but `typeof` gives them their own answer, so this is a handy way to check that something can be called.

---

## Q7. Spread shallow copy

```javascript
const user = { name: 'A', addr: { city: 'Pune' } };
const copy = { ...user };
copy.addr.city = 'Goa';
console.log(user.addr.city);
```

**Answer (output):** `Goa`

Why: the spread `{ ...user }` makes a shallow copy. It creates a new outer object, but for the nested `addr` it only copies the reference, the "address" of the same inner object. So `copy.addr` and `user.addr` are one and the same object. Changing the city through `copy` changes it for `user` too. It is like photocopying a page that says "the keys are in the blue box": you now have two pages, but still only one blue box.

**Real-world example:** An edit-profile form changed the city in the header before the user clicked Save. Copy nested objects, or clone when editing starts.

**Cross-question:** `copy.name = 'B'`?

**Cross-answer:** `user.name` stays `A`. The top level is copied. `name` holds a plain string, and the copy got its own `name` field, so changing it on `copy` does not touch `user`.

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

Why: an `async` function runs normally, line by line, until it hits its first `await`. These callbacks have no `await`, so each `n.push(x)` happens right away while `forEach` is looping. By the time `console.log` runs, all three are pushed. `forEach` itself never waits for the promises the callbacks return. It just calls each callback and moves on.

Safer example they use:

```javascript
[1,2,3].forEach(async x => { await save(x); });
console.log('done');
```

**Output:** `done` prints **before** the saves finish. Each callback starts its `save`, hits `await`, and hands control back. `forEach` does not wait for any of them, so `console.log('done')` runs while all three saves are still in progress.

**Real-world example:** A “Close all” action logged done while the API calls were still in flight. Use `for...of` + `await`, which really waits for each save before going to the next one.

**Cross-question:** `map` + `Promise.all`?

**Cross-answer:** They run in parallel, and you can wait for all of them. `map` returns an array of promises, one per item, and `await Promise.all(...)` waits until every one has finished. That is usually faster than `for...of`, which does them one by one, and it still gives a clear point where everything is done.

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

`a || b` means "use `a` if it is truthy, otherwise `b`". Falsy values are the ones that count as false in a check: `0`, `''`, `false`, `null`, `undefined` and `NaN`. So `0 || 10` gives `10`, even though `0` was a real value someone passed. `undefined || 10` also gives `10`, which is what we wanted. `??` only falls back when the value is `null` or `undefined`, so `0 ?? 10` keeps `0`.

**Real-world example:** A page size of 0, caused by a bug, silently became 10 and hid our “empty page” tests.

**Cross-question:** `??` with `''`?

**Cross-answer:** `''` is kept, because `??` only replaces null/undefined. An empty string is falsy, so `''` with `||` would be replaced, but with `??` it stays as it is.

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

Step by step: calling `go()` runs its first line straight away, so `1` prints. At `await`, the rest of `go` (the line that prints `2`) is put aside as a microtask, and `go()` returns to the caller. The caller does not wait, so `console.log(3)` runs and prints `3`. Now the main code is finished, the microtask runs, and `2` prints last.

**Real-world example:** `ngOnInit` called `await load()`, but the next line in the caller still ran if the caller did not await. The spinner hid too early.

**Cross-question:** If `go()` is awaited by the caller?

**Cross-answer:** Then the caller also waits, and the order depends on that caller. For example, if the caller wrote `await go(); console.log(3);` inside its own async function, it would print `1`, `2`, `3`, because `3` now runs only after `go` has finished.
