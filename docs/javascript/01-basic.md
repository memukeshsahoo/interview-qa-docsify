# JavaScript — Basic

The browser runs JS. TypeScript adds types on top. Interviews still ask these basics.

---

## Q1. JavaScript vs TypeScript?

**Answer:**

JavaScript is what the browser actually runs. TypeScript adds types on top, and those types are erased when it compiles. So if the types are wrong, or you use `any`, you can still get runtime errors.

A "type" is just a label that says what kind of value something holds, like "this is a number" or "this is a user with a name and an email". TypeScript checks those labels while I write the code, so it can tell me "you passed text where a number was expected" before anyone runs the app. Think of it like a spell checker: it catches mistakes while you type, but it doesn't go with the letter once you post it.

Why it matters: in a big project, many mistakes are simple mix-ups, like a wrong property name or a missing field. TypeScript catches those early. But once the code is compiled, only plain JavaScript reaches the browser. So if data from an API doesn't match the type I wrote, or I used `any` (which switches checking off), nothing stops that bad value at runtime.

**Cross-question:** Is TypeScript a runtime?

**Cross-answer:**

No. After it compiles, it is just JS. The browser or Node never sees TypeScript. All the type checks happen before the code runs, so there is nothing left at runtime to check the real data. If I need to check data coming from outside (like an API response), I have to write that check myself in normal JS.

---

## Q2. `var` vs `let` vs `const`?

**Answer:**

`var`: function scope. It is the old way, so I avoid it.

`let`: block scope, and you can reassign it.

`const`: block scope, and you cannot reassign the binding. But the object it points to can still change.

"Scope" means where in the code a variable can be seen. Function scope means the variable is visible everywhere inside the whole function, even outside the `if` or loop where I wrote it. Block scope means it only lives inside the nearest `{ }`, like one `if` block or one loop. Block scope is easier to reason about, because a variable can't leak out and get mixed up with code further down.

My rule of thumb: I use `const` by default, and `let` only when the value really needs to change, like a counter. That way, when I read the code, I know straight away which values stay fixed.

**Cross-question:** Why does `const user = {}; user.name = 'A'` work?

**Cross-answer:**

`const` locks the variable, not the object. If I really need to stop changes, I use `Object.freeze`, and that is only a shallow freeze.

Think of `const` as a name tag stuck on a box. You can't move the tag to a different box, but you can still put things in the box or take them out. `Object.freeze` seals the box itself, but only the top layer: if there's a smaller box inside, that inner box can still change.

---

## Q3. Primitives vs objects?

**Answer:**

The primitives are string, number, boolean, null, undefined, symbol and bigint. Everything else is an object, including arrays and functions. When you use `===` on objects, it checks if they are the same instance.

Primitives are simple single values, like `5` or `'hello'`. They are compared by their value, so `'hi' === 'hi'` is true. Objects are compared by identity: are these two variables pointing at the very same thing in memory? So `{} === {}` is false, because those are two separate objects, even though they look the same.

A real-life comparison: two people can each own the same model of phone. Same look, but they are still two different phones. `===` on objects asks "is this the same phone?", not "is it the same model?". That is why comparing two objects with `===` can surprise beginners.

**Cross-question:** `typeof null`?

**Cross-answer:**

It returns `'object'`. That is an old bug in the language. So I check `=== null` instead. The bug comes from the very first version of JavaScript, and it can't be fixed now because too many existing websites depend on it.

---

## Q4. `==` vs `===`?

**Answer:**

`==` converts types before comparing, so `'0' == 0` is true. `===` does not convert anything. I use `===`.

With `==`, JavaScript tries to be helpful: if the two sides are different types, it quietly turns one into the other and then compares. That sounds nice, but the rules are hard to remember and give odd results, like `'' == 0` being true. With `===`, a string and a number are never equal, full stop.

Why it matters: a value from a form input is always a string, like `'5'`. If I compare it with `==`, it may "match" a number and hide the fact that I forgot to convert it. `===` makes that mistake visible.

**Cross-question:** `null == undefined`?

**Cross-answer:**

That is true with `==`. Some teams use `== null` on purpose to check for both. I still prefer to be explicit, unless the team likes that style. With `===`, `null === undefined` is false, so to cover both I'd write the two checks, or use `??` when I just want a default value.

---

## Q5. Truthy and falsy?

**Answer:**

