# JavaScript — Basic

The browser runs JS. TypeScript adds types on top. Interviews still ask these basics.

---

## Q1. JavaScript vs TypeScript?

**Answer:**

JavaScript is what the browser actually runs. TypeScript adds types on top, and those types are erased when it compiles. So if the types are wrong, or you use `any`, you can still get runtime errors.

**Cross-question:** Is TypeScript a runtime?

**Cross-answer:**

No. After it compiles, it is just JS.

---

## Q2. `var` vs `let` vs `const`?

**Answer:**

`var`: function scope. It is the old way, so I avoid it.

`let`: block scope, and you can reassign it.

`const`: block scope, and you cannot reassign the binding. But the object it points to can still change.

**Cross-question:** Why does `const user = {}; user.name = 'A'` work?

**Cross-answer:**

`const` locks the variable, not the object. If I really need to stop changes, I use `Object.freeze`, and that is only a shallow freeze.

---

## Q3. Primitives vs objects?

**Answer:**

The primitives are string, number, boolean, null, undefined, symbol and bigint. Everything else is an object, including arrays and functions. When you use `===` on objects, it checks if they are the same instance.

**Cross-question:** `typeof null`?

**Cross-answer:**

It returns `'object'`. That is an old bug in the language. So I check `=== null` instead.

---

## Q4. `==` vs `===`?

**Answer:**

`==` converts types before comparing, so `'0' == 0` is true. `===` does not convert anything. I use `===`.

**Cross-question:** `null == undefined`?

**Cross-answer:**

That is true with `==`. Some teams use `== null` on purpose to check for both. I still prefer to be explicit, unless the team likes that style.

---

## Q5. Truthy and falsy?

**Answer:**

The falsy values are `false`, `0`, `''`, `null`, `undefined` and `NaN`. An empty array `[]` and an empty object `{}` are truthy.

**Cross-question:** `if (arr)` on an empty array?

**Cross-answer:**

It is true. So I check `arr.length` instead.

---

## Q6. Function declaration vs arrow?

**Answer:**

`function add() {}` is hoisted, so you can call it before its line. Arrows are not, and they keep the `this` from the outer scope.

**Cross-question:** When do arrows hurt?

**Cross-answer:**

When you use them as object methods that need their own `this`. Also as constructors, because you cannot use `new` with an arrow.

---

## Q7. What is `this`?

**Answer:**

It depends on **how** you call the function.

`obj.fn()` → obj.

Plain `fn()` → `undefined` in strict mode.

`new Fn()` → the new object.

Arrow → the outer `this`.

**Cross-question:** Callback lost `this`?

**Cross-answer:**

That happens when I pass `obj.method` without calling it. The fix is `bind`, or `() => obj.method()`.

---

## Q8. Array methods you use daily?

**Answer:**

`map`, `filter`, `reduce`, `find`, `some`, `every`, `slice` (makes a copy) and `splice` (changes the array). For Angular state, I prefer copying over mutating.

**Cross-question:** `map` vs `forEach`?

**Cross-answer:**

`map` returns a new array. `forEach` is for side effects. I never use `map` and then ignore the result.

---

## Q9. Copy an object?

**Answer:**

For a shallow copy I use `{ ...obj }`, but nested objects are still shared. For a deep copy I use `structuredClone` in modern browsers. A JSON copy turns dates into strings and drops functions.

**Cross-question:** Destructuring rest to drop password?

**Cross-answer:**

`const { password, ...safe } = user` — and then I never log `password`.

---

## Q10. Template strings and defaults?

**Answer:**

A template string looks like `` `Hello ${name}` ``. A default parameter looks like `function f(x = 1)`.

**Cross-question:** Tagged templates?

**Cross-answer:**

It is a function placed in front of the template string. Some libraries use them. I rarely write my own.

---

## Q11. JSON pitfalls?

**Answer:**

JSON has no functions and no `undefined`. Dates become strings, BigInt fails, and circular references throw. And never use `eval` to parse JSON.

**Cross-question:** Revive dates?

**Cross-answer:**

I use a reviver in `JSON.parse`, or map the values afterwards. ISO strings do not turn into Date objects by magic.

---

## Q12. try / catch and promises?

**Answer:**

`catch` only sees errors from `await` or from sync code that throws. A bare `fetch().then()` that rejects is unhandled unless I add `.catch` or `await` it.

**Cross-question:** Throw a string?

**Cross-answer:**

No, I throw an `Error` so I get a stack trace.

---

## Q13. `import` vs `require`?

**Answer:**

`import`/`export` is ESM, the standard module system. `require` is the older CommonJS style from Node. Browsers use ESM or a bundler.

**Cross-question:** Default vs named export?

**Cross-answer:**

Named exports are easier to search for and rename. A lot of teams use named exports only.

---

## Q14. `undefined` vs `null` vs not defined?

**Answer:**

Not defined: the variable was never declared, so you get a `ReferenceError`.

