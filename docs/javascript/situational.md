# JavaScript — Situational questions

Browser and language traps. Each one has the same shape: answer, code, and a real product story.

---

## Q1. You pass `obj.save` as a callback and `this` is wrong. How do you fix it?

**Answer:**

The method wasn't called as a method, so `this` is not `obj` anymore. I fix it by binding it, wrapping the call in an arrow function, or making the method an arrow class field.

In JavaScript, `this` is decided by how a function is called, not where it was written. When I write `obj.save()`, the dot tells JavaScript "`this` is `obj`". But when I hand `obj.save` to something else, like an event listener, I'm only passing the function itself. The object is left behind. Later the browser calls it on its own, so `this` becomes the button (for DOM events) or `undefined` (in strict mode). Then any line like `this.api` blows up.

It's like giving someone a house key without telling them which house it's for. The key is fine, but they don't know where to use it. `bind` and the arrow wrapper both "attach the address": `bind` makes a new function with `this` fixed for good, and an arrow function keeps `this` from the place where it was written.

**Example:**

```javascript
button.addEventListener('click', () => incident.save());
// or
button.addEventListener('click', incident.save.bind(incident));
```

**Real-world example:**

We had a “Close ticket” button that called `this.api` and crashed, because `this` was the button. An arrow wrapper kept the component instance as `this`.

**Cross-question:** Bind in an Angular template every digest?

**Cross-answer:**

That creates a new function again and again. I bind once in the class instead.

`bind` always returns a brand-new function. If I call it inside a template, it runs every time Angular checks the view, so I keep making new functions for nothing. It also means the value never looks "the same" twice, which can make child components think their input changed. Binding once in the class (or using an arrow class field) gives me one stable function I can reuse.

---

## Q2. `for (var i = 0; i < 3; i++) setTimeout(() => console.log(i))` prints 3,3,3. Fix?

**Answer:**

With `var` there is only one `i` for the whole loop, so every callback sees the final value. I switch to `let`, which gives each loop pass its own `i`.

Here is what happens step by step. The loop runs three times very quickly and schedules three callbacks with `setTimeout`. None of them run yet; they wait until the loop is finished. By then the single shared `i` has gone 0, 1, 2 and finally 3, which is when the loop stops. Now the three callbacks run, all look at the same `i`, and all print 3.

With `let`, JavaScript makes a fresh `i` for each pass of the loop. Each callback remembers its own copy (this "remembering" is called a closure), so they print 0, 1, 2. Think of `var` as one whiteboard that everyone reads at the end, and `let` as giving each person their own sticky note with the number written at that moment.

**Example:**

```javascript
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
```

**Real-world example:**

We attached three download buttons in a loop with `var index`, and every button downloaded the last file. Using `let` (or closing over `id`) fixed it.

**Cross-question:** `forEach` with `var`?

**Cross-answer:**

The `forEach` callback gets its own parameter on each call, so it's usually fine. The classic bug is `for` plus `var`.

With `forEach`, each item is passed into a new function call, and every call has its own copy of the parameter. So there's no single shared variable for the callbacks to fight over. The problem only comes back if the callback reads a `var` declared outside it that keeps changing.

---

## Q3. Search fires on every key and the slow response wins. Same as Angular `switchMap` — how in plain JS?

**Answer:**

I debounce the keystrokes, and I abort the previous `fetch` with `AbortController` before starting a new one.

There are two problems here. First, sending a request on every key press is wasteful. Debouncing means "wait until the user stops typing for a moment (here 300 ms), then search once". It's like a lift that waits a few seconds for more people before closing the doors, instead of leaving for each person.

Second, network replies don't always come back in the order they were sent. If the request for "inc" is slow and the one for "incident" is fast, the old "inc" results can arrive last and overwrite the correct ones. `AbortController` lets me cancel the older request, so only the latest search can update the screen. That's exactly what `switchMap` does in Angular: it drops the old request when a new one starts.

**Example:**

```javascript
let t;
let ctrl;
input.addEventListener('input', () => {
  clearTimeout(t);
  t = setTimeout(async () => {
    ctrl?.abort();
    ctrl = new AbortController();
    const res = await fetch('/api/search?q=' + encodeURIComponent(input.value), {
      signal: ctrl.signal,
    });
    const data = await res.json();
    render(data);
  }, 300);
});
```