The falsy values are `false`, `0`, `''`, `null`, `undefined` and `NaN`. An empty array `[]` and an empty object `{}` are truthy.

When I write `if (something)`, JavaScript doesn't need a real `true` or `false`. It treats a few values as "counts as false" (falsy), and every other value as "counts as true" (truthy). It's like asking "is there anything here?": an empty text or the number zero says "no", but any object, even an empty one, says "yes".

Why it matters: it's a common source of bugs. For example, if a quantity is `0`, then `if (quantity)` is false, even though `0` is a perfectly valid quantity. So for numbers I compare directly, like `quantity !== undefined`.

**Cross-question:** `if (arr)` on an empty array?

**Cross-answer:**

It is true. So I check `arr.length` instead. An empty array is still an object, so it's truthy. `if (arr.length)` is false for an empty array because its length is `0`, which is falsy. So that's the check that really means "does this list have items?".

---

## Q6. Function declaration vs arrow?

**Answer:**

`function add() {}` is hoisted, so you can call it before its line. Arrows are not, and they keep the `this` from the outer scope.

"Hoisted" means JavaScript prepares the function before running the file, so I can call it near the top even if it's defined at the bottom. An arrow function is usually stored in a `const`, like `const add = () => {}`, and it only exists once that line has run.

The other difference is `this`, which roughly means "the object this function is working for". A normal function gets its `this` from how it's called. An arrow doesn't have its own `this`; it uses the one from the code around it. That makes arrows great for short callbacks inside a class method, because they keep pointing at the same object.

**Cross-question:** When do arrows hurt?

**Cross-answer:**

When you use them as object methods that need their own `this`. Also as constructors, because you cannot use `new` with an arrow.

For example, if I write `const counter = { count: 0, inc: () => this.count++ }`, the `this` inside the arrow is not `counter`. It's whatever `this` was outside the object, so the count never goes up. A normal method, `inc() { this.count++; }`, works as expected.

---

## Q7. What is `this`?

**Answer:**

It depends on **how** you call the function.

`obj.fn()` → obj. The object before the dot becomes `this`.

Plain `fn()` → `undefined` in strict mode. There's no object before the dot, so there's nothing to point at.

`new Fn()` → the new object. `new` creates a fresh empty object and runs the function with `this` set to it.

Arrow → the outer `this`. An arrow doesn't pick its own; it borrows the one from where it was written.

A simple way to think about it: `this` is like the word "me" in a sentence. Who "me" refers to depends on who is speaking at that moment, not on where the sentence was written down. In the same way, the same function can have a different `this` each time it's called.

**Cross-question:** Callback lost `this`?

**Cross-answer:**

That happens when I pass `obj.method` without calling it. The fix is `bind`, or `() => obj.method()`.

When I pass `obj.method` to something like `setTimeout` or a click handler, I'm only handing over the function itself, not the object. Later it gets called as a plain function, so `this` is no longer `obj`. `bind` creates a copy of the function with `this` locked to `obj`. The arrow version works because it calls `obj.method()` with the object in front of the dot.

---

## Q8. Array methods you use daily?

**Answer:**

`map`, `filter`, `reduce`, `find`, `some`, `every`, `slice` (makes a copy) and `splice` (changes the array). For Angular state, I prefer copying over mutating.

In plain words: `map` changes each item into something else (like prices into prices with tax), `filter` keeps only the items I want, `reduce` boils the list down to one value (like a total), `find` gives me the first match, and `some` / `every` answer "does at least one / does every item match?".

Why copying matters: "mutating" means changing the original array in place. In Angular, the screen often updates only when it sees a new array or object. If I change the old one in place, Angular may not notice, and the screen shows stale data. Making a copy also means other parts of the app that hold the old array don't get surprised.

**Cross-question:** `map` vs `forEach`?

**Cross-answer:**

`map` returns a new array. `forEach` is for side effects. I never use `map` and then ignore the result.

A "side effect" is something done outside the function, like logging, saving or updating the screen. If I just want to do something for each item, `forEach` says that clearly. Using `map` and throwing away the result builds an array for nothing and confuses the next person who reads the code.

---

## Q9. Copy an object?

**Answer:**

For a shallow copy I use `{ ...obj }`, but nested objects are still shared. For a deep copy I use `structuredClone` in modern browsers. A JSON copy turns dates into strings and drops functions.

