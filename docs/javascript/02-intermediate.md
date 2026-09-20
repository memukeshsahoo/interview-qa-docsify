# JavaScript — Intermediate

Closures, prototypes, promises. This is the usual second wave of questions.

---

## Q1. What is a closure?

**Answer:**

A function remembers the variables around it, even after the outer function finished.

```javascript
function makeCounter() {
  let n = 0;
  return () => ++n;
}
```

I use this for private state and callbacks.

**Cross-question:** Memory leak?

**Cross-answer:**

A long-lived inner function keeps those outer variables alive. A DOM handler that closes over a big object can keep it after you left the page. Remove listeners.

---

## Q2. The `var` + `setTimeout` in a loop bug?

**Answer:**

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// 3 3 3
```

One `var i`. `let` makes a new `i` each loop.

**Cross-question:** Does `let` in `for` really get a new binding?

**Cross-answer:**

Yes. That is in the spec so this bug dies.

---

## Q3. Hoisting and TDZ?

**Answer:**

`function` declarations can be called above their line. `let`/`const` are in a dead zone until the line runs. Touch them early and you throw.

**Cross-question:** Classes?

**Cross-answer:**

You cannot use the class before its line. Like `let`, not like `function`.

---

## Q4. Prototypes vs `class`?

**Answer:**

`class` is nicer syntax over prototypes. Methods live on `Type.prototype` and are shared. Each instance has a link up the chain.

**Cross-question:** Own property vs prototype?

**Cross-answer:**

Fields on `this` are own. Methods are usually on the prototype. `Object.hasOwn` checks own.

---

## Q5. `call` / `apply` / `bind`?

**Answer:**

They set `this`. `bind` returns a new function. `call` runs now with a list of args. `apply` uses an array of args.

**Cross-question:** Bind an arrow?

**Cross-answer:**

Does not change `this`. Arrows are lexical.

---

## Q6. Promise states?

**Answer:**

Pending, then fulfilled or rejected. `.then` returns a new promise. If I forget to return or throw, I swallow bugs.

**Cross-question:** `all` vs `allSettled` vs `race` vs `any`?

**Cross-answer:**

`all`: one fail fails all.

`allSettled`: wait for every one.

`race`: first to finish, success or fail.

`any`: first success.

---

## Q7. `async`/`await` vs `.then`?

**Answer:**

Same engine. `await` is easier to read. `try/catch` works with `await`. Many `await`s in a row are sequential. `Promise.all` runs together.

**Cross-question:** `forEach(async ...)`?

**Cross-answer:**

`forEach` does not wait. Use `for...of` or `map` + `Promise.all`.

---

## Q8. Microtask vs macrotask?

**Answer:**

```javascript
console.log('a');
setTimeout(() => console.log('b'), 0);
Promise.resolve().then(() => console.log('c'));
console.log('d');
// a d c b
```

Promises before timers.

**Cross-question:** Can microtasks starve the screen?

**Cross-answer:**

Yes, if I keep queueing them forever. Paint waits until that queue is empty.

---

## Q9. Debounce vs throttle?

**Answer:**

Debounce: wait until the user pauses (search).

Throttle: at most once per X ms (scroll).

**Cross-question:** Tiny debounce?

**Cross-answer:**

```javascript
function debounce(fn, ms) {
  let t;
  return (...args) => {
    clearTimeout(t);
    t = setTimeout(() => fn(...args), ms);
  };
}
```

Clear the timer on destroy in Angular.

---

## Q10. Map vs Set vs WeakMap?

**Answer:**

Map: any key. Set: unique values. WeakMap: object keys, does not keep them alive, cannot loop keys. WeakMap is for extra data on a DOM node, not a string-id cache.

**Cross-question:** Cache API results in WeakMap?

**Cross-answer:**

Keys must be objects. String ids need a Map plus a size limit.

---

## Q11. Shallow copy traps?

**Answer:**

`{ ...obj }` shares nested objects. Changing `copy.address.city` changes the original. That also breaks OnPush if I thought I had a new object.

**Cross-question:** Fix?

**Cross-answer:**

Copy the nested part too, or use a structured clone when I really need a deep copy.

---

## Q12. `?.` `??` `??=`?

**Answer:**

Optional chain. Nullish default. `??=` assigns only if nullish, so `0` stays.

**Cross-question:** `||` for pageSize?

**Cross-answer:**

`0 || 10` is 10. Wrong. Use `??`.

---

## Q13. Iterators and generators?

**Answer:**

`for...of` uses iterators. `function*` + `yield` makes one. Observables are push. Generators are pull. UI events fit RxJS better.

**Cross-question:** `for...in` on arrays?

**Cross-answer:**

Gives index strings and extra keys. I do not use it for arrays.

---

## Q14. Event bubbling?

**Answer:**

The event goes down (capture) then up (bubble). `preventDefault` stops the browser action (form submit). `stopPropagation` stops other handlers on parents.

**Cross-question:** Row click vs button click?

**Cross-answer:**

The button click bubbles to the row. I stop propagation if the row should not also fire.

---

## Q15. CORS from the Angular side?

**Answer:**

I cannot “fix CORS” in Angular except a **dev proxy**. The API must send the right headers. Postman works because it is not a browser.

**Cross-question:** Is CORS auth?

**Cross-answer:**

No. CORS is a browser lock. The API still needs login.

---

## Q16. `bind` in a React/Angular template?

**Answer:**

`bind` in a template creates a new function every check. I bind once in the class or use an arrow field.

**Cross-question:** Arrow as a class field?

**Cross-answer:**

Keeps `this`. Costs one function per instance. Fine for UI handlers.

---

## Q17. Deep equality of objects?

**Answer:**

`===` is not deep. I compare ids, or JSON stringify if I accept the limits, or a helper. For tests, a matcher. For Angular, I often replace the object instead of deep-watching.

**Cross-question:** Lodash `isEqual` in a hot list?

**Cross-answer:**

Can be slow. Prefer id + version.

---

## Q18. `queueMicrotask`?

**Answer:**

Schedule a function as a microtask, same family as `then`. Useful to run after the current stack, before paint.

**Cross-question:** vs `setTimeout(0)`?

**Cross-answer:**

Microtask is sooner. Timeout is a task.

---

## Q19. Error boundaries? (JS)

**Answer:**

`window.onerror` and `unhandledrejection` catch leftovers. In Angular I also use `ErrorHandler`. I still try/catch near risky I/O.

**Cross-question:** Swallow in empty catch?

**Cross-answer:**

Never. At least log with context.

---

## Q20. `Object.freeze` vs `const`?

**Answer:**

`const` = cannot reassign the variable. `freeze` = cannot add/change keys (shallow). Nested objects are not frozen.

**Cross-question:** Frozen in Angular state?

**Cross-answer:**

Nice for discipline. Signals + new objects are the usual path.

---

## Q21. Module cycles?

**Answer:**

A imports B imports A. You can see a `let` still in the TDZ. I put shared bits in a third file.

**Cross-question:** `import type` in TS?

**Cross-answer:**

Erased. No runtime cycle from types only.

---

## Q22. `new Promise` executor runs when?

**Answer:**

Right now, when you construct it. The resolve/reject functions are for later. I do not wrap an existing promise in `new Promise` without a reason (that is the “promise constructor anti-pattern”).

**Cross-question:** Convert callback API?

**Cross-answer:**

Yes, `new Promise` is for wrapping `fs.readFile(cb)` style. Not for wrapping `fetch`.

---

## Q23. `Array.from` vs spread?

**Answer:**

Both copy array-like things. `Array.from` can map in one step. Spread needs something iterable.

**Cross-question:** `NodeList`?

**Cross-answer:**

`querySelectorAll` is not a real array. I use `Array.from` or spread to get `.map`.

---

## Q24. Strict mode?

**Answer:**

Modules and classes are strict. Assign to a missing variable throws. `this` is not `window` in a bare call. `with` is banned.

**Cross-question:** Script became a module and `this` broke?

**Cross-answer:**

Yes. Strict. I use `window` on purpose if I need the global.

---

## Q25. How do you avoid callback hell without async/await?

**Answer:**

Promise chains. Still better than nested callbacks. Today I use async/await. I still return the promise so callers can wait.


---
## Additional Questions
### Q26. `Promise.all` vs `Promise.allSettled`?
**Answer:** Promise.all fails when one promise rejects. allSettled waits for every promise and gives me each result, so it is useful for independent dashboard calls.
### Q27. Debounce vs throttle?
**Answer:** Debounce waits until activity stops, good for search. Throttle limits how often something runs, good for scroll or resize events.
### Q28. What is the nullish coalescing operator?
**Answer:** `value ?? fallback` uses the fallback only for null or undefined. Unlike `||`, it does not replace valid values like 0 or false.
