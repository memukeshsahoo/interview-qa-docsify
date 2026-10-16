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

In the example, `makeCounter` finishes and returns the little arrow function. Normally `n` would be gone at that point, but the arrow still uses it, so JavaScript keeps `n` alive just for that arrow. Each call to the returned function adds one to the same `n`: 1, 2, 3...

Think of it like a backpack. When the inner function is created, it packs the variables it needs and carries them wherever it goes. Nobody outside can open that backpack, which is why it's useful for private state: code outside can't change `n` directly, only through the function.

**Cross-question:** Memory leak?

**Cross-answer:**

Yes, it can cause one. A long-lived inner function keeps those outer variables alive. For example, a DOM handler that closes over a big object can keep it in memory even after you leave the page. So I remove listeners.

A memory leak means memory that's no longer needed but never gets freed. Normally JavaScript cleans up values nobody uses. But if an event handler is still attached and still "carries" a big object in its backpack, that object counts as in use. Removing the listener (or unsubscribing) when the component goes away lets JavaScript clean both up.

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

Step by step: the loop runs three times and schedules three callbacks, but none of them run yet, because `setTimeout` always waits until the current code is done. By the time the loop finishes, `i` has been increased to `3` (that's what made the loop stop). Then the three callbacks run, and they all look at the same single `i`, so each prints `3`.

With `let i`, each round of the loop gets its own copy of `i`: one with `0`, one with `1`, one with `2`. Each callback remembers its own copy, so the output is `0 1 2`.

**Cross-question:** Does `let` in `for` really get a new binding?

**Cross-answer:**

Yes. It is written into the spec so this bug goes away. The spec is the official rulebook for JavaScript. It says that for a `let` in a `for` loop, each round gets a fresh variable, starting from the value the previous round ended with. So every callback captures a different `i`.

---

## Q3. Hoisting and TDZ?

**Answer:**

`function` declarations can be called above their line. `let`/`const` sit in a “dead zone” (the TDZ) until their line runs. If you touch them early, it throws.

Before running any code in a scope, JavaScript first notes down every declared name. A `function` declaration is fully ready from the start. A `let` or `const` name is known, but locked until its line runs. That locked period is the TDZ (temporal dead zone). Touching it early gives a `ReferenceError`.

Why it's helpful: it turns a silent mistake into a loud one. With old `var`, using a variable too early quietly gave `undefined`. With `let`/`const`, I get a clear error pointing at the problem.

**Cross-question:** Classes?

**Cross-answer:**

You cannot use a class before its line. It behaves like `let`, not like `function`. So if I write `new User()` above `class User {}`, I get a `ReferenceError`. The fix is just to define the class before using it, or keep it in its own file and import it.

---

## Q4. Prototypes vs `class`?

**Answer:**

`class` is just nicer syntax on top of prototypes. Methods live on `Type.prototype` and are shared. Each instance has a link up the prototype chain.

A prototype is a shared "parent" object that other objects can borrow from. When I ask an object for a property it doesn't have, JavaScript follows the link to its prototype and looks there, then that prototype's prototype, and so on. That path is the prototype chain.

It's like a family recipe book. Each family member (instance) has their own notes, like their name, but they all share one recipe book (the methods on the prototype). There's no need to copy the whole book for every person, which saves memory. `class` just gives us a cleaner way to write this setup; underneath it's still prototypes.

**Cross-question:** Own property vs prototype?

**Cross-answer:**

Fields set on `this` are own properties. Methods are usually on the prototype. `Object.hasOwn` checks for own properties.

An "own" property belongs directly to that one object, like `this.name = 'Asha'`. A property from the prototype is borrowed. Both can be read the same way, so `Object.hasOwn(obj, 'name')` is how I tell them apart when it matters, for example when looping over an object's keys.

---

## Q5. `call` / `apply` / `bind`?

**Answer:**

All three set `this`. `bind` returns a new function. `call` runs the function right now with a list of arguments. `apply` does the same but takes an array of arguments.

`this` is the object a function is working for. Normally it's decided by how the function is called. These three let me choose it myself. For example, `greet.call(user, 'Hi')` runs `greet` straight away with `this` set to `user`. `greet.apply(user, ['Hi'])` does the same, but the arguments come in an array. `greet.bind(user)` doesn't run anything yet; it gives me a new function that will always use `user`, which is handy for callbacks and event handlers.

**Cross-question:** Bind an arrow?

**Cross-answer:**

It does not change `this`. Arrows take `this` from where they are written. An arrow function has no `this` of its own, so there's nothing for `bind`, `call` or `apply` to replace. They still pass the arguments, but `this` stays the outer one.

---

## Q6. Promise states?

**Answer:**

A promise starts as pending, then becomes fulfilled or rejected. `.then` returns a new promise. If I forget to return or throw inside it, I end up swallowing bugs.

A promise is like an order receipt for something that isn't ready yet. "Pending" means still waiting. "Fulfilled" means it worked and you got the result. "Rejected" means it failed, with an error. Once it's fulfilled or rejected, it never changes again.

Because each `.then` gives back a new promise, steps can be chained. But the next step only gets what I `return`. If I start another async call inside a `.then` and don't return it, the chain won't wait for it, and if it fails, the error doesn't travel down to my `.catch`. That's how bugs get quietly lost.

**Cross-question:** `all` vs `allSettled` vs `race` vs `any`?

**Cross-answer:**

`all`: if one fails, the whole thing fails. If all succeed, I get every result in order.

`allSettled`: waits for every one to finish. I get a result for each, saying whether it succeeded or failed.

`race`: the first one to finish, success or fail. Useful for something like a timeout: race the request against a timer.

`any`: the first success. It only fails if every one of them fails.

---

## Q7. `async`/`await` vs `.then`?

**Answer:**

It is the same thing underneath. `await` is just easier to read, and `try/catch` works with `await`. One thing to watch: many `await`s in a row run one after another. `Promise.all` runs them together.

An `async` function always returns a promise, and `await` pauses that function until a promise is done. So the code reads top to bottom like normal code, instead of a chain of `.then` callbacks.

The "in a row" point matters for speed. If I load a user, then their orders, then their messages, each with its own `await`, and they don't depend on each other, the total wait is all three added up. Starting them together with `Promise.all` means the total wait is about as long as the slowest one. It's like putting three things in the oven at once instead of baking them one after another.

**Cross-question:** `forEach(async ...)`?

**Cross-answer:**

`forEach` does not wait. I use `for...of`, or `map` + `Promise.all`.

`forEach` fires the async function for each item and moves on straight away, ignoring the promises that come back. So the code after it runs before the work is done. `for...of` with `await` does the items one at a time, in order. `map` + `Promise.all` starts them all together and waits until every one is finished.

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

Step by step: first, all the normal code runs top to bottom. That prints `a`, schedules the timer, schedules the promise callback, and prints `d`. Now the current code is done. Next, JavaScript runs all waiting microtasks, which are promise callbacks, so `c` prints. Only after that does it take the next macrotask (normal task) from the queue, which is the timer, so `b` prints last.

A simple comparison: microtasks are the "one more quick thing before I move on" list, and macrotasks are the next appointments in the diary. JavaScript always clears the quick list before starting the next appointment.

**Cross-question:** Can microtasks starve the screen?

**Cross-answer:**

Yes, if I keep queueing them forever. The browser waits until that queue is empty before it paints.

If each microtask adds another microtask, the quick list never empties. The browser never gets a chance to redraw the screen or handle clicks, so the page looks frozen. "Starve" just means the painting and other tasks never get their turn.

---

## Q9. Debounce vs throttle?

**Answer:**

Debounce: wait until the user pauses, like in a search box.

Throttle: run at most once every X ms, like on scroll.

Both control how often a function runs when an event fires many times very quickly. Without them, typing "laptop" in a search box could send six requests, one per letter. With debounce, I wait until the user stops typing for, say, 300 ms, then send just one request.

Throttle is different: it doesn't wait for a pause, it just sets a speed limit. A scroll event can fire many times a second. With throttle, my handler still runs regularly while the user scrolls, but never more than once every X ms. A lift door is a good picture of debounce: it waits until people stop walking in before it closes.

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

How it works: every call cancels the previous timer and starts a new one. So `fn` only runs once the calls stop for `ms` milliseconds. The variable `t` is kept in a closure, so all calls share the same timer. Clearing it on destroy stops `fn` from running after the component is gone, which could otherwise cause errors or wasted calls.

---

## Q10. Map vs Set vs WeakMap?

**Answer:**

Map: a key can be anything. Set: unique values only. WeakMap: keys must be objects, it does not keep them alive, and you cannot loop over the keys. I use WeakMap for attaching extra data to a DOM node, not for a cache keyed by string ids.

A `Map` is like a plain object for key-value pairs, but the keys don't have to be text: they can be numbers, objects, anything. A `Set` is a list where each value can appear only once, so it's an easy way to remove duplicates.

A `WeakMap` holds its keys "weakly". If nothing else in the app still uses the key object, JavaScript is allowed to throw away both the key and its value. That's why it's good for attaching extra data to a DOM node: when the node is removed and forgotten, the extra data goes with it automatically, with no leak. It's also why you can't loop over it, because entries can disappear at any time.

**Cross-question:** Cache API results in WeakMap?

**Cross-answer:**

The keys must be objects. For string ids I use a Map with a size limit.

A `WeakMap` simply won't accept a string as a key. A normal `Map` will, but it keeps everything forever unless I remove it, so a cache in it can grow without end. A size limit (for example, dropping the oldest entry when it gets full) keeps memory under control.

---

## Q11. Shallow copy traps?

**Answer:**

`{ ...obj }` still shares the nested objects. So changing `copy.address.city` also changes the original. It also breaks OnPush change detection if I thought I had a new object.

Spread copies only the top layer. The `address` inside is not copied, both objects just point at the same `address`. So editing the city through the copy edits it for the original too, which surprises people.

OnPush is an Angular setting where a component only re-checks when its input gets a new object reference, not when something inside the same object changes. If I change something nested in place, the nested object is still the same one, so a component that receives that nested object doesn't see a change, and the screen can show old data.

**Cross-question:** Fix?

**Cross-answer:**

I copy the nested part too, or use a structured clone when I really need a deep copy. For example, `{ ...obj, address: { ...obj.address, city: 'Pune' } }` makes a new top object and a new `address`, so the original stays untouched and Angular sees new references. `structuredClone` copies every level, which is simpler but does more work, so I keep it for when I really need it.

---

## Q12. `?.` `??` `??=`?

**Answer:**

`?.` is optional chaining. `??` gives a default only for null or undefined. `??=` assigns only if the value is null or undefined, so `0` stays.

In plain words: `user?.name` means "read `name`, but if `user` is missing, just give me undefined instead of crashing". `count ?? 10` means "use `count`, unless it has no value at all, then use 10". `settings.size ??= 20` means "if `size` has no value yet, set it to 20; otherwise leave it alone".

Why they matter: they handle missing data safely without treating real values like `0`, `''` or `false` as missing. That's the difference from `||`, which treats all of those as empty.

**Cross-question:** `||` for pageSize?

**Cross-answer:**

`0 || 10` gives 10, which is wrong. Use `??`. `||` falls back for any falsy value, and `0` is falsy. `0 ?? 10` gives `0`, because `??` only falls back for null or undefined.

---

## Q13. Iterators and generators?

**Answer:**

`for...of` uses iterators. `function*` with `yield` creates one. Observables push values to you, while generators let you pull values. For UI events, RxJS is a better fit.

An iterator is an object that hands out values one at a time when you ask for the next one. Arrays, strings, Maps and Sets all have one, which is why `for...of` works on them. A generator is an easy way to write your own: each `yield` hands out one value and pauses the function until the next value is asked for.

Pull vs push: with a generator, my code decides when to get the next value, like taking the next ticket from a dispenser. With an Observable (from RxJS), values arrive whenever they happen, like notifications on a phone. Clicks and key presses happen whenever the user does them, so the push style fits them better.

**Cross-question:** `for...in` on arrays?

**Cross-answer:**

It gives you the indexes as strings, plus any extra keys. I do not use it for arrays. So for `['a', 'b']` I get `'0'` and `'1'` as text, not the values. If someone has added an extra property to the array or its prototype, that can show up too. `for...of` gives the actual values, which is what I usually want.

---

## Q14. Event bubbling?

**Answer:**

An event first travels down (capture), then back up (bubble). `preventDefault` stops the browser's default action, like a form submit. `stopPropagation` stops handlers on the parents from running.

When I click a button inside a card inside a page, the click doesn't just belong to the button. The browser first passes it down from the top of the page to the button (capture), then back up from the button through the card to the page (bubble). Handlers on any of those elements can react. By default, handlers listen during the bubble phase.

These two methods do different jobs. `preventDefault` says "don't do the browser's normal thing", like following a link or reloading the page on submit. `stopPropagation` says "don't tell the parents about this event". One doesn't do the other's job.

**Cross-question:** Row click vs button click?

**Cross-answer:**

The button click bubbles up to the row. If the row should not also fire, I stop propagation.

For example, a table row opens a details page when clicked, and the row has a "Delete" button. Clicking Delete also bubbles up to the row, so without a fix it deletes and opens details at the same time. Calling `stopPropagation` in the button's handler keeps the click to the button only.

---

## Q15. CORS from the Angular side?

**Answer:**

I cannot really “fix CORS” in Angular, except with a **dev proxy**. The API has to send the right headers. Postman works because it is not a browser.

CORS (Cross-Origin Resource Sharing) is a browser rule. If my Angular app runs on one address and calls an API on a different one, the browser checks whether the API has said "I allow calls from this site", using special response headers. If those headers aren't there, the browser blocks my code from reading the response.

Because the check is about what the server says, the real fix is on the API. A dev proxy works during development because the Angular dev server forwards the calls for me, so the browser thinks it's talking to the same address. Postman isn't a browser, so it doesn't apply this rule at all, which is why a call can work in Postman and fail in the app.

**Cross-question:** Is CORS auth?

**Cross-answer:**

No. CORS is a browser safety lock. The API still needs its own login. CORS only controls which websites' code is allowed to read the API's responses inside a browser. Anyone can still call the API directly from a script, a server or Postman, so it doesn't prove who the caller is. The API needs real authentication, like a token, to protect its data.

---

## Q16. `bind` in a React/Angular template?

**Answer:**

`bind` in a template creates a new function on every check. I bind once in the class, or use an arrow field.

The template is re-checked often, every time the framework looks for changes. If the template contains `.bind(this)`, each check makes a brand new function. That's extra work, and to the framework a new function looks like a changed value, which can cause child components to update when nothing really changed. Doing the binding once, in the class, means the same function is reused every time.

**Cross-question:** Arrow as a class field?

**Cross-answer:**

It keeps `this`. It costs one function per instance, which is fine for UI handlers.

An arrow class field looks like `onSave = () => { ... }`. Because it's an arrow, `this` always points to the component, even when it's passed around as a callback. The cost is that each component instance gets its own copy of the function, instead of sharing one on the prototype. For a handful of handlers on a component, that cost is tiny.

---

## Q17. Deep equality of objects?

**Answer:**

`===` does not compare deeply. I compare ids, or use JSON stringify if I accept its limits, or a helper function. In tests, I use a matcher. In Angular, I often replace the object instead of deep-watching it.

"Deep" comparison means checking every field, and every field inside those fields, to see if two objects hold the same data. `===` only asks "is this the very same object?". So two separate objects with identical data are not equal with `===`.

The options each have trade-offs. Comparing ids is quick and often enough. `JSON.stringify` on both and comparing the text is simple, but it fails if the keys are in a different order and it loses things JSON can't hold, like dates or functions. Test tools have matchers (like `toEqual`) built for this. In Angular, if I always make a new object when data changes, a simple `===` check is enough to know something changed, so I don't need a slow deep check at all.

**Cross-question:** Lodash `isEqual` in a hot list?

**Cross-answer:**

It can be slow there. I prefer comparing id + version.

A "hot" list is one that's checked very often, like a big table that re-renders a lot. `isEqual` walks through every field of every object each time, which adds up. If each item has an id and a version number that goes up whenever it changes, comparing those two small values tells me the same thing much faster.

---

## Q18. `queueMicrotask`?

**Answer:**

It schedules a function as a microtask, the same kind as `then`. It is useful to run something after the current stack, but before paint.

"After the current stack" means after the code that's running right now has finished. "Before paint" means before the browser redraws the screen. So it's a way to say "do this right after I'm done, but before the user sees anything". That's handy when I want to group several small updates and handle them once, without the user seeing an in-between state.

**Cross-question:** vs `setTimeout(0)`?

**Cross-answer:**

The microtask runs sooner. A timeout is a normal task. All waiting microtasks run as soon as the current code finishes. A `setTimeout` callback has to wait its turn in the normal task queue, and the browser may paint the screen before it runs.

---

## Q19. Error boundaries? (JS)

**Answer:**

`window.onerror` and `unhandledrejection` catch whatever slips through. In Angular I also use `ErrorHandler`. I still use try/catch near risky I/O.

`window.onerror` is a global listener for errors nobody caught. `unhandledrejection` is the same idea for promises that failed with no `.catch`. Angular's `ErrorHandler` is a single place where Angular sends errors from inside the app, so I can log them or show a friendly message.

These are the last safety net, like a building's main fuse box. They stop errors from vanishing silently and help me report them. But by the time an error reaches them, it's too late to recover nicely, so near risky things like network calls or file reads, I still use try/catch and handle the problem right there.

**Cross-question:** Swallow in empty catch?

**Cross-answer:**

Never. At the very least, I log it with some context.

An empty `catch` hides the error completely. The app keeps going in a broken state, and nobody knows why. Logging it with context, like which action failed and for which record, means that when something goes wrong, I can actually find out what happened.

---

## Q20. `Object.freeze` vs `const`?

**Answer:**

`const` means you cannot reassign the variable. `freeze` means you cannot add or change keys, but only at the top level. Nested objects are not frozen.

These protect two different things. `const` protects the name: it will always point to the same object. `Object.freeze` protects the object itself, so its properties can't be added, removed or changed. For example, with `const config = Object.freeze({ api: { url: '...' } })`, I can't change `config.api` to something else, but I can still change `config.api.url`, because the inner object wasn't frozen.

**Cross-question:** Frozen in Angular state?

**Cross-answer:**

It is nice for discipline. But signals plus new objects is the usual approach.

Freezing state makes any accidental in-place change fail, which helps catch bugs. But the usual way in Angular today is to keep state in signals (values that tell Angular when they change) and always set a new object when updating. That way Angular sees the change and updates the screen, without needing freeze.

---

## Q21. Module cycles?

**Answer:**

That is when A imports B, and B imports A. You can end up reading a `let` that is still in the TDZ. I move the shared parts into a third file.

When A starts loading, it first needs B, so B loads. But B needs A, and A isn't finished yet. So B sees A's exports before they've been set up, and using them throws an error or gives unexpected results. It's like two people each waiting for the other to hand over a document first.

Moving the shared code into a third file, C, breaks the loop: A and B both import from C, and C imports neither of them. Each file loads cleanly in order.

**Cross-question:** `import type` in TS?

**Cross-answer:**

It is erased at compile time. So types alone do not create a runtime cycle. `import type` brings in only type information, for TypeScript's checks. Once compiled, that import line is gone from the JavaScript, so nothing is loaded at runtime and no loop can form.

---

## Q22. `new Promise` executor runs when?

**Answer:**

Right away, when you construct the promise. The resolve/reject functions are for later. I do not wrap an existing promise in `new Promise` without a reason (that is the “promise constructor anti-pattern”).

The "executor" is the function I pass to `new Promise((resolve, reject) => { ... })`. It runs immediately, on the same line, not later. What happens later is calling `resolve` (success) or `reject` (failure), when the slow work finishes.

An "anti-pattern" is a common habit that causes problems. If I already have a promise, like from `fetch`, wrapping it in another `new Promise` adds extra code and makes it easy to forget to pass errors along, so failures can get lost. I just return or `await` the existing promise instead.

**Cross-question:** Convert callback API?

**Cross-answer:**

Yes, that is what `new Promise` is for, wrapping `fs.readFile(cb)` style APIs. Not for wrapping `fetch`.

Older APIs report their result by calling a function I pass in (a callback), instead of returning a promise. Wrapping them in `new Promise` lets me call `resolve` with the result or `reject` with the error inside that callback. After that, I can use `await` with it like any modern API. `fetch` already returns a promise, so it doesn't need wrapping.

---

## Q23. `Array.from` vs spread?

**Answer:**

Both turn array-like things into a real array. `Array.from` can also map in the same step. Spread needs something iterable.

"Array-like" means something that has a length and numbered items but isn't a real array, so it's missing methods like `map` and `filter`. "Iterable" means something you can loop over with `for...of`. Spread (`[...x]`) only works on iterables. `Array.from` also accepts plain array-like objects that only have a length and numbered keys.

The mapping step is handy: `Array.from({ length: 5 }, (_, i) => i)` builds `[0, 1, 2, 3, 4]` in one go.

**Cross-question:** `NodeList`?

**Cross-answer:**

What `querySelectorAll` returns is not a real array. I use `Array.from` or spread to get `.map`. A `NodeList` is a list of page elements. It does have `forEach`, but not `map`, `filter` or `reduce`. Turning it into a real array gives me all the usual array methods.

---

## Q24. Strict mode?

**Answer:**

Modules and classes are always strict. Assigning to an undeclared variable throws. `this` is not `window` in a plain function call. And `with` is banned.

Strict mode is a stricter set of rules for JavaScript that turns some silent mistakes into real errors. Without it, a typo like `totl = 5` (instead of `total`) quietly creates a new global variable, and the bug is hard to find. In strict mode, it throws an error straight away.

The `this` change works the same way: in a plain function call, `this` is `undefined` instead of the global `window`, so code that accidentally uses `this` fails loudly instead of changing global values. Since modules and classes are always strict, most modern code already follows these rules.

**Cross-question:** Script became a module and `this` broke?

**Cross-answer:**

Yes, because modules are strict. If I need the global, I use `window` on purpose.

In an old-style script, `this` at the top level is `window`. In a module, it's `undefined`. So old code that relied on `this` meaning the global object stops working after the switch. Writing `window` (or `globalThis`) makes it clear I really mean the global object.

---

## Q25. How do you avoid callback hell without async/await?

**Answer:**

With promise chains. They are still better than nested callbacks. Today I use async/await. Either way, I return the promise so callers can wait on it.

"Callback hell" is when each step's code sits inside the previous step's callback, so the code drifts further and further to the right and becomes hard to read. Error handling also has to be repeated at every level. With a promise chain, each step is a `.then` at the same level, read top to bottom, with one `.catch` at the end for all errors.

Returning the promise matters because it lets whoever called my function know when the work is done and whether it failed. If I don't return it, the caller has nothing to wait on, and errors can go unnoticed.


---
## Additional Questions
### Q26. `Promise.all` vs `Promise.allSettled`?
**Answer:** Promise.all fails as soon as one promise rejects. allSettled waits for every promise and gives me each result, so it is useful for independent dashboard calls. For example, if a dashboard loads sales, tasks and alerts separately and only the alerts call fails, `allSettled` still lets me show sales and tasks, with an error message just for alerts. With `Promise.all`, that one failure would make the whole thing fail. I use `Promise.all` when every result is needed and one missing piece means the whole step can't continue.
### Q27. Debounce vs throttle?
**Answer:** Debounce waits until the activity stops, which is good for search. Throttle limits how often something runs, which is good for scroll or resize events. With debounce, if someone types quickly, nothing happens until they pause, then it runs once. With throttle, it keeps running while the activity goes on, but only at a fixed pace, like once every 200 ms. Both stop the app from doing heavy work, like API calls or layout updates, on every single event.
### Q28. What is the nullish coalescing operator?
**Answer:** `value ?? fallback` uses the fallback only for null or undefined. Unlike `||`, it does not replace valid values like 0 or false. "Nullish" just means null or undefined, the two values that mean "nothing here". So if a user sets a discount of `0`, `discount ?? 10` keeps `0`, while `discount || 10` would wrongly change it to `10`.