`undefined`: it was declared but has no value, or the property is missing.

`null`: I set it on purpose.

**Cross-question:** `user?.address?.city`?

**Cross-answer:**

It stops at the first null or undefined and returns undefined. `??` means “if this is null or undefined, use this other value”. It does not treat `0` as empty.

---

## Q15. Event loop, very simple?

**Answer:**

JS runs one thing at a time. Sync code runs first. Then microtasks, like `Promise.then`. Then timers and events. That is why `Promise.then` can run before `setTimeout(0)`.

**Cross-question:** Is `setTimeout(fn, 0)` now?

**Cross-answer:**

No. It runs after the current stack is done, and maybe later if the tab is busy.

---

## Q16. Why not `document.getElementById` in Angular?

**Answer:**

It works against Angular, it breaks tests and SSR, and using `innerHTML` is an XSS risk. I bind in the template, or use `@ViewChild` if I really have to.

**Cross-question:** What is the DOM?

**Cross-answer:**

It is the tree of HTML nodes the browser keeps in memory. JS can change it. Angular already does that for you.

---

## Q17. `NaN` — what is it?

**Answer:**

It means “Not a Number”. `Number('abc')` gives `NaN`. Also, `NaN === NaN` is false. So I check with `Number.isNaN`.

**Cross-question:** `isNaN('hello')`?

**Cross-answer:**

The old `isNaN` converts the value first, so it says true. `Number.isNaN` is stricter, so I use that.

---

## Q18. `typeof` results you should know?

**Answer:**

`'string'`, `'number'`, `'boolean'`, `'undefined'`, `'object'` (null gives this too), `'function'`, `'bigint'` and `'symbol'`. Arrays also give `'object'`, so I use `Array.isArray` for them.

**Cross-question:** Check for a Promise?

**Cross-answer:**

I use duck typing: `typeof x?.then === 'function'`. It is not perfect, but it is common.

---

## Q19. Spread in function calls?

**Answer:**

`fn(...args)` works like `apply`. Rest is the same syntax in the definition: `function f(...args)`.

**Cross-question:** Spread a string?

**Cross-answer:**

`[...'ab']` gives `['a','b']`. That can be surprising.

---

## Q20. `for...of` vs `for...in` vs `forEach`?

**Answer:**

`for...of` gives you values. `for...in` gives you keys, including inherited ones, so it is bad for arrays. `forEach` cannot `break` easily, and it does not wait for async code.

**Cross-question:** Async inside `forEach`?

**Cross-answer:**

It will not wait. I use `for...of` with `await` instead.

---

## Q21. Template vs `'string' + x`?

**Answer:**

Templates are easier to read. It is the same idea underneath. For HTML, I still never paste user text into HTML strings.

**Cross-question:** Performance?

**Cross-answer:**

For normal UI it does not matter. Just be careful when building huge strings in a hot loop.

---

## Q22. `parseInt` vs `Number`?

**Answer:**

`parseInt('10px')` gives 10. `Number('10px')` gives `NaN`. With `parseInt`, always pass the radix: `parseInt(s, 10)`.

**Cross-question:** `parseInt('08')`?

**Cross-answer:**

I always pass 10, so the old octal rules cannot bite.

---

## Q23. What is hoisting in one line?

**Answer:**

Declarations are moved to the top of their scope. So `function foo(){}` can be called before its line. `let`/`const` exist too, but they throw if you touch them before their line. That gap is called the TDZ (temporal dead zone).

**Cross-question:** `var` hoisting?

**Cross-answer:**

You get `undefined` until the line that assigns it. That is why `var` bugs are sneaky.

---

## Q24. Short-circuit `&&` and `||`?

**Answer:**

`a && b`: if a is truthy, the result is b. `a || b`: if a is falsy, the result is b. For defaults I prefer `??`, so `0` stays `0`.

**Cross-question:** `value || 10` when value is 0?

**Cross-answer:**

You get 10. That is wrong for something like page size. Use `??`.

---

## Q25. Why is JS single-threaded, but still “async”?

**Answer:**

There is one call stack. Slow I/O is handed off to the browser or runtime. When it finishes, a callback is queued. So the page stays responsive without extra threads. If I do need extra threads, there are Workers.

**Cross-question:** Does `await` make a new thread?

**Cross-answer:**

No. It just schedules the rest of the function to run later.


---
## Additional Questions
### Q26. `null` vs `undefined`?
**Answer:** Both mean “no useful value”, but they mean slightly different things. I use null when I set a value to empty on purpose, and undefined when a value or property is missing. The main thing is to be consistent.
### Q27. What do `map`, `filter`, and `reduce` do?
**Answer:** map transforms each item, filter keeps the items that match, and reduce combines the items into one result, like a total or an object.
### Q28. What is optional chaining?
**Answer:** `user?.profile?.name` stops and returns undefined if an earlier value is null or undefined. It saves me from writing a lot of defensive if statements.
