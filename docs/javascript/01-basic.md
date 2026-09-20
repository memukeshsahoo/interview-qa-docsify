# JavaScript — Basic

The browser runs JS. TypeScript is extra types on top. Interviews still ask this.

---

## Q1. JavaScript vs TypeScript?

**Answer:**

JavaScript is what the browser runs. TypeScript adds types, then those types are erased. If types are wrong or you use `any`, you still get runtime errors.

**Cross-question:** Is TypeScript a runtime?

**Cross-answer:**

No. After compile, it is JS.

---

## Q2. `var` vs `let` vs `const`?

**Answer:**

`var`: function scope, old, avoid.

`let`: block scope, can reassign.

`const`: block scope, cannot reassign the binding. The object inside can still change.

**Cross-question:** Why does `const user = {}; user.name = 'A'` work?

**Cross-answer:**

`const` locks the variable, not the object. I use `Object.freeze` only if I really need a shallow freeze.

---

## Q3. Primitives vs objects?

**Answer:**

Primitives: string, number, boolean, null, undefined, symbol, bigint. Everything else is an object, including arrays and functions. `===` on objects is “same instance”.

**Cross-question:** `typeof null`?

**Cross-answer:**

It says `'object'`. Old bug. I check `=== null`.

---

## Q4. `==` vs `===`?

**Answer:**

`==` converts types. `'0' == 0` is true. `===` does not convert. I use `===`.

**Cross-question:** `null == undefined`?

**Cross-answer:**

True with `==`. Some teams use `== null` to mean both. I still prefer to be explicit if the team likes that.

---

## Q5. Truthy and falsy?

**Answer:**

Falsy: `false`, `0`, `''`, `null`, `undefined`, `NaN`. Empty array `[]` and object `{}` are truthy.

**Cross-question:** `if (arr)` on an empty array?

**Cross-answer:**

True. Check `arr.length`.

---

## Q6. Function declaration vs arrow?

**Answer:**

`function add() {}` is hoisted. Arrows are not, and they keep `this` from outside.

**Cross-question:** When do arrows hurt?

**Cross-answer:**

As object methods that need their own `this`. As constructors. `new` is not for arrows.

---

## Q7. What is `this`?

**Answer:**

It depends on **how** you call the function.

`obj.fn()` → obj.

Plain `fn()` → `undefined` in strict mode.

`new Fn()` → new object.

Arrow → outer `this`.

**Cross-question:** Callback lost `this`?

**Cross-answer:**

I passed `obj.method` without calling it. Fix: `bind`, or `() => obj.method()`.

---

## Q8. Array methods you use daily?

**Answer:**

`map`, `filter`, `reduce`, `find`, `some`, `every`, `slice` (copy), `splice` (mutates). In Angular state I prefer copy, not mutate.

**Cross-question:** `map` vs `forEach`?

**Cross-answer:**

`map` returns a new array. `forEach` is for side effects. I do not `map` and ignore the result.

---

## Q9. Copy an object?

**Answer:**

Shallow: `{ ...obj }`. Nested objects are still shared. Deep: `structuredClone` in modern browsers. JSON copy drops dates and functions.

**Cross-question:** Destructuring rest to drop password?

**Cross-answer:**

`const { password, ...safe } = user` — then I never log `password`.

---

## Q10. Template strings and defaults?

**Answer:**

`` `Hello ${name}` ``. `function f(x = 1)`.

**Cross-question:** Tagged templates?

**Cross-answer:**

A function in front of the string. Used by some libraries. I rarely write my own.

---

## Q11. JSON pitfalls?

**Answer:**

No functions, no `undefined`, dates become strings, BigInt fails, cycles throw. Do not `eval` JSON.

**Cross-question:** Revive dates?

**Cross-answer:**

A reviver in `JSON.parse`, or map after. ISO strings are not Date objects by magic.

---

## Q12. try / catch and promises?

**Answer:**

`catch` only sees `await` or sync throws. A bare `fetch().then()` reject is unhandled unless I `.catch` or `await`.

**Cross-question:** Throw a string?

**Cross-answer:**

I throw `Error` so I get a stack.

---

## Q13. `import` vs `require`?

**Answer:**

`import`/`export` is ESM. `require` is old Node CommonJS. Browsers use ESM or a bundler.

**Cross-question:** Default vs named export?

**Cross-answer:**

Named is easier to search and rename. Teams often prefer named only.

---

## Q14. `undefined` vs `null` vs not defined?

**Answer:**

Not defined: variable never declared → `ReferenceError`.

