# JavaScript — Advanced

Engine, memory, security. Short words, still correct.

---

## Q1. How does the engine run JS?

**Answer:**

The engine is the program inside the browser (or Node) that actually runs our JavaScript. Chrome's engine is called V8. It does not run our text line by line as we wrote it. It works in steps.

The engine parses the code, turns it into bytecode, and interprets it. Parsing means reading the text and understanding its structure. Bytecode is a simpler, compact set of instructions that is quick to start running. Then it JIT-compiles the hot functions, meaning it compiles the code that runs a lot into fast machine code while the app is running. "JIT" stands for "just in time": it compiles only when it sees the code is worth it, like a cook who starts preparing a dish in bulk only after noticing that many people keep ordering it.

If you change object shapes, it may de-optimize that code. The fast version was built on guesses like "this object always has these fields". When the guess turns out wrong, the engine throws away the fast version and goes back to the slower path. Stable shapes help. In a CRUD app, I do not micro-manage this. Network calls and rendering cost far more than these engine details, so I only think about it when a profiler points at a hot function.

**Cross-question:** Hidden class / shape?

**Cross-answer:**

A shape (also called a hidden class) is the engine's internal note about which fields an object has and in what order. Objects with the same fields in the same order share a shape, and then the engine can find a field quickly, like knowing the third drawer always holds the spoons. If you add fields in random order on a hot path, it can slow things down, because the engine ends up with many different shapes and loses its shortcut. The simple habit is to create objects with all their fields set up front, in the same order.

---

## Q2. Memory and leaks in JS?

**Answer:**

The stack is for function calls, the heap is for objects, and the garbage collector (GC) cleans up what is no longer used. The stack is like a pile of plates: each function call puts a plate on top, and it comes off when the function finishes. The heap is a big storage room where objects live. The GC walks through that room now and then and throws away anything nothing points to any more.

A leak happens when we keep pointing at things we no longer need. The GC cannot remove them, so memory keeps growing. The page gets slower and slower, and on a long-running dashboard it can eventually crash the tab.

Common leaks are globals, caches with no limit, listeners, detached DOM nodes and closures. For example, a global array that we push into on every page visit never gets cleaned. A listener added on `window` and never removed keeps its whole component alive. A detached DOM node is an element removed from the page but still held in a variable. A closure is a function that remembers variables from where it was created, so a long-lived callback can quietly hold on to big data.

**Cross-question:** How do you find a leak?

**Cross-answer:**

I take a heap snapshot in Chrome, compare before and after a navigation, and look for arrays that keep growing and detached nodes. A heap snapshot is a picture of everything in memory at that moment. In practice I open a page, go away, come back a few times, and take snapshots in between. If memory keeps climbing and the same objects keep piling up, the snapshot's "retainers" view shows what is still holding on to them.

---

## Q3. Browser event loop vs Node?

**Answer:**

The event loop is how JavaScript does many things with a single thread. Think of one cashier with a queue: they serve one person at a time, and anything that has to wait (a timer, a network reply) goes back in the queue until it is ready.

The idea is the same: a call stack, microtasks and tasks. The call stack is the code running right now. Microtasks are small follow-up jobs like promise callbacks, and they run as soon as the current code finishes. Tasks are bigger jobs like timers and click events, and they run one at a time after that. The browser also paints and has `requestAnimationFrame`, because it has to draw the screen between tasks. Node has no screen, but it has some extra phases, for things like file and network callbacks and `setImmediate`. For an Angular job, I focus on the browser.

**Cross-question:** `rAF` vs `setTimeout` for animation?

**Cross-answer:**

`rAF` follows the screen's frames and slows down in background tabs. So my animation code runs right before each repaint, usually about 60 times a second, and stops wasting battery when no one is looking. Timers drift. A `setTimeout` of 16 ms is only a "not before" time. It can fire late, or between two frames, which gives jumpy movement.

---

## Q4. Workers — is JS multi-threaded then?

**Answer:**

Normally all our page's JavaScript runs on one thread, the main thread, which also handles clicks and drawing. If we do a heavy calculation there, the page freezes until it is done.

Workers are extra threads, but with **no DOM** access. They talk to the main thread with messages. So a worker is like a helper in the back room: you hand them a job through a window, they work on it, and they hand the result back, but they cannot touch the shop floor (the page) themselves.

