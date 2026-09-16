# JavaScript — Advanced

Engine, memory, security. Short words, still correct.

---

## Q1. How does the engine run JS?

**Answer:**

Parse, bytecode, interpret, then JIT-compile hot functions. If you change object shapes, it may de-optimize. Stable shapes help. I do not micro-manage this in a CRUD app.

**Cross-question:** Hidden class / shape?

**Cross-answer:**

Objects with the same fields in the same order share a shape. Adding fields in random order on a hot path can slow things down.

---

## Q2. Memory and leaks in JS?

**Answer:**

Stack for calls, heap for objects, GC cleans. Leaks: globals, caches with no limit, listeners, detached DOM, closures.

**Cross-question:** How do you find a leak?

**Cross-answer:**

Chrome heap snapshot, compare before and after a navigation, look for growing arrays and detached nodes.

---

## Q3. Browser event loop vs Node?

**Answer:**

Same idea: stack, microtasks, tasks. Browser also paints and uses `requestAnimationFrame`. Node has extra phases. For an Angular job I focus on the browser.

**Cross-question:** `rAF` vs `setTimeout` for animation?

**Cross-answer:**

`rAF` follows frames and calms down in background tabs. Timers drift.

---

## Q4. Workers — is JS multi-threaded then?

**Answer:**

Workers are extra threads with **no DOM**. They talk with messages. `async` does not add threads. Heavy CPU can go to a worker or WASM.

**Cross-question:** Angular in a worker?

**Cross-answer:**

Not components. Maybe pure TS logic. UI stays on the main thread.

---

## Q5. `structuredClone` vs JSON?

**Answer:**

Clone handles Date, Map, ArrayBuffer. Fails on functions and DOM nodes. Workers use this. Transfer an ArrayBuffer to avoid copying 50 MB.

**Cross-question:** Transfer vs copy?

**Cross-answer:**

Transfer: original buffer is empty after. Copy: two memories.

---

## Q6. Proxy?

**Answer:**

Intercept get/set. Vue 3 uses this. Slower than a plain object. Some built-ins ignore traps. I do not proxy everything.

**Cross-question:** Proxy a Map?

**Cross-answer:**

Easy to get wrong because of internal slots. Wrap instead of a naive proxy.

---

## Q7. `WeakRef`?

**Answer:**

A pointer that does not keep the object alive. Finalizers run “sometime”. I do not put business logic there. Prefer explicit dispose.

**Cross-question:** Perfect cache?

**Cross-answer:**

No. You cannot know when GC runs. Still need a max size.

---

## Q8. Tail call / recursion?

**Answer:**

Most engines do not guarantee tail-call wipe of the stack. Deep recursion throws. I use a loop for big trees.

**Cross-question:** 10 nodes vs 10,000?

**Cross-answer:**

Small: recursion is fine. Huge: iterative.

---

## Q9. XSS and prototype pollution?

**Answer:**

XSS: attacker script on my origin. Sources: `innerHTML`, `eval`, `javascript:` URLs.

Pollution: merge `__proto__` into `Object.prototype`. Use `Object.create(null)` or a safe merge.

**Cross-question:** Is `JSON.parse` enough?

**Cross-answer:**

It does not run code. If I blindly merge the object, I can still pollute. Validate shape.

---

## Q10. Why no `eval`?

**Answer:**

Runs a string as code. XSS and slower. CSP often blocks it. I use `obj[key]` with an allow-list.

**Cross-question:** Dynamic keys?

**Cross-answer:**

Allow-list. Never `eval('obj.' + key)`.

---

## Q11. Class fields vs methods vs arrow fields?

**Answer:**

Methods on the prototype: shared. Arrow fields: per instance, `this` is fixed. I use arrows for callbacks I pass around.

**Cross-question:** Passing `this.method` to a child?

**Cross-answer:**

Can lose `this`. Arrow or bind.

---

## Q12. Numbers and big ids from .NET?

**Answer:**

JS numbers are doubles. `long` ids can lose precision. The API should send big ids as **strings**. Money: integer cents or a decimal library, not `0.1 + 0.2`.

**Cross-question:** `0.1 + 0.2`?

**Cross-answer:**