A shallow copy copies only the top layer. If `user` has an `address` object inside it, both the original and the copy still point at the same `address`. So changing the city in the copy also changes it in the original. A deep copy copies every layer, so the two are fully separate.

It's like photocopying a folder's cover page, while both folders still share the same papers inside. That is a shallow copy. A deep copy photocopies every page. The JSON trick (`JSON.parse(JSON.stringify(obj))`) also does a deep copy, but it loses anything JSON can't hold, so I prefer `structuredClone`.

**Cross-question:** Destructuring rest to drop password?

**Cross-answer:**

`const { password, ...safe } = user` — and then I never log `password`.

This line pulls `password` out into its own variable, and puts every other field into a new object called `safe`. So `safe` is a copy of the user without the password, which I can log or send to the screen without leaking it.

---

## Q10. Template strings and defaults?

**Answer:**

A template string looks like `` `Hello ${name}` ``. A default parameter looks like `function f(x = 1)`.

A template string uses backticks instead of quotes, and anything inside `${ }` gets filled in with its value. So if `name` is `'Asha'`, I get `'Hello Asha'`. It also allows strings over several lines. It's much easier to read than gluing pieces together with `+`.

A default parameter is the value a function uses when the caller doesn't pass one. With `function f(x = 1)`, calling `f()` makes `x` equal to `1`. It saves me from writing "if x is missing, set it to 1" at the start of every function.

**Cross-question:** Tagged templates?

**Cross-answer:**

It is a function placed in front of the template string. Some libraries use them. I rarely write my own.

For example, `` tag`Hello ${name}` `` calls `tag` with the fixed text parts and the filled-in values separately. That lets the library decide how to combine them, for example to escape values safely or to build CSS or GraphQL queries.

---

## Q11. JSON pitfalls?

**Answer:**

JSON has no functions and no `undefined`. Dates become strings, BigInt fails, and circular references throw. And never use `eval` to parse JSON.

JSON is a plain text format for sending data, mainly between a server and a browser. It only knows simple things: text, numbers, true/false, null, lists and objects. So when I turn an object into JSON, anything else gets dropped or changed. A Date becomes text like `"2024-01-05T10:00:00.000Z"`. A circular reference means an object points back to itself (like a person whose "friend" points back to them), so turning it into text would never end, and it throws an error instead.

Why not `eval`: `eval` runs text as real code. If the "JSON" came from somewhere I don't control, someone could slip code into it and it would run in my page. `JSON.parse` only reads data, so it's safe.

**Cross-question:** Revive dates?

**Cross-answer:**

I use a reviver in `JSON.parse`, or map the values afterwards. ISO strings do not turn into Date objects by magic.

A reviver is a small function I pass as the second argument to `JSON.parse`. It is called for every value, so I can spot date strings and turn them back into real `Date` objects. Without that step, the field is just text, and something like `.getFullYear()` will fail on it.

---

## Q12. try / catch and promises?

**Answer:**

`catch` only sees errors from `await` or from sync code that throws. A bare `fetch().then()` that rejects is unhandled unless I add `.catch` or `await` it.

`try / catch` works like a safety net under the code inside the `try`. But a promise is work that finishes later. If I start a `fetch` without `await`, the code moves on and leaves the `try` block straight away. When the request fails later, the safety net is already gone, so the error escapes.

With `await`, the function waits right there, inside the `try`, until the promise finishes. If it fails, the error comes out at that line and the `catch` gets it. Without `await`, I need to attach `.catch` to the promise itself.

**Cross-question:** Throw a string?

**Cross-answer:**

No, I throw an `Error` so I get a stack trace. A stack trace is the list of function calls that led to the error, with file names and line numbers. A plain string has none of that, so when it shows up in the logs I can't tell where it came from.

---

## Q13. `import` vs `require`?

**Answer:**

`import`/`export` is ESM, the standard module system. `require` is the older CommonJS style from Node. Browsers use ESM or a bundler.

A "module" is just one file of code that shares some things with other files. Splitting code into modules keeps each file small and focused. ESM (ECMAScript Modules) is the official way built into the language, so it works in both browsers and Node. CommonJS came earlier, from Node, before the language had its own system.

One practical difference: `import` statements are fixed at the top of the file, so tools can see what is used before the code runs. That lets a bundler (a tool that packs all files into a few files for the browser) remove code nobody uses, which keeps the download smaller.

**Cross-question:** Default vs named export?

**Cross-answer:**

Named exports are easier to search for and rename. A lot of teams use named exports only.