`async` does not add threads. An `async` function still runs on the main thread. It just lets the main thread do other things while it waits for something like a network reply. Heavy CPU work can go to a worker or to WASM. WASM (WebAssembly) is a fast, low-level format that runs code compiled from languages like C or Rust. For example, parsing a huge CSV file or resizing images is a good fit for a worker, so the UI stays smooth.

**Cross-question:** Angular in a worker?

**Cross-answer:**

Not components. Components need the DOM, and a worker does not have it. Maybe pure TS logic, like heavy filtering, sorting or calculations that only take data in and give data back. The UI stays on the main thread.

---

## Q5. `structuredClone` vs JSON?

**Answer:**

Both are ways to make a deep copy of data, so changing the copy does not change the original. The old trick was `JSON.parse(JSON.stringify(obj))`, which turns the object into text and back. That loses anything JSON does not understand, for example a Date comes back as a plain string.

`structuredClone` handles Date, Map and ArrayBuffer. It fails on functions and DOM nodes, because those cannot be copied as plain data. Workers use the same mechanism. When we send data to a worker with `postMessage`, the browser copies it with this same "structured clone" method. You can transfer an ArrayBuffer to avoid copying something like 50 MB. Transfer means handing the memory over instead of making a second copy, which is much faster for big data.

**Cross-question:** Transfer vs copy?

**Cross-answer:**

With transfer, the original buffer is empty afterwards. It is like handing someone your suitcase: they have it now, and you do not. With copy, you have two copies in memory. That is safer when both sides still need the data, but it costs time and double the memory.

---

## Q6. Proxy?

**Answer:**

A Proxy is a wrapper around an object that can watch and control what happens to it. A Proxy lets you intercept get and set on an object. So every time code reads or writes a property, my own function (called a "trap") runs first. It is like a receptionist who sees everyone going in and out of an office and can log them or stop them.

Vue 3 uses this. That is how Vue knows which data changed and which part of the screen to update. It is also handy for validation or logging.

It is slower than a plain object, and some built-ins ignore the traps. Every property access now goes through an extra function, and some built-in objects work in ways the traps do not see. So I do not proxy everything. I use it only where the extra control is really needed.

**Cross-question:** Proxy a Map?

**Cross-answer:**

It is easy to get wrong because of internal slots. A Map keeps its data in hidden internal storage, not in normal properties. Methods like `get` and `set` expect `this` to be the real Map, so calling them through a proxy throws an error. I wrap it instead of using a naive proxy, for example a small class that holds a real Map inside and exposes the methods I need.

---

## Q7. `WeakRef`?

**Answer:**

It is a reference that does not keep the object alive. A normal variable pointing at an object tells the GC "I still need this". A `WeakRef` points at the object without making that promise. If nothing else uses the object, the GC is free to remove it, and the `WeakRef` then gives back `undefined`.

Finalizers run “sometime”, with no promise of when. A finalizer is a callback (registered with `FinalizationRegistry`) that runs after an object has been cleaned up. It may run late, or in some cases not at all. So I do not put business logic there, like saving data or closing a connection. I prefer an explicit dispose, meaning a method like `close()` or `destroy()` that I call myself at the right time.

**Cross-question:** Perfect cache?

**Cross-answer:**

No. You cannot know when GC will run. It might clear the cached items too early, or keep them for a long time and let memory grow. You still need a max size, so the cache stays under control whatever the GC does.

---

## Q8. Tail call / recursion?

**Answer:**

Recursion is when a function calls itself, for example to walk through a tree of folders. Each call adds a frame to the call stack, and the stack has a limited size. A tail call is when the last thing a function does is call another function. In theory the engine could reuse the current frame for it, so the stack would not grow.

Most engines do not guarantee that a tail call clears the stack. The rule exists in the spec, but in practice only Safari does it. So deep recursion throws, usually "Maximum call stack size exceeded". For big trees I use a loop. I keep my own list of nodes still to visit, so the depth no longer depends on the call stack.

**Cross-question:** 10 nodes vs 10,000?

**Cross-answer:**

For a small tree, recursion is fine. It is short and easy to read, and 10 levels are nowhere near the limit. For a huge one, I go iterative, with a loop and my own stack or queue, because a very deep tree could blow the call stack.

