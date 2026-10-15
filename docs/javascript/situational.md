# JavaScript — Situational questions

Browser and language traps. Each one has the same shape: answer, code, and a real product story.

---

## Q1. You pass `obj.save` as a callback and `this` is wrong. How do you fix it?

**Answer:**

The method wasn't called as a method, so `this` is not `obj` anymore. I fix it by binding it, wrapping the call in an arrow function, or making the method an arrow class field.

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

---

## Q2. `for (var i = 0; i < 3; i++) setTimeout(() => console.log(i))` prints 3,3,3. Fix?

**Answer:**

With `var` there is only one `i` for the whole loop, so every callback sees the final value. I switch to `let`, which gives each loop pass its own `i`.

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

---

## Q3. Search fires on every key and the slow response wins. Same as Angular `switchMap` — how in plain JS?

**Answer:**

I debounce the keystrokes, and I abort the previous `fetch` with `AbortController` before starting a new one.

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

---

## Q4. `0.1 + 0.2 === 0.3` is false. How do you store money?

**Answer:**

I don't use floats for money. I store integer cents, or use a decimal library. When I compare, I compare rounded cent values.

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

---

## Q5. `JSON.parse` of user text then `el.innerHTML = obj.html`. What is the risk?

**Answer:**

The risk is XSS. `JSON.parse` doesn't run code, but putting that HTML into the DOM can. I use `textContent` or Angular interpolation instead. I never use `innerHTML` for user content.

**Example:**

```javascript
el.textContent = user.name; // safe text
```

**Real-world example:**

An incident title with `<script>` in it showed up as plain text in Angular `{{ title }}`, which is safe. But an old jQuery toast used `innerHTML`, and the script ran there. We switched it to text.

**Cross-question:** Markdown preview?

**Cross-answer:**

I sanitize it with an allow-list. No raw HTML from the user.

---

## Q6. `Promise.all` of three APIs — one 404 fails the page. User still needs the other two. What?

**Answer:**

I use `Promise.allSettled` and check the `status` of each call.

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

---

## Q7. Copy with `{ ...user }` then change `copy.address.city`. The original user changes too. Why?

**Answer:**

Spread only makes a shallow copy, so nested objects are still shared. I copy the nested part too, or use `structuredClone`.

**Example:**

```javascript
const copy = { ...user, address: { ...user.address, city: 'Pune' } };
```

**Real-world example:**

Our edit-profile form changed the user in the Angular store, because the form held the same `address` object. After save-cancel, the header already showed Pune as the city. We made a deep copy when editing started and wrote back only on save.

**Cross-question:** OnPush did not update?

**Cross-answer:**

If you only change nested fields, the top-level reference stays the same, so OnPush doesn't see a change. I replace the object I bind to.

---

## Q8. CORS error in Chrome, Postman works. What do you tell the interviewer?

**Answer:**

Postman is not a browser, and CORS is enforced by the browser. So the **API** has to allow the Angular origin. I can't fix CORS inside Angular, except with a **dev proxy**.

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

---

## Q9. `forEach(async item => await save(item))` returns before saves finish. Why?

**Answer:**

Because `forEach` doesn't wait for async callbacks. I use `for...of` with `await` to save one by one, or `Promise.all` to run them in parallel.

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

---

## Q10. A click on a row and a click on a button inside the row both fire. How do you stop the row?

**Answer:**

The button's click event bubbles up to the row. I call `stopPropagation` in the button handler, or check `event.target` in the row handler.

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


---
## Additional Questions
### Q11. A page becomes slow after opening and closing it many times. What do you check?
**Answer:** I look for subscriptions, timers, event listeners and global references that stay alive after the component is destroyed. Chrome memory snapshots can show whether objects keep growing.
### Q12. An API response arrives out of order. How do you prevent stale data?
**Answer:** For search, I cancel or ignore the older requests. In RxJS I use switchMap. With fetch I can use AbortController or check a request version number.
### Q13. A button creates duplicate records when clicked twice. What do you do?
**Answer:** I disable the button or ignore clicks while the request is running, but I also make the API idempotent where duplicates matter. Protecting only the client is not enough.