With a named export, everyone imports it with the same name, like `import { formatDate } from './utils'`. So searching for `formatDate` finds every use, and the editor can rename it everywhere. With a default export, each file can import it under any name it likes, so the same thing ends up with different names across the project.

---

## Q14. `undefined` vs `null` vs not defined?

**Answer:**

Not defined: the variable was never declared, so you get a `ReferenceError`.

`undefined`: it was declared but has no value, or the property is missing.

`null`: I set it on purpose.

Think of a form with fields. "Not defined" is a field that isn't on the form at all, so asking about it is an error. `undefined` is a field that's there but nobody filled in. `null` is a field where someone deliberately wrote "none". Keeping that difference clear makes code easier to read: `null` tells the reader "this is empty on purpose".

**Cross-question:** `user?.address?.city`?

**Cross-answer:**

It stops at the first null or undefined and returns undefined. `??` means “if this is null or undefined, use this other value”. It does not treat `0` as empty.

Without `?.`, if `user` or `address` is missing, reading `.city` crashes with "cannot read properties of undefined". With `?.`, it just gives back `undefined` instead. Combined with `??`, I can write `user?.address?.city ?? 'Unknown'` to show a fallback when anything along the way is missing.

---

## Q15. Event loop, very simple?

**Answer:**

JS runs one thing at a time. Sync code runs first. Then microtasks, like `Promise.then`. Then timers and events. That is why `Promise.then` can run before `setTimeout(0)`.

