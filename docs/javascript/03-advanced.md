# JavaScript — Advanced

Engine, memory, security. Short words, still correct.

---

## Q1. How does the engine run JS?

**Answer:**

The engine parses the code, turns it into bytecode, and interprets it. Then it JIT-compiles the hot functions, meaning it compiles the code that runs a lot into fast machine code while the app is running. If you change object shapes, it may de-optimize that code. Stable shapes help. In a CRUD app, I do not micro-manage this.

**Cross-question:** Hidden class / shape?

**Cross-answer:**

Objects with the same fields in the same order share a shape. If you add fields in random order on a hot path, it can slow things down.

---

## Q2. Memory and leaks in JS?

**Answer:**

The stack is for function calls, the heap is for objects, and the garbage collector (GC) cleans up what is no longer used. Common leaks are globals, caches with no limit, listeners, detached DOM nodes and closures.

**Cross-question:** How do you find a leak?

**Cross-answer:**

I take a heap snapshot in Chrome, compare before and after a navigation, and look for arrays that keep growing and detached nodes.

---

## Q3. Browser event loop vs Node?

**Answer:**

The idea is the same: a call stack, microtasks and tasks. The browser also paints and has `requestAnimationFrame`. Node has some extra phases. For an Angular job, I focus on the browser.

**Cross-question:** `rAF` vs `setTimeout` for animation?

**Cross-answer:**

`rAF` follows the screen's frames and slows down in background tabs. Timers drift.

---

## Q4. Workers — is JS multi-threaded then?

**Answer:**

Workers are extra threads, but with **no DOM** access. They talk to the main thread with messages. `async` does not add threads. Heavy CPU work can go to a worker or to WASM.

**Cross-question:** Angular in a worker?

**Cross-answer:**

Not components. Maybe pure TS logic. The UI stays on the main thread.

---

## Q5. `structuredClone` vs JSON?

**Answer:**

`structuredClone` handles Date, Map and ArrayBuffer. It fails on functions and DOM nodes. Workers use the same mechanism. You can transfer an ArrayBuffer to avoid copying something like 50 MB.

**Cross-question:** Transfer vs copy?

**Cross-answer:**

With transfer, the original buffer is empty afterwards. With copy, you have two copies in memory.

---

## Q6. Proxy?

**Answer:**

A Proxy lets you intercept get and set on an object. Vue 3 uses this. It is slower than a plain object, and some built-ins ignore the traps. So I do not proxy everything.

**Cross-question:** Proxy a Map?

**Cross-answer:**

It is easy to get wrong because of internal slots. I wrap it instead of using a naive proxy.

---

## Q7. `WeakRef`?

**Answer:**

It is a reference that does not keep the object alive. Finalizers run “sometime”, with no promise of when. So I do not put business logic there. I prefer an explicit dispose.

**Cross-question:** Perfect cache?

**Cross-answer:**

No. You cannot know when GC will run. You still need a max size.

---

## Q8. Tail call / recursion?

**Answer:**

Most engines do not guarantee that a tail call clears the stack. So deep recursion throws. For big trees I use a loop.

**Cross-question:** 10 nodes vs 10,000?

**Cross-answer:**

For a small tree, recursion is fine. For a huge one, I go iterative.

---

## Q9. XSS and prototype pollution?

**Answer:**

XSS is when an attacker's script runs on my origin. Common sources are `innerHTML`, `eval` and `javascript:` URLs.

Prototype pollution is when merging an object with `__proto__` ends up changing `Object.prototype`. I use `Object.create(null)` or a safe merge.

**Cross-question:** Is `JSON.parse` enough?

**Cross-answer:**

It does not run code. But if I blindly merge the result, I can still pollute the prototype. So I validate the shape.

---

## Q10. Why no `eval`?

**Answer:**

It runs a string as code. That opens the door to XSS, and it is slower. CSP often blocks it anyway. Instead I use `obj[key]` with an allow-list.

**Cross-question:** Dynamic keys?

**Cross-answer:**

Use an allow-list. Never `eval('obj.' + key)`.

---

## Q11. Class fields vs methods vs arrow fields?

**Answer:**

Methods live on the prototype, so they are shared. Arrow fields are created per instance, and their `this` is fixed. I use arrows for callbacks that I pass around.

**Cross-question:** Passing `this.method` to a child?

**Cross-answer:**

It can lose `this`. Use an arrow or bind.

---

## Q12. Numbers and big ids from .NET?

**Answer:**

JS numbers are doubles, so `long` ids can lose precision. The API should send big ids as **strings**. For money, use integer cents or a decimal library, not plain floats like `0.1 + 0.2`.

**Cross-question:** `0.1 + 0.2`?

**Cross-answer:**

It is not exactly `0.3`. So never compare money with `===` on floats.

---

## Q13. Design a tiny Observable?

**Answer:**

`subscribe` adds a listener and returns an `unsubscribe` function. When I call `next`, I loop over a copy of the listener list, so unsubscribing in the middle of a notify is safe. Cold vs hot is a design choice.

**Cross-question:** vs an array of callbacks?

**Cross-answer:**