`undefined`: declared, no value, or missing property.

`null`: I set it on purpose.

**Cross-question:** `user?.address?.city`?

**Cross-answer:**

Stops on null/undefined and returns undefined. `??` is “if null or undefined, use this”. It does not treat `0` as empty.

---

## Q15. Event loop, very simple?

**Answer:**

JS runs one thing at a time. Sync code first. Then microtasks (`Promise.then`). Then timers and events. That is why `Promise.then` can run before `setTimeout(0)`.

**Cross-question:** Is `setTimeout(fn, 0)` now?

**Cross-answer:**

No. After the current stack, and maybe later if the tab is busy.

---

## Q16. Why not `document.getElementById` in Angular?

**Answer:**

It fights Angular, breaks tests and SSR, and `innerHTML` is an XSS risk. I bind in the template or use `@ViewChild` if I must.

**Cross-question:** What is the DOM?

**Cross-answer:**

The tree of HTML nodes the browser keeps. JS can change it. Angular already does that for you.

---

## Q17. `NaN` — what is it?

**Answer:**

“Not a number”. `Number('abc')` is `NaN`. `NaN === NaN` is false. I use `Number.isNaN`.

**Cross-question:** `isNaN('hello')`?

**Cross-answer:**

Old `isNaN` converts first and says true. `Number.isNaN` is stricter. I use that.

---

## Q18. `typeof` results you should know?

**Answer:**

`'string'`, `'number'`, `'boolean'`, `'undefined'`, `'object'` (and null), `'function'`, `'bigint'`, `'symbol'`. Arrays are `'object'`. I use `Array.isArray`.

**Cross-question:** Check for a Promise?

**Cross-answer:**

Duck: `typeof x?.then === 'function'`. Not perfect, but common.

---

## Q19. Spread in function calls?

**Answer:**

`fn(...args)` is like apply. Rest in the definition: `function f(...args)`.

**Cross-question:** Spread a string?

**Cross-answer:**

`[...'ab']` is `['a','b']`. Sometimes surprising.

---

## Q20. `for...of` vs `for...in` vs `forEach`?

**Answer:**

`for...of`: values. `for...in`: keys, including inherited — bad on arrays. `forEach`: cannot `break` easily, does not wait for async.

**Cross-question:** Async inside `forEach`?

**Cross-answer:**

It will not wait. Use `for...of` with `await`.

---

## Q21. Template vs `'string' + x`?

**Answer:**

Templates are easier to read. Same idea. For HTML I still do not paste user text into HTML strings.

**Cross-question:** Performance?

**Cross-answer:**

Does not matter for normal UI. Do not build huge strings in a hot loop without care.

---

## Q22. `parseInt` vs `Number`?

**Answer:**

`parseInt('10px')` is 10. `Number('10px')` is `NaN`. `parseInt` needs a radix: `parseInt(s, 10)`.

**Cross-question:** `parseInt('08')`?

**Cross-answer:**

Always pass 10 so old octal rules cannot bite.

---

## Q23. What is hoisting in one line?

**Answer:**

Declarations are moved up in their scope. `function foo(){}` can be called early. `let`/`const` exist but throw if you touch them before the line (TDZ).

**Cross-question:** `var` hoisting?

**Cross-answer:**

You get `undefined` until the assign line. That is why `var` bugs are sneaky.

---

## Q24. Short-circuit `&&` and `||`?

**Answer:**

`a && b`: if a is truthy, result is b. `a || b`: if a is falsy, result is b. I prefer `??` for defaults so `0` stays `0`.

**Cross-question:** `value || 10` when value is 0?

**Cross-answer:**

You get 10. Wrong for page size. Use `??`.

---

## Q25. Why is JS single-threaded, but still “async”?

**Answer:**

One call stack. Slow I/O is handed off. When it finishes, a callback is queued. The page can stay responsive without extra threads. Workers exist if I need extra threads.

**Cross-question:** Does `await` make a new thread?

**Cross-answer:**

No. It just schedules the rest of the function for later.


---
## Additional Questions
### Q26. `null` vs `undefined`?
**Answer:** Both mean “no useful value”, but they have different semantics. I use null when I intentionally set a value to empty and undefined when a value/property is missing. The important thing is consistency.
### Q27. What do `map`, `filter`, and `reduce` do?
**Answer:** map transforms items, filter keeps matching items, and reduce combines items into one result such as a total or object.
### Q28. What is optional chaining?
**Answer:** `user?.profile?.name` stops and returns undefined if an earlier value is null or undefined. It avoids a lot of defensive if statements.