---

## Q9. XSS and prototype pollution?

**Answer:**

XSS is when an attacker's script runs on my origin. "Origin" means my site's address. If their script runs there, it can do anything my own code can: read data on the page, send requests as the logged-in user, or change what the user sees. Common sources are `innerHTML`, `eval` and `javascript:` URLs. For example, if I put a user's comment into the page with `innerHTML` and the comment contains an `<img onerror=...>` tag, that code runs for everyone who opens the page.

Prototype pollution is when merging an object with `__proto__` ends up changing `Object.prototype`. Almost every object in JavaScript inherits from `Object.prototype`, so if an attacker adds a property there, it suddenly shows up on all objects. A check like `if (user.isAdmin)` could become true for everyone. I use `Object.create(null)` or a safe merge. `Object.create(null)` makes an object with no prototype at all, and a safe merge skips keys like `__proto__`, `constructor` and `prototype`.

**Cross-question:** Is `JSON.parse` enough?

**Cross-answer:**

It does not run code. But if I blindly merge the result, I can still pollute the prototype. `JSON.parse` happily creates an object with a `__proto__` key in it, and a careless deep merge then copies that key onto the real prototype. So I validate the shape, meaning I check that the data has only the fields and types I expect before I use it.

---

## Q10. Why no `eval`?

**Answer:**

It runs a string as code. So whatever text ends up in that string gets run with full access to my page. If any part of it comes from a user or a URL, an attacker can put their own code there. That opens the door to XSS, and it is slower, because the engine cannot optimize code it only sees at run time. CSP often blocks it anyway. CSP (Content Security Policy) is a set of browser rules the site sends, and a typical policy does not allow `eval` at all.

Instead I use `obj[key]` with an allow-list. For example, if the user picks a column to sort by, I check the key against a fixed list of allowed column names and then read `obj[key]`.

**Cross-question:** Dynamic keys?

**Cross-answer:**

Use an allow-list, a fixed list of keys I accept, and reject anything else. Never `eval('obj.' + key)`, because a crafted `key` could turn that string into any code the attacker wants.

---

## Q11. Class fields vs methods vs arrow fields?

**Answer:**

Methods live on the prototype, so they are shared. If I create 1,000 objects from a class, there is still only one copy of each method, and every object uses it. Arrow fields are created per instance, and their `this` is fixed. Each object gets its own copy of the function, which costs a bit more memory, but `this` always points to that object, no matter who calls the function.

I use arrows for callbacks that I pass around, like a click handler or a callback given to a timer or a child component. There the function is called by someone else, and a normal method would lose its `this`.

**Cross-question:** Passing `this.method` to a child?

**Cross-answer:**

It can lose `this`. When the child calls the function on its own, it is no longer called as `myObj.method()`, so `this` is no longer my object and reading `this.something` fails. Use an arrow or bind. An arrow field keeps `this` fixed, and `this.method.bind(this)` makes a copy of the function with `this` locked in.

---

## Q12. Numbers and big ids from .NET?

**Answer:**

JS numbers are doubles, so `long` ids can lose precision. A double is the standard 64-bit floating-point number, and it can only store whole numbers exactly up to about 9 quadrillion (`Number.MAX_SAFE_INTEGER`). A C# `long` can go much higher. If a bigger id arrives as a number in JSON, JavaScript quietly rounds it, and I end up asking the API for the wrong record.

The API should send big ids as **strings**. A string keeps every digit, and I do not need to do math on an id anyway. For money, use integer cents or a decimal library, not plain floats like `0.1 + 0.2`. Storing ₹10.50 as `1050` paise keeps the math exact.

**Cross-question:** `0.1 + 0.2`?

**Cross-answer:**

It is not exactly `0.3`. It gives `0.30000000000000004`, because numbers like 0.1 cannot be stored exactly in binary, just like 1/3 cannot be written exactly in decimal. So never compare money with `===` on floats. A check that should be true can come out false.

---

## Q13. Design a tiny Observable?

**Answer:**

An Observable is a source of values over time that others can listen to, like a news channel people subscribe to. RxJS, which Angular uses a lot, is built around this idea.