**Real-world example:**

This was the global search in the header. If you typed “inc” and then “incident”, the `inc` results showed up last. With abort plus debounce, the results matched what was in the box.

**Cross-question:** Ignore abort errors?

**Cross-answer:**

Yes. An `AbortError` is normal here, so I don't show it as a failure toast.

When I cancel a `fetch`, the old promise rejects with an `AbortError`. That's not a real failure; I asked for it. So in the `catch` I check the error name, skip `AbortError` quietly, and still show a message for real errors like a network failure or a 500.

---

## Q4. `0.1 + 0.2 === 0.3` is false. How do you store money?

**Answer:**

I don't use floats for money. I store integer cents, or use a decimal library. When I compare, I compare rounded cent values.

JavaScript numbers are binary floating-point numbers. Many simple decimals, like 0.1, can't be stored exactly in binary, just like 1/3 can't be written exactly in decimal (0.3333...). So `0.1 + 0.2` gives `0.30000000000000004`, and the comparison with `0.3` is false. For most things that tiny error doesn't matter, but with money it shows up as totals that are off by a cent.

Whole numbers (up to a very large limit) are stored exactly. So if I keep ₹10.50 as `1050` paise, or $0.30 as `30` cents, all my adding and comparing is exact. I only divide by 100 at the very end, to show the amount on screen.

**Example:**

```javascript
const cents = 10 + 20;
const display = (cents / 100).toFixed(2);
```

**Real-world example:**

Our invoice totals were off by one cent. Now the API sends `amountCents`, and Angular only formats it for the screen.

**Cross-question:** C# `long` id in JS?

**Cross-answer:**

It's the same family of bug: JS numbers are doubles, and doubles lose precision on big integers. So I send ids as strings.

A JS number can only hold whole numbers exactly up to `Number.MAX_SAFE_INTEGER` (9,007,199,254,740,991). A C# `long` can go much higher. If the API sends a bigger id as a JSON number, JavaScript quietly rounds it, and I end up asking for a record that doesn't exist. Sending the id as a string keeps every digit. I don't do maths on ids anyway, so a string is fine.

---

## Q5. `JSON.parse` of user text then `el.innerHTML = obj.html`. What is the risk?

**Answer:**

The risk is XSS. `JSON.parse` doesn't run code, but putting that HTML into the DOM can. I use `textContent` or Angular interpolation instead. I never use `innerHTML` for user content.

XSS (cross-site scripting) means an attacker gets their own script to run inside my page, in another user's browser. That script can read what the page can read, like tokens, and act as that user. `JSON.parse` only turns text into an object, so it's harmless. The danger is the next step: `innerHTML` tells the browser "treat this text as HTML", so tags inside it become real elements, including things like an image with an `onerror` handler that runs code.

`textContent` does the opposite: it shows the text exactly as typed, so `<script>` appears on screen as plain characters. It's the difference between reading a note aloud and doing whatever the note says.

**Example:**

```javascript
el.textContent = user.name; // safe text
```

**Real-world example:**

An incident title with `<script>` in it showed up as plain text in Angular `{{ title }}`, which is safe. But an old jQuery toast used `innerHTML`, and the script ran there. We switched it to text.

**Cross-question:** Markdown preview?

**Cross-answer:**

I sanitize it with an allow-list. No raw HTML from the user.

Markdown has to become HTML to be shown, so I can't just use plain text. After converting, I run the result through a sanitizer that keeps only safe tags I list (like bold, lists and links) and strips everything else, such as scripts and event attributes. An allow-list is safer than a block-list, because anything I forgot to mention is removed by default.

---

## Q6. `Promise.all` of three APIs — one 404 fails the page. User still needs the other two. What?

**Answer:**

I use `Promise.allSettled` and check the `status` of each call.

`Promise.all` is all-or-nothing: if any one call fails, the whole thing fails straight away and I lose the results that did work. `Promise.allSettled` waits for every call to finish, success or failure, and gives me one result per call. Each result has `status: 'fulfilled'` with a `value`, or `status: 'rejected'` with a `reason`.

So I can show the parts that loaded and show a small message or hide the part that failed. It's like a restaurant bringing the two dishes that are ready instead of cancelling the whole order because one item is out of stock.

**Example:**

