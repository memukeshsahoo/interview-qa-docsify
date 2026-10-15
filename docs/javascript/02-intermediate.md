# JavaScript — Intermediate

Closures, prototypes, promises. This is the usual second round of questions.

---

## Q1. What is a closure?

**Answer:**

A closure is when a function remembers the variables around it, even after the outer function has finished.

```javascript
function makeCounter() {
  let n = 0;
  return () => ++n;
}
```

I use this for private state and for callbacks.

**Cross-question:** Memory leak?

**Cross-answer:**

Yes, it can cause one. A long-lived inner function keeps those outer variables alive. For example, a DOM handler that closes over a big object can keep it in memory even after you leave the page. So I remove listeners.

---

## Q2. The `var` + `setTimeout` in a loop bug?

**Answer:**

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// 3 3 3
```

There is only one `var i`, shared by all three callbacks. `let` creates a new `i` for each loop iteration.

**Cross-question:** Does `let` in `for` really get a new binding?

**Cross-answer:**

Yes. It is written into the spec so this bug goes away.

---

## Q3. Hoisting and TDZ?

**Answer:**

`function` declarations can be called above their line. `let`/`const` sit in a “dead zone” (the TDZ) until their line runs. If you touch them early, it throws.

**Cross-question:** Classes?

**Cross-answer:**

You cannot use a class before its line. It behaves like `let`, not like `function`.

---

## Q4. Prototypes vs `class`?

**Answer:**

`class` is just nicer syntax on top of prototypes. Methods live on `Type.prototype` and are shared. Each instance has a link up the prototype chain.

**Cross-question:** Own property vs prototype?

**Cross-answer:**

Fields set on `this` are own properties. Methods are usually on the prototype. `Object.hasOwn` checks for own properties.

---

## Q5. `call` / `apply` / `bind`?

**Answer:**

All three set `this`. `bind` returns a new function. `call` runs the function right now with a list of arguments. `apply` does the same but takes an array of arguments.

**Cross-question:** Bind an arrow?

**Cross-answer:**

It does not change `this`. Arrows take `this` from where they are written.

---

## Q6. Promise states?

**Answer:**

A promise starts as pending, then becomes fulfilled or rejected. `.then` returns a new promise. If I forget to return or throw inside it, I end up swallowing bugs.

**Cross-question:** `all` vs `allSettled` vs `race` vs `any`?

**Cross-answer:**

`all`: if one fails, the whole thing fails.

`allSettled`: waits for every one to finish.

`race`: the first one to finish, success or fail.

`any`: the first success.

---

## Q7. `async`/`await` vs `.then`?

**Answer:**

It is the same thing underneath. `await` is just easier to read, and `try/catch` works with `await`. One thing to watch: many `await`s in a row run one after another. `Promise.all` runs them together.

**Cross-question:** `forEach(async ...)`?

**Cross-answer:**

`forEach` does not wait. I use `for...of`, or `map` + `Promise.all`.

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

Promise callbacks run before timers.

**Cross-question:** Can microtasks starve the screen?

**Cross-answer:**

Yes, if I keep queueing them forever. The browser waits until that queue is empty before it paints.

---

## Q9. Debounce vs throttle?

**Answer:**

Debounce: wait until the user pauses, like in a search box.

Throttle: run at most once every X ms, like on scroll.

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

In Angular, I clear the timer when the component is destroyed.

---

## Q10. Map vs Set vs WeakMap?

**Answer:**

Map: a key can be anything. Set: unique values only. WeakMap: keys must be objects, it does not keep them alive, and you cannot loop over the keys. I use WeakMap for attaching extra data to a DOM node, not for a cache keyed by string ids.

**Cross-question:** Cache API results in WeakMap?

**Cross-answer:**

The keys must be objects. For string ids I use a Map with a size limit.

---

## Q11. Shallow copy traps?

**Answer:**

`{ ...obj }` still shares the nested objects. So changing `copy.address.city` also changes the original. It also breaks OnPush change detection if I thought I had a new object.

**Cross-question:** Fix?

**Cross-answer:**

I copy the nested part too, or use a structured clone when I really need a deep copy.

---

## Q12. `?.` `??` `??=`?

**Answer:**

`?.` is optional chaining. `??` gives a default only for null or undefined. `??=` assigns only if the value is null or undefined, so `0` stays.

**Cross-question:** `||` for pageSize?

**Cross-answer:**

`0 || 10` gives 10, which is wrong. Use `??`.

---

## Q13. Iterators and generators?

**Answer:**

`for...of` uses iterators. `function*` with `yield` creates one. Observables push values to you, while generators let you pull values. For UI events, RxJS is a better fit.

**Cross-question:** `for...in` on arrays?

**Cross-answer:**

It gives you the indexes as strings, plus any extra keys. I do not use it for arrays.

---

## Q14. Event bubbling?

**Answer:**

An event first travels down (capture), then back up (bubble). `preventDefault` stops the browser's default action, like a form submit. `stopPropagation` stops handlers on the parents from running.

**Cross-question:** Row click vs button click?

**Cross-answer:**

The button click bubbles up to the row. If the row should not also fire, I stop propagation.

---

## Q15. CORS from the Angular side?

**Answer:**

I cannot really “fix CORS” in Angular, except with a **dev proxy**. The API has to send the right headers. Postman works because it is not a browser.

**Cross-question:** Is CORS auth?

**Cross-answer:**

No. CORS is a browser safety lock. The API still needs its own login.

---

## Q16. `bind` in a React/Angular template?

**Answer:**

`bind` in a template creates a new function on every check. I bind once in the class, or use an arrow field.

**Cross-question:** Arrow as a class field?

**Cross-answer:**

It keeps `this`. It costs one function per instance, which is fine for UI handlers.

---

## Q17. Deep equality of objects?

**Answer:**

`===` does not compare deeply. I compare ids, or use JSON stringify if I accept its limits, or a helper function. In tests, I use a matcher. In Angular, I often replace the object instead of deep-watching it.

**Cross-question:** Lodash `isEqual` in a hot list?

**Cross-answer:**

It can be slow there. I prefer comparing id + version.

---

## Q18. `queueMicrotask`?

**Answer:**

It schedules a function as a microtask, the same kind as `then`. It is useful to run something after the current stack, but before paint.

**Cross-question:** vs `setTimeout(0)`?

**Cross-answer:**

The microtask runs sooner. A timeout is a normal task.

---

## Q19. Error boundaries? (JS)

**Answer:**

`window.onerror` and `unhandledrejection` catch whatever slips through. In Angular I also use `ErrorHandler`. I still use try/catch near risky I/O.

**Cross-question:** Swallow in empty catch?

**Cross-answer:**

Never. At the very least, I log it with some context.

---

## Q20. `Object.freeze` vs `const`?

**Answer:**

`const` means you cannot reassign the variable. `freeze` means you cannot add or change keys, but only at the top level. Nested objects are not frozen.

**Cross-question:** Frozen in Angular state?

**Cross-answer:**

It is nice for discipline. But signals plus new objects is the usual approach.

---

## Q21. Module cycles?

**Answer:**

That is when A imports B, and B imports A. You can end up reading a `let` that is still in the TDZ. I move the shared parts into a third file.

**Cross-question:** `import type` in TS?

**Cross-answer:**

It is erased at compile time. So types alone do not create a runtime cycle.

---

## Q22. `new Promise` executor runs when?

**Answer:**

Right away, when you construct the promise. The resolve/reject functions are for later. I do not wrap an existing promise in `new Promise` without a reason (that is the “promise constructor anti-pattern”).

**Cross-question:** Convert callback API?

**Cross-answer:**

Yes, that is what `new Promise` is for, wrapping `fs.readFile(cb)` style APIs. Not for wrapping `fetch`.

---

## Q23. `Array.from` vs spread?

**Answer:**

Both turn array-like things into a real array. `Array.from` can also map in the same step. Spread needs something iterable.

**Cross-question:** `NodeList`?

**Cross-answer:**

What `querySelectorAll` returns is not a real array. I use `Array.from` or spread to get `.map`.

---

## Q24. Strict mode?

**Answer:**

Modules and classes are always strict. Assigning to an undeclared variable throws. `this` is not `window` in a plain function call. And `with` is banned.

**Cross-question:** Script became a module and `this` broke?

**Cross-answer:**

Yes, because modules are strict. If I need the global, I use `window` on purpose.

---

## Q25. How do you avoid callback hell without async/await?

**Answer:**

With promise chains. They are still better than nested callbacks. Today I use async/await. Either way, I return the promise so callers can wait on it.


---
## Additional Questions
### Q26. `Promise.all` vs `Promise.allSettled`?
**Answer:** Promise.all fails as soon as one promise rejects. allSettled waits for every promise and gives me each result, so it is useful for independent dashboard calls.
### Q27. Debounce vs throttle?
**Answer:** Debounce waits until the activity stops, which is good for search. Throttle limits how often something runs, which is good for scroll or resize events.
### Q28. What is the nullish coalescing operator?
**Answer:** `value ?? fallback` uses the fallback only for null or undefined. Unlike `||`, it does not replace valid values like 0 or false.