The difference is the contract: unsubscribe, error and complete. A plain array of callbacks usually leaks.

---

## Q14. Strict mode details that bite?

**Answer:**

Things that failed silently now throw. `this` is undefined in a plain call. Duplicate parameter names are an error.

**Cross-question:** `'use strict'` in a module?

**Cross-answer:**

Modules are already strict. The pragma is not needed.

---

## Q15. Memory barrier / Atomics?

**Answer:**

SharedArrayBuffer plus Atomics is for real threads. It is rare in Angular CRUD apps. I only bring it up if they ask about workers and shared memory.

**Cross-question:** Race on a normal object from two workers?

**Cross-answer:**

Workers do not share objects unless you use shared buffers. You pass clones or transfers.

---

## Q16. How would you debug a “works in Chrome, fails in old Edge”?

**Answer:**

I check which language features we use against the target in `tsconfig`, and whether we need polyfills. I look at the actual error. I do not just guess “it is the cache”.

**Cross-question:** Optional chaining in an unsupported browser?

**Cross-answer:**

It is a syntax error, so the whole file dies. The bundler must transpile for that target, or we drop that browser.

---

## Q17. Event loop and rendering?

**Answer:**

After tasks run, the browser may do style, layout and paint. Heavy JS blocks paint. So I break up huge loops, use workers, or use `requestIdleCallback` for work that is not urgent.

**Cross-question:** Layout thrash?

**Cross-answer:**

That is when you read layout, write to the DOM, read layout, write… in a loop. The browser has to recalculate again and again. The fix is to batch all the reads, then do the writes.

---

## Q18. `with` statement?

**Answer:**

Do not use it. It makes scope unclear and kills optimizations.

**Cross-question:** Still in some old code?

**Cross-answer:**

I rewrite it when I touch that file.

---

## Q19. How does `this` work in event listeners?

**Answer:**

With a normal function, `this` is the element. With an arrow, it is the outer `this`. Angular templates call handlers with the component as `this`. I do not mix these without thinking.

**Cross-question:** `addEventListener` and remove?

**Cross-answer:**

I must pass the **same** function reference to `removeEventListener`. An inline arrow cannot be removed later.

---

## Q20. Module loading in the browser?

**Answer:**

A `type="module"` script is deferred by default, runs in strict mode, and CORS applies to the file. Classic scripts behave differently. Bundlers hide all this, but I still know it for a tiny page without Angular.

**Cross-question:** Import maps?

**Cross-answer:**

The browser can map `'rxjs'` to a URL. That is handy without a bundler. Most Angular apps still bundle.

---

## Q21. What is a thenable?

**Answer:**

An object with a `then` function. `await` will treat it like a promise. That is how some libraries hook in. The catch is that any ordinary object with a `then` method gets treated the same way, which can be confusing. A `then` that is not a function, like `then: true`, is ignored.

**Cross-question:** Danger?

**Cross-answer:**

Plain JSON from a user can't contain functions, so that part is safe. The real risk is a polluted prototype: if someone manages to add a `then` function to `Object.prototype`, `await` will call it. Another reason not to blindly await unknown objects.

---

## Q22. GC types in one line?

**Answer:**

Modern engines use generational GC: young objects die fast, and old ones are scanned less often. I cannot call `gc()` in production JS. I just drop references.

**Cross-question:** `delete obj.x` vs set null?

**Cross-answer:**

Setting it to null/undefined is enough. `delete` can also change the object's shape and be slower. And for a leak, the real problem is still that something holds on to the whole object.

---

## Q23. How do you keep a SPA from XSS if you must show rich text?

**Answer:**

I use a sanitizer library with an allow-list of tags and no scripts, plus Angular's sanitizer, and I never bypass it. If the product allows it, I prefer markdown over HTML.

**Cross-question:** `innerHTML = userHtml`?

**Cross-answer:**

No.

---

## Q24. `performance.now` vs `Date.now`?

**Answer:**

`Date.now` is the wall clock, and it can jump. `performance.now` is monotonic, meaning it only moves forward, so it is good for measuring a function. I use it when I actually measure something.

**Cross-question:** Measure in production?

**Cross-answer:**

I sample. I do not `console.log` every request in a hot path.

---

## Q25. What is tail latency in the browser?

**Answer:**

It is not the average click, it is the slow 1%. It comes from things like GC pauses, huge lists, a cold cache, or a third-party script. I look at p95, not only the happy demo.

**Cross-question:** How would you improve a slow list?

**Cross-answer:**

Virtual scroll, track by id, OnPush or signals, less work per row, and fewer watchers. But I profile first, so I am not guessing.


---
## Additional Questions
### Q26. What is event delegation?
**Answer:** Instead of adding a listener to every child, I listen on the parent and check the event target. It is useful for large, dynamic lists.
### Q27. What is a microtask?
**Answer:** Promise callbacks and queueMicrotask run as microtasks. The event loop runs all the microtasks before it moves to the next task, so a huge microtask chain can still block the UI.
### Q28. Why can `JSON.stringify` be a poor deep-clone strategy?
**Answer:** It loses things like undefined, functions, Map and Set, and it changes some special values. For supported data, structuredClone is a better general tool.