`subscribe` adds a listener and returns an `unsubscribe` function. So whoever subscribes also gets a simple way to stop listening later. When I call `next`, I loop over a copy of the listener list, so unsubscribing in the middle of a notify is safe. If a listener removes itself while I am looping over the original list, the list shrinks under my feet and the next listener can get skipped. Looping over a copy avoids that.

Cold vs hot is a design choice. A cold Observable starts fresh for each subscriber, like a video on demand. A hot one shares one running source, like a live TV broadcast where late viewers miss what already happened.

**Cross-question:** vs an array of callbacks?

**Cross-answer:**

The difference is the contract: unsubscribe, error and complete. Everyone knows how to stop listening, how a failure is reported, and when no more values will come. A plain array of callbacks usually leaks, because there is no standard way to remove yourself, so old callbacks (and everything they hold on to) stay in the array.

---

## Q14. Strict mode details that bite?

**Answer:**

Strict mode is a stricter version of JavaScript that we turn on with `'use strict'`. It removes some old, forgiving behaviour that used to hide bugs.

Things that failed silently now throw. For example, assigning to a variable you never declared used to quietly create a global. In strict mode it is an error, so a typo in a variable name shows up straight away. `this` is undefined in a plain call. If you call a function on its own, not as `obj.fn()`, `this` is `undefined` instead of the global object, so code that relied on that breaks. Duplicate parameter names are an error, like `function f(a, a)`.

**Cross-question:** `'use strict'` in a module?

**Cross-answer:**

Modules are already strict. Any file loaded as an ES module, and that includes the code Angular and TypeScript produce, is in strict mode automatically. The pragma is not needed.

---

## Q15. Memory barrier / Atomics?

**Answer:**

Normally workers do not share memory. Each one gets its own copy of the data. A SharedArrayBuffer is a block of memory that several threads can read and write at the same time. That brings the classic problem of real threads: two threads changing the same value at once can lose updates. Atomics are operations that do a read or a write as one step that cannot be interrupted, and they let threads wait for and wake each other. A memory barrier is the guarantee that one thread's writes become visible to the other threads in the right order.

SharedArrayBuffer plus Atomics is for real threads. It is rare in Angular CRUD apps. I only bring it up if they ask about workers and shared memory.

**Cross-question:** Race on a normal object from two workers?

**Cross-answer:**

Workers do not share objects unless you use shared buffers. You pass clones or transfers. Each worker gets its own copy, or takes over the buffer, so two workers can never change the same normal object at the same time. There is no race to worry about.

---

## Q16. How would you debug a “works in Chrome, fails in old Edge”?

**Answer:**

Usually this means the old browser does not understand some newer JavaScript feature we used.

I check which language features we use against the target in `tsconfig`, and whether we need polyfills. The target setting decides which JavaScript version TypeScript outputs. A polyfill is a small piece of code that adds a missing built-in function, like `Array.prototype.includes`, to an older browser. I look at the actual error, in the old browser's console. I do not just guess “it is the cache”. The error message usually names the exact line or the missing function, which saves a lot of time.

**Cross-question:** Optional chaining in an unsupported browser?

**Cross-answer:**

It is a syntax error, so the whole file dies. A polyfill cannot fix new syntax like `?.`, because the browser cannot even read the file. The bundler must transpile for that target, meaning rewrite the code into older syntax the browser understands, or we drop that browser.

---

## Q17. Event loop and rendering?

**Answer:**

The browser uses the same main thread for our JavaScript and for drawing the page. After tasks run, the browser may do style, layout and paint. Style works out which CSS applies, layout works out the size and position of each element, and paint draws the pixels.

Heavy JS blocks paint. While a long loop is running, the screen cannot update, so the page looks frozen and clicks feel ignored. So I break up huge loops, use workers, or use `requestIdleCallback` for work that is not urgent. Breaking up a loop means processing, say, a few hundred rows, then letting the browser breathe, then continuing. `requestIdleCallback` runs code only when the browser has nothing more important to do.

**Cross-question:** Layout thrash?

**Cross-answer:**

That is when you read layout, write to the DOM, read layout, write… in a loop. Reading something like `offsetHeight` forces the browser to work out the layout right now, and each write makes that layout out of date again. The browser has to recalculate again and again. The fix is to batch all the reads, then do the writes. It is like making all your measurements first and then cutting, instead of measuring again after every cut.