Picture a single cashier at a shop. The cashier finishes serving the current customer first (that's the sync code). Then they deal with a small "urgent" list, quick follow-ups that were promised to be done right after (microtasks, like promise callbacks). Only after that do they take the next customer from the main queue (timers, clicks and other events).

Why it matters: it explains the order things happen in. If I'm surprised that a log appears earlier or later than I expected, the event loop is usually the reason. It also explains why one long piece of code freezes the page: the cashier can't serve anyone else until it's done.

**Cross-question:** Is `setTimeout(fn, 0)` now?

**Cross-answer:**

No. It runs after the current stack is done, and maybe later if the tab is busy. The `0` only means "no extra waiting". The callback still goes into the queue, so it waits for the current code and all pending microtasks to finish first. Browsers can also delay timers, for example in background tabs.

---

## Q16. Why not `document.getElementById` in Angular?

**Answer:**

It works against Angular, it breaks tests and SSR, and using `innerHTML` is an XSS risk. I bind in the template, or use `@ViewChild` if I really have to.

Angular controls the page for me. It decides what's on the screen based on my component's data. If I reach in and change elements by hand, Angular doesn't know about it, and it may overwrite my changes on its next update, or the two can get out of sync.

SSR (server-side rendering) means the page is first built on a server, where there is no real browser `document`, so code that grabs elements directly can fail there. Unit tests can hit the same problem. XSS (cross-site scripting) is when an attacker gets their own script to run in my page. Setting `innerHTML` with user text can do exactly that. Angular's template binding cleans values for me, so I let it handle the page.

**Cross-question:** What is the DOM?

**Cross-answer:**

It is the tree of HTML nodes the browser keeps in memory. JS can change it. Angular already does that for you.

DOM stands for Document Object Model. Think of it like a family tree of the page: the `<html>` at the top, with `<body>` under it, then sections, buttons, text and so on. When something changes in that tree, the browser updates what you see on the screen.

---

## Q17. `NaN` — what is it?

**Answer:**

It means “Not a Number”. `Number('abc')` gives `NaN`. Also, `NaN === NaN` is false. So I check with `Number.isNaN`.

`NaN` is what you get when a maths operation or a conversion can't give a real number, like turning the word "abc" into a number. The odd part is that its type is still `'number'`. And it's the only value in JavaScript that isn't equal to itself, so `x === NaN` never works as a check.

Why it matters: `NaN` spreads quietly. Any maths with `NaN` gives `NaN`, so one bad input can turn a whole total into `NaN` on the screen. Checking with `Number.isNaN` early catches it.

**Cross-question:** `isNaN('hello')`?

**Cross-answer:**

The old `isNaN` converts the value first, so it says true. `Number.isNaN` is stricter, so I use that.

The old `isNaN` first turns `'hello'` into a number, gets `NaN`, and says true. So it really answers "would this fail to become a number?". `Number.isNaN('hello')` is false, because `'hello'` is a string, not the `NaN` value itself. It only says true for actual `NaN`.

---

## Q18. `typeof` results you should know?

**Answer:**

`'string'`, `'number'`, `'boolean'`, `'undefined'`, `'object'` (null gives this too), `'function'`, `'bigint'` and `'symbol'`. Arrays also give `'object'`, so I use `Array.isArray` for them.

`typeof` tells me what kind of value I'm holding, which is handy when a function can receive different kinds of input. The catch is that it's not very detailed for objects: an array, a date, a plain object and `null` all come back as `'object'`. So for those I use more specific checks, like `Array.isArray(x)`, `x === null`, or `x instanceof Date`.

**Cross-question:** Check for a Promise?

**Cross-answer:**

I use duck typing: `typeof x?.then === 'function'`. It is not perfect, but it is common.

"Duck typing" comes from the saying "if it walks like a duck and quacks like a duck, it's a duck". Here: if it has a `then` method, I treat it as a promise. That's actually how `await` and promise chains decide too. It's not perfect because any object with a `then` function would pass, even if it isn't a real promise.

---

## Q19. Spread in function calls?

**Answer:**

`fn(...args)` works like `apply`. Rest is the same syntax in the definition: `function f(...args)`.

Spread unpacks a list. If `args` is `[1, 2, 3]`, then `fn(...args)` is the same as `fn(1, 2, 3)`. It's like tipping a bag of items onto the table so each item is separate. Rest does the opposite: in a function definition, `...args` collects all the passed arguments into one array. So the same three dots either spread things out or gather them up, depending on where they're written.

**Cross-question:** Spread a string?

**Cross-answer:**

`[...'ab']` gives `['a','b']`. That can be surprising. Spread works on anything you can loop over, and looping over a string gives you its characters one by one. So spreading a string splits it into single characters.

---

## Q20. `for...of` vs `for...in` vs `forEach`?

**Answer:**

`for...of` gives you values. `for...in` gives you keys, including inherited ones, so it is bad for arrays. `forEach` cannot `break` easily, and it does not wait for async code.

For a list like `['a', 'b']`: `for...of` gives `'a'` then `'b'`, which is usually what I want. `for...in` gives the positions, as text, `'0'` then `'1'`, and it can also pick up extra keys that came from elsewhere. That's why I keep `for...in` for plain objects, not arrays.

`forEach` runs a function for every item, but I can't stop halfway with `break`. If I need to stop early, like "find the first match and stop", `for...of` with `break` is cleaner.

**Cross-question:** Async inside `forEach`?

**Cross-answer:**

It will not wait. I use `for...of` with `await` instead.

`forEach` calls my async function for each item and moves straight on, without waiting for any of them to finish. So code after the loop runs too early, and errors can go unhandled. With `for...of`, each `await` really pauses the loop until that item is done.

---

## Q21. Template vs `'string' + x`?

**Answer:**

Templates are easier to read. It is the same idea underneath. For HTML, I still never paste user text into HTML strings.

Compare `'Hello ' + name + ', you have ' + count + ' messages'` with `` `Hello ${name}, you have ${count} messages` ``. The second reads like the final sentence, and it's harder to forget a space. Both produce the same text.

The HTML point is about safety, not style. If I build HTML from text a user typed, and that text contains a `<script>` or similar, it can run in the page. A template string doesn't clean that up. I let the framework put user text on the screen, because it escapes it.

**Cross-question:** Performance?

**Cross-answer:**

For normal UI it does not matter. Just be careful when building huge strings in a hot loop. A "hot loop" is code that runs a very large number of times. There, building a big string bit by bit can get slow, and collecting the pieces in an array and calling `join` once is often a cleaner option.

---

## Q22. `parseInt` vs `Number`?

**Answer:**

`parseInt('10px')` gives 10. `Number('10px')` gives `NaN`. With `parseInt`, always pass the radix: `parseInt(s, 10)`.

`parseInt` reads from the start and stops at the first character that isn't part of a number, so it happily takes the `10` from `'10px'`. `Number` is strict: the whole text must be a number, or you get `NaN`. So I use `parseInt` when I expect extra text (like CSS sizes), and `Number` when anything extra means the input is wrong.

The radix is the number base. `10` means our normal decimal numbers. Passing it makes it clear and avoids surprises with unusual inputs.

**Cross-question:** `parseInt('08')`?

**Cross-answer:**

I always pass 10, so the old octal rules cannot bite. Octal is base 8. Some very old engines treated a leading `0` as "this is octal", which gave odd results for inputs like `'08'`. Modern engines default to base 10 here, but passing `10` makes the intent clear and removes any doubt.

---

## Q23. What is hoisting in one line?

**Answer:**

Declarations are moved to the top of their scope. So `function foo(){}` can be called before its line. `let`/`const` exist too, but they throw if you touch them before their line. That gap is called the TDZ (temporal dead zone).

Nothing is really moved in the file. Before running the code, JavaScript scans it and sets up all the declared names first. A function declaration is set up completely, so it's ready to call. A `let` or `const` name is reserved, but it isn't usable until its line actually runs.

Think of a restaurant booking: the table (the name) is reserved from the start, but you can't sit at it until your booking time. Trying to sit early (using the variable early) gets you turned away with an error. That waiting time is the TDZ.

**Cross-question:** `var` hoisting?

**Cross-answer:**

You get `undefined` until the line that assigns it. That is why `var` bugs are sneaky. With `let`, using the variable too early gives a clear error. With `var`, there is no error at all; the code quietly carries on with `undefined`, and the problem shows up somewhere else later.

---

## Q24. Short-circuit `&&` and `||`?

**Answer:**

`a && b`: if a is truthy, the result is b. `a || b`: if a is falsy, the result is b. For defaults I prefer `??`, so `0` stays `0`.

"Short-circuit" means JavaScript stops as soon as it knows the answer. With `&&`, if `a` is falsy, the result is `a` and `b` is never even looked at. With `||`, if `a` is truthy, the result is `a` and `b` is skipped. That's why people write `user && user.name` (only read the name if there is a user), or `name || 'Guest'` (use 'Guest' if there's no name).