```javascript
const [a, b, c] = await Promise.allSettled([
  fetchCounts(),
  fetchList(),
  fetchFlags(),
]);
```

**Real-world example:**

Our home dashboard loaded KPIs, a list, and feature flags. On a new environment the flags endpoint returned 404, and with `all` the whole home page went blank. With `allSettled` we showed the KPIs and the list and just hid the flags.

**Cross-question:** `all` when all three are required?

**Cross-answer:**

Then failing fast is the right behavior. For something like login + tenant + permissions, `all` is fine.

If the page can't work without every piece, there's no point waiting for the others once one has failed. `Promise.all` stops at the first error, so I can show a clear error or send the user back to login right away, instead of showing a half-working screen.

---

## Q7. Copy with `{ ...user }` then change `copy.address.city`. The original user changes too. Why?

**Answer:**

Spread only makes a shallow copy, so nested objects are still shared. I copy the nested part too, or use `structuredClone`.

A shallow copy copies the top level only. For `address`, which is an object, it doesn't copy the object; it copies a reference to it, like copying someone's home address onto a new card. Both cards still point to the same house. So when I change `copy.address.city`, I'm changing the one address object that `user` also points to.

To fix it, I make a new `address` object as well (like in the example), or I use `structuredClone`, which copies every level. That way the copy and the original are truly separate.

**Example:**

```javascript
const copy = { ...user, address: { ...user.address, city: 'Pune' } };
```

**Real-world example:**

Our edit-profile form changed the user in the Angular store, because the form held the same `address` object. After save-cancel, the header already showed Pune as the city. We made a deep copy when editing started and wrote back only on save.

**Cross-question:** OnPush did not update?

**Cross-answer:**

If you only change nested fields, the top-level reference stays the same, so OnPush doesn't see a change. I replace the object I bind to.

With OnPush, Angular checks an input only by asking "is this the same object as before?". It doesn't look inside it. If I change `user.address.city` in place, `user` is still the same object, so Angular skips the update. When I create a new object with the change, the reference is different, and Angular re-renders.

---

## Q8. CORS error in Chrome, Postman works. What do you tell the interviewer?

**Answer:**

Postman is not a browser, and CORS is enforced by the browser. So the **API** has to allow the Angular origin. I can't fix CORS inside Angular, except with a **dev proxy**.

An origin is the scheme, host and port of a site, like `http://localhost:4200`. By default, a browser blocks a page from reading responses from a different origin. CORS (Cross-Origin Resource Sharing) is how the server says "I allow this origin", using response headers. The browser checks those headers and blocks the response if the origin isn't allowed. Postman doesn't do this check at all, so the same call works there.

That's why the fix belongs on the API: it has to list the Angular app's origin. In development, a proxy is a handy shortcut. Angular's dev server forwards `/api` calls to the real API, so from the browser's point of view everything is on one origin and CORS never comes up.

**Example:**

`proxy.conf.json` in dev:

```json
{ "/api": { "target": "https://localhost:5001", "secure": false } }
```

Angular then calls `/api/...` on the same origin.

**Real-world example:**

A new developer said, “The API is down.” The Network tab showed a CORS error. We added the origin on the API and set up a proxy for local work.

**Cross-question:** Disable CORS in the browser for production?

**Cross-answer:**

No. That's not a fix you can ship to users.

Turning off browser security only works on my own machine. Real users run normal browsers, so they'd still get the error. And the rule exists to protect them, so asking anyone to switch it off would be a bad idea anyway. The real fix is the correct CORS settings on the API.

---

## Q9. `forEach(async item => await save(item))` returns before saves finish. Why?

**Answer:**

Because `forEach` doesn't wait for async callbacks. I use `for...of` with `await` to save one by one, or `Promise.all` to run them in parallel.

An `async` function returns a promise straight away and finishes its work later. `forEach` just calls the callback for each item and ignores what it returns, so it throws all those promises away and moves on. The `await` inside the callback only pauses that one callback, not the code that called `forEach`. So the next line runs while the saves are still in progress.

With `for...of`, the `await` is in my own function, so it really pauses until each save finishes. `Promise.all` starts all saves together and gives me one promise I can wait on until every save is done.

**Example:**

```javascript
for (const item of items) {
  await save(item); // one after another
}
await Promise.all(items.map(save)); // parallel
```