---

## Q18. `with` statement?

**Answer:**

`with (obj) { ... }` lets you use the object's properties inside the block as if they were plain variables. Do not use it. It makes scope unclear and kills optimizations. When you read `x` inside the block, neither you nor the engine can tell from the code whether it means `obj.x` or an outer variable `x`. That only becomes clear at run time, so the engine cannot optimize it. It is also not allowed in strict mode at all, so it does not work in modules.

**Cross-question:** Still in some old code?

**Cross-answer:**

I rewrite it when I touch that file. Usually it is just a matter of writing `obj.x` explicitly, or putting the object in a short local variable.

---

## Q19. How does `this` work in event listeners?

**Answer:**

`this` means "the object this function is working for right now", and it depends on how the function is called.

With a normal function, `this` is the element. So in `button.addEventListener('click', function () { ... })`, `this` is the button that was clicked. With an arrow, it is the outer `this`. An arrow function does not get its own `this`, it uses whatever `this` was where the arrow was written, for example the class instance. Angular templates call handlers with the component as `this`, so in `(click)="save()"`, `this` inside `save` is the component. I do not mix these without thinking, because a handler that expects the element but gets the component (or the other way round) fails in confusing ways.

**Cross-question:** `addEventListener` and remove?

**Cross-answer:**

I must pass the **same** function reference to `removeEventListener`. The browser matches listeners by the function object itself, not by what the code looks like. An inline arrow cannot be removed later, because writing the same arrow again creates a new function. So I store the handler in a variable or a class field and pass that same one to both calls.

---

## Q20. Module loading in the browser?

**Answer:**

A module is a JavaScript file that uses `import` and `export` to share code with other files. A `type="module"` script is deferred by default, runs in strict mode, and CORS applies to the file. Deferred means it waits until the HTML has been parsed, so it does not block the page from showing. CORS is the browser's rule about loading files from another site: a module from another domain only loads if that server allows it, and it also means opening the page straight from disk (`file://`) usually fails.

Classic scripts behave differently. They block parsing unless you add `defer` or `async`, they are not strict by default, and their top-level variables become globals. Bundlers hide all this, but I still know it for a tiny page without Angular.

**Cross-question:** Import maps?

**Cross-answer:**

An import map is a small JSON block in the HTML page that tells the browser where to find a package by name. The browser can map `'rxjs'` to a URL. So `import { of } from 'rxjs'` works in plain browser code. That is handy without a bundler, for demos or small pages. Most Angular apps still bundle.

---

## Q21. What is a thenable?

**Answer:**

An object with a `then` function. `await` will treat it like a promise. So JavaScript does not check "is this a real Promise?". It checks "does this have a `then` method?", a bit like a club that lets in anyone wearing the right badge. That is how some libraries hook in. Older promise libraries and some query builders can be awaited this way.

The catch is that any ordinary object with a `then` method gets treated the same way, which can be confusing. If my object happens to have a method called `then`, awaiting it calls that method instead of just giving me the object back. A `then` that is not a function, like `then: true`, is ignored.

**Cross-question:** Danger?

**Cross-answer:**

Plain JSON from a user can't contain functions, so that part is safe. The real risk is a polluted prototype: if someone manages to add a `then` function to `Object.prototype`, `await` will call it. Then every object you await suddenly runs the attacker's code. Another reason not to blindly await unknown objects.

---

## Q22. GC types in one line?

**Answer:**

Modern engines use generational GC: young objects die fast, and old ones are scanned less often. Most objects are short-lived, like a temporary array inside a function. So the GC checks the "young" area often and cheaply, and objects that survive a few rounds move to the "old" area, which is checked less often. It is like clearing your desk every day but tidying the cupboard only once in a while.

I cannot call `gc()` in production JS. I just drop references. Once nothing points to an object any more, the GC will clean it up on its own time.

**Cross-question:** `delete obj.x` vs set null?

**Cross-answer:**

Setting it to null/undefined is enough. That drops the reference, so the old value can be collected. `delete` can also change the object's shape and be slower, because the engine may switch the object to a slower internal mode. And for a leak, the real problem is still that something holds on to the whole object. Clearing one field does not help if the object itself is still kept somewhere, like in a global list.