The catch with `||` is that it treats every falsy value as "missing", including `0`, `''` and `false`. `??` only treats `null` and `undefined` as missing, which is usually what I mean by "no value".

**Cross-question:** `value || 10` when value is 0?

**Cross-answer:**

You get 10. That is wrong for something like page size. Use `??`. With `value ?? 10`, a real `0` stays `0`, and only `null` or `undefined` fall back to `10`.

---

## Q25. Why is JS single-threaded, but still “async”?

**Answer:**

There is one call stack. Slow I/O is handed off to the browser or runtime. When it finishes, a callback is queued. So the page stays responsive without extra threads. If I do need extra threads, there are Workers.

A "thread" is one line of work that can run at a time. JavaScript has one, so it can only do one thing at a time. But slow jobs like network requests or timers aren't done by JavaScript itself. The browser (or Node) does that waiting in the background, and when it's done, it puts my callback in the queue to run.

It's like ordering at a café counter: you order, get a buzzer, and step aside so the next person can order. The counter (JavaScript) never stands idle waiting for your coffee. When the buzzer goes off (the callback is queued), you come back. Workers are for heavy calculations that would otherwise freeze the page, because they run code on a separate thread.

**Cross-question:** Does `await` make a new thread?

**Cross-answer:**

No. It just schedules the rest of the function to run later. When the code hits `await`, the function pauses and gives control back, so other code can run on the same single thread. Once the promise settles, the rest of the function is queued and continues from where it left off.


---
## Additional Questions
### Q26. `null` vs `undefined`?
**Answer:** Both mean “no useful value”, but they mean slightly different things. I use null when I set a value to empty on purpose, and undefined when a value or property is missing. The main thing is to be consistent. For example, a "middle name" field can be `null` when I know the person has none, and `undefined` when we never asked. If a project mixes both without a rule, every check has to cover both cases, which is easy to forget.
### Q27. What do `map`, `filter`, and `reduce` do?
**Answer:** map transforms each item, filter keeps the items that match, and reduce combines the items into one result, like a total or an object. Take a shopping cart: `map` can turn each item into its price, `filter` can keep only the items that are in stock, and `reduce` can add the prices up into one bill. `map` and `filter` give back a new array and leave the original alone, which is why they're safe to use for screen data.
### Q28. What is optional chaining?
**Answer:** `user?.profile?.name` stops and returns undefined if an earlier value is null or undefined. It saves me from writing a lot of defensive if statements. Without it, I'd write `user && user.profile && user.profile.name`, or the code would crash with "cannot read properties of undefined" when data hasn't loaded yet. It's common when showing API data that may be missing some parts.