**Real-world example:**

“Close all selected” returned 200 while the saves were still running. The UI navigated away and some closes never happened. Switching to `for...of` with `await` made it actually wait.

**Cross-question:** Parallel on a DbContext in C#?

**Cross-answer:**

I don't do that, because a DbContext is not thread-safe. In JS, `Promise.all` over HTTP calls is OK if the API can handle it.

A `DbContext` is EF Core's single unit of work with the database, and it's built for one operation at a time. If two queries run on it at once, EF Core throws an error. In the browser it's different: each HTTP call is separate, so sending a few in parallel is fine, as long as I don't flood the server with hundreds at once.

---

## Q10. A click on a row and a click on a button inside the row both fire. How do you stop the row?

**Answer:**

The button's click event bubbles up to the row. I call `stopPropagation` in the button handler, or check `event.target` in the row handler.

"Bubbling" means that when I click an element, the click event first fires on that element and then travels up through each parent. The button sits inside the row, so the same click reaches the row's handler too. It's like knocking on an inner door in a house: people in the outer room hear it as well.

`stopPropagation()` stops the event at the button, so the row never hears it. The other way is to let it bubble and have the row handler check `event.target` (the element that was actually clicked) and do nothing if it was the button.

**Example:**

```html
<tr (click)="open(row)">
  <button (click)="delete(row); $event.stopPropagation()">Delete</button>
</tr>
```

**Real-world example:**

In our incident table, clicking a row opened the detail page. Clicking Delete opened the detail **and** deleted the row. Adding `$event.stopPropagation()` on Delete fixed it.

**Cross-question:** `preventDefault` instead?

**Cross-answer:**

That stops the browser's default action, like a form submit or following a link. To stop bubbling you need `stopPropagation`.

They solve different problems. `preventDefault()` says "don't do your usual thing" to the browser, but the event still travels up to the parents. `stopPropagation()` says "don't tell the parents", but the browser's usual action still happens. Sometimes I need both, for example a link inside a clickable row.


---
## Additional Questions
### Q11. A page becomes slow after opening and closing it many times. What do you check?
**Answer:** I look for subscriptions, timers, event listeners and global references that stay alive after the component is destroyed. Chrome memory snapshots can show whether objects keep growing.

This pattern usually means a memory leak. When I leave a page, its component should be thrown away. But if something still holds on to it, like an RxJS subscription I never unsubscribed, a `setInterval` I never cleared, or a `window` listener I never removed, the browser can't free that memory. Every visit adds another copy, and those old copies may even keep running code in the background. It's like leaving a tap running in every room you walk out of.

To prove it, I open the page, close it, take a heap snapshot in Chrome DevTools, repeat a few times and compare. If the number of my component objects keeps going up, something is holding them. The fix is to clean up when the component is destroyed (for example in `ngOnDestroy`, or with `takeUntilDestroyed` in Angular).
### Q12. An API response arrives out of order. How do you prevent stale data?
**Answer:** For search, I cancel or ignore the older requests. In RxJS I use switchMap. With fetch I can use AbortController or check a request version number.

Requests can finish in any order, because one may hit a slower server or a bigger result. If I just show whatever arrives last, an old answer can overwrite a newer one, and the screen no longer matches what the user asked for. That's "stale data".

`switchMap` cancels the previous request when a new one starts. With plain `fetch`, `AbortController` does the cancelling. The version-number idea is simple too: I give each request a number, remember the latest one, and when a reply comes back I only use it if its number is still the latest. It's like only answering the most recent text message in a chat and ignoring older ones that arrive late.
### Q13. A button creates duplicate records when clicked twice. What do you do?
**Answer:** I disable the button or ignore clicks while the request is running, but I also make the API idempotent where duplicates matter. Protecting only the client is not enough.

Users double-click, networks are slow, and browsers or proxies can retry. Disabling the button while the request runs stops most accidental repeats and shows the user that something is happening. But the browser isn't fully under my control: a second tab, a retry, or a script can still send the same request twice.

Idempotent means "doing it twice has the same effect as doing it once". A common way is for the client to send a unique key with the request, and the server remembers keys it has already handled. If the same key comes again, it returns the first result instead of creating a second record. This matters most for things like payments and orders, where a duplicate costs real money.
