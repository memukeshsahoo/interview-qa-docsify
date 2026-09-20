# JavaScript — Situational questions

Browser and language traps. Same shape: answer, code, real product story.

---

## Q1. You pass `obj.save` as a callback and `this` is wrong. How do you fix it?

**Answer:**

The method was not called as a method, so `this` is not `obj`. Bind it, wrap in an arrow, or use an arrow class field.

**Example:**

```javascript
button.addEventListener('click', () => incident.save());
// or
button.addEventListener('click', incident.save.bind(incident));
```

**Real-world example:**

A “Close ticket” button called `this.api` and crashed because `this` was the button. An arrow wrapper kept the component instance.

**Cross-question:** Bind in an Angular template every digest?

**Cross-answer:**

Creates a new function often. Bind once in the class.

---

## Q2. `for (var i = 0; i < 3; i++) setTimeout(() => console.log(i))` prints 3,3,3. Fix?

**Answer:**

One `var i` for the loop. Use `let`.

**Example:**

```javascript
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
```

**Real-world example:**

Attaching three download buttons in a loop with `var index` — every button downloaded the last file. `let` (or close over `id`) fixed it.

**Cross-question:** `forEach` with `var`?

**Cross-answer:**

`forEach` callback has its own parameter. Usually fine. The classic bug is `for` + `var`.

---

## Q3. Search fires on every key and the slow response wins. Same as Angular `switchMap` — how in plain JS?

**Answer:**

Debounce the keys. Abort the previous `fetch` with `AbortController`.

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

Global search in the header. Typing “inc” then “incident” showed `inc` results last. Abort + debounce matched the box.

**Cross-question:** Ignore abort errors?

**Cross-answer:**

Yes, `AbortError` is normal. Do not toast it as a failure.

---

## Q4. `0.1 + 0.2 === 0.3` is false. How do you store money?

**Answer:**

Do not use float money. Integer cents, or a decimal library. Compare with a rounded cent value.

**Example:**

```javascript
const cents = 10 + 20;
const display = (cents / 100).toFixed(2);
```

**Real-world example:**

Invoice totals were off by one cent. API now sends `amountCents`. Angular only formats for the screen.

**Cross-question:** C# `long` id in JS?

**Cross-answer:**

Same family of bug: doubles lose big integers. Send ids as strings.

---

## Q5. `JSON.parse` of user text then `el.innerHTML = obj.html`. What is the risk?

**Answer:**

XSS. `JSON.parse` does not run code, but putting HTML into the DOM does. Use textContent or Angular interpolation. Never `innerHTML` for user content.

**Example:**

```javascript
el.textContent = user.name; // safe text
```

**Real-world example:**

Incident title with `<script>` showed as text in Angular `{{ title }}` (safe). An old jQuery toast used `innerHTML` and the script ran. We switched to text.

**Cross-question:** Markdown preview?

**Cross-answer:**

Sanitize with an allow-list. No raw HTML from the user.

---

## Q6. `Promise.all` of three APIs — one 404 fails the page. User still needs the other two. What?

**Answer:**

`Promise.allSettled`. Read `status` per call.

**Example:**

```javascript
const [a, b, c] = await Promise.allSettled([
  fetchCounts(),
  fetchList(),
  fetchFlags(),
]);
```

**Real-world example:**

Home dashboard: KPIs, list, feature flags. Flags endpoint 404 on a new env. `all` blanked the whole home. `allSettled` showed KPIs and list, hid the flags.

**Cross-question:** `all` when all three are required?

**Cross-answer:**

Then fail fast is correct. Login + tenant + permissions: `all` is fine.

---

## Q7. Copy with `{ ...user }` then change `copy.address.city`. The original user changes too. Why?

**Answer:**

Spread is shallow. Nested objects are shared. Copy the nested part too, or `structuredClone`.

**Example:**

```javascript
const copy = { ...user, address: { ...user.address, city: 'Pune' } };
```

**Real-world example:**

Edit-profile form mutated the Angular store’s user because the form held the same `address` object. After save-cancel, the header city was already Pune. Deep copy on edit, write back only on save.

**Cross-question:** OnPush did not update?

**Cross-answer:**

Same reference at the top if you only mutate nested fields. Replace the object you bind.

---

## Q8. CORS error in Chrome, Postman works. What do you tell the interviewer?

**Answer:**

Postman is not a browser. CORS is enforced in the browser. The **API** must allow the Angular origin. I cannot fix CORS inside Angular except a **dev proxy**.

**Example:**

`proxy.conf.json` in dev:

```json
{ "/api": { "target": "https://localhost:5001", "secure": false } }
```

Angular calls `/api/...` same origin.

**Real-world example:**

New developer: “API is down.” Network tab: CORS. We added the origin on the API and a proxy for local.

**Cross-question:** Disable CORS in the browser for production?

**Cross-answer:**

No. That is not a product fix.

---

## Q9. `forEach(async item => await save(item))` returns before saves finish. Why?

**Answer:**

`forEach` does not wait for async callbacks.

**Example:**

```javascript
for (const item of items) {
  await save(item); // one after another
}
await Promise.all(items.map(save)); // parallel
```

**Real-world example:**

“Close all selected” returned 200 while saves were still running. The UI navigated away and some closes never happened. `for...of` + `await` made it honest.

**Cross-question:** Parallel on a DbContext in C#?

**Cross-answer:**

Do not; context is not thread-safe. JS `Promise.all` on HTTP is OK if the API allows it.

---

## Q10. A click on a row and a click on a button inside the row both fire. How do you stop the row?

**Answer:**

The button event bubbles. `stopPropagation` on the button handler, or check `event.target`.

**Example:**

```html
<tr (click)="open(row)">
  <button (click)="delete(row); $event.stopPropagation()">Delete</button>
</tr>
```

**Real-world example:**

Incident table: row click opens detail. Delete opened detail **and** deleted. `$event.stopPropagation()` on Delete fixed it.

**Cross-question:** `preventDefault` instead?

**Cross-answer:**

That stops browser default (submit, link). Bubbling is `stopPropagation`.


---
## Additional Questions
### Q11. A page becomes slow after opening and closing it many times. What do you check?
**Answer:** I look for subscriptions, timers, event listeners and global references that survive component destruction. Chrome memory snapshots can show whether objects keep growing.
### Q12. An API response arrives out of order. How do you prevent stale data?
**Answer:** For search I cancel or ignore older requests. In RxJS I use switchMap; in fetch I can use AbortController or a request version check.
### Q13. A button creates duplicate records when clicked twice. What do you do?
**Answer:** Disable or exhaust the UI action, but also make the API idempotent where duplicate creation matters. Client protection alone is not enough.