---

## Q23. How do you keep a SPA from XSS if you must show rich text?

**Answer:**

Rich text means formatted content like bold, lists and links, usually stored as HTML. The risk is that this HTML can also hide a script that runs on our page.

I use a sanitizer library with an allow-list of tags and no scripts, plus Angular's sanitizer, and I never bypass it. A sanitizer cleans the HTML: it keeps safe tags like `<b>` and `<p>` and removes anything dangerous, like `<script>` tags or `onclick` attributes. Never bypassing it means not using Angular's `bypassSecurityTrust...` functions on user content. If the product allows it, I prefer markdown over HTML. Markdown is a much smaller format, so there is far less room for tricks.

**Cross-question:** `innerHTML = userHtml`?

**Cross-answer:**

No. That puts the user's raw HTML straight into the page with no cleaning, so any script trick inside it, like an `<img onerror=...>`, runs for every visitor.

---

## Q24. `performance.now` vs `Date.now`?

**Answer:**

`Date.now` is the wall clock, and it can jump. It reads the computer's clock, which can change when the system syncs its time or the user changes it. If that happens while I am timing something, I can get a wrong or even negative duration.

`performance.now` is monotonic, meaning it only moves forward, so it is good for measuring a function. It counts from when the page started and is also more precise, with fractions of a millisecond. I use it when I actually measure something, like how long a big sort takes. For timestamps I show or store, I still use `Date`.

**Cross-question:** Measure in production?

**Cross-answer:**

I sample. That means I measure only a small share of requests or users, which is enough to see the pattern without slowing everyone down. I do not `console.log` every request in a hot path, because logging itself takes time and floods the console.

---

## Q25. What is tail latency in the browser?

**Answer:**

Latency is how long the user waits for something to happen. It is not the average click, it is the slow 1%. The "tail" is the far end of the chart, the rare slow cases. The average can look fine while some users regularly wait several seconds, and those are the people who complain.

It comes from things like GC pauses, huge lists, a cold cache, or a third-party script. A cold cache means the data or files are not loaded yet, so the first visit is slower. I look at p95, not only the happy demo. p95 is the time that 95% of actions are faster than, so it shows what the slower users actually feel.

**Cross-question:** How would you improve a slow list?

**Cross-answer:**

Virtual scroll, track by id, OnPush or signals, less work per row, and fewer watchers. Virtual scroll only creates the rows that are on screen, not all 10,000. Track by id lets Angular reuse existing rows instead of rebuilding them. OnPush or signals make Angular check a row only when its data really changed. But I profile first, so I am not guessing, and I fix the part that is actually slow.


---
## Additional Questions
### Q26. What is event delegation?
**Answer:** Instead of adding a listener to every child, I listen on the parent and check the event target. This works because most events "bubble": a click on a list item also travels up to its parent. The event's target tells me which child was actually clicked.

It is useful for large, dynamic lists. With 1,000 rows I need one listener instead of 1,000, which saves memory. Rows added later work automatically, and I do not have to remember to remove listeners from rows that get deleted. It is like one receptionist at the front desk handling every visitor, instead of a guard at every office door.
### Q27. What is a microtask?
**Answer:** A microtask is a small job that runs right after the current code finishes, before the browser moves on to anything else. Promise callbacks and queueMicrotask run as microtasks. So a `.then` callback runs before a `setTimeout(..., 0)`, even if the timer was set first.

The event loop runs all the microtasks before it moves to the next task, so a huge microtask chain can still block the UI. That includes new microtasks added while the queue is being emptied. The browser cannot paint or handle clicks until the microtask queue is empty, so a promise chain that keeps adding more work freezes the page just like a long loop.
### Q28. Why can `JSON.stringify` be a poor deep-clone strategy?
**Answer:** People often deep-copy with `JSON.parse(JSON.stringify(obj))`, which turns the object into text and back. It loses things like undefined, functions, Map and Set, and it changes some special values. Fields set to `undefined` and functions just disappear, a Map or Set becomes an empty object, a Date comes back as a string, and `NaN` and `Infinity` become `null`. It also throws an error if the object refers back to itself.

For supported data, structuredClone is a better general tool. It keeps Dates, Maps, Sets and self-references. It still cannot copy functions or DOM nodes.