Not `0.3` exactly. Never compare money with `===` on floats.

---

## Q13. Design a tiny Observable?

**Answer:**

`subscribe` adds a listener, returns `unsubscribe`. When I `next`, I loop a copy of the list so unsubscribe mid-notify is safe. Cold vs hot is a choice.

**Cross-question:** vs an array of callbacks?

**Cross-answer:**

The contract: unsubscribe, error, complete. Arrays of callbacks usually leak.

---

## Q14. Strict mode details that bite?

**Answer:**

Silent failures become throws. `this` is undefined. Duplicate params error.

**Cross-question:** `'use strict'` in a module?

**Cross-answer:**

Modules already are. The pragma is extra.

---

## Q15. Memory barrier / Atomics?

**Answer:**

SharedArrayBuffer plus Atomics for real threads. Rare in Angular CRUD. I mention it only if they ask about workers and shared memory.

**Cross-question:** Race on a normal object from two workers?

**Cross-answer:**

Workers do not share objects unless you use shared buffers. You pass clones or transfers.

---

## Q16. How would you debug a “works in Chrome, fails in old Edge”?

**Answer:**

Check language features vs the target in `tsconfig`. Polyfills. Look at the actual error. Do not guess “cache”.

**Cross-question:** Optional chaining in an unsupported browser?

**Cross-answer:**

Syntax error, whole file dies. The bundler must transpile for that target, or we drop the browser.

---

## Q17. Event loop and rendering?

**Answer:**

After tasks, the browser may style, layout, paint. Heavy JS blocks paint. I break huge loops, use workers, or `requestIdleCallback` for non-urgent work.

**Cross-question:** Layout thrash?

**Cross-answer:**

Read layout, write DOM, read layout, write… in a loop. The browser recalculates again and again. Batch reads then writes.

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

A normal function: `this` is the element. Arrow: outer `this`. Angular templates call with the component. I do not mix these without thinking.

**Cross-question:** `addEventListener` and remove?

**Cross-answer:**

I must pass the **same** function reference to `removeEventListener`. An inline arrow cannot be removed later.

---

## Q20. Module loading in the browser?

**Answer:**

`type="module"` is deferred by default, strict, and CORS applies to the file. Classic scripts are different. Bundlers hide this. I still know it for a tiny page without Angular.

**Cross-question:** Import maps?

**Cross-answer:**

Browser can map `'rxjs'` to a URL. Handy without a bundler. Most Angular apps still bundle.

---

## Q21. What is a thenable?

**Answer:**

An object with a `then` function. `await` will treat it like a promise. That is how some libraries hook in. A random object with `then: true` can confuse `await`.

**Cross-question:** Danger?

**Cross-answer:**

If user JSON has a `then` function (rare) or a polluted prototype, `await` might call it. Another reason not to blindly await unknown objects.

---

## Q22. GC types in one line?

**Answer:**

Modern engines use generational GC: young objects die fast, old ones are scanned less often. I cannot call `gc()` in production JS. I just drop references.

**Cross-question:** `delete obj.x` vs set null?

**Cross-answer:**

Setting null/undefined is enough. `delete` can also change the shape and be slower. For a leak, the real issue is still holding the whole object.

---

## Q23. How do you keep a SPA from XSS if you must show rich text?

**Answer:**

A sanitizer library, allow-list of tags, no scripts, Angular sanitizer, never bypass. Prefer markdown to HTML if the product allows it.

**Cross-question:** `innerHTML = userHtml`?

**Cross-answer:**

No.

---

## Q24. `performance.now` vs `Date.now`?

**Answer:**

`Date.now` is wall clock, can jump. `performance.now` is monotonic, good for measuring a function. I use it when I actually measure.

**Cross-question:** Measure in production?

**Cross-answer:**

Sample. Do not `console.log` every request in a hot path.

---

## Q25. What is tail latency in the browser?

**Answer:**

Not the average click, the slow 1%. Caused by GC pauses, huge lists, a cold cache, a third-party script. I look at p95, not only the happy demo.

**Cross-question:** How would you improve a slow list?

**Cross-answer:**

Virtual scroll, track by id, OnPush/signals, less work per row, fewer watchers. Profile first so I do not guess.
