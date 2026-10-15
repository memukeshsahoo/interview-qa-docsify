# Angular — Intermediate

RxJS, forms, routing, speed.

---

## Q1. Observable vs Promise vs Signal?

**Answer:**

A Promise gives you one value. It starts right away, and it is hard to cancel.

An Observable gives you zero or more values over time. It only starts when you subscribe, and you can cancel it.

A Signal is a value that Angular can track in the template. It is great for local state. HTTP calls usually still return an Observable, and then I convert it with `toSignal()`.

**Cross-question:** Can I `await` an Observable?

**Cross-answer:**

Not directly. `firstValueFrom` subscribes for you and gives back a Promise. But I prefer the async pipe or signals, so cancelling still works.

---

## Q2. Cold vs hot Observable?

**Answer:**

A cold Observable starts new work for each subscriber. With `HttpClient.get`, two subscribers means two calls.

A hot Observable is already running, like a `Subject`. `shareReplay(1)` lets several subscribers share one HTTP result.

**Cross-question:** API fired twice, once in `ngOnInit` and once in the template?

**Cross-answer:**

That happens when you subscribe in code and also use the `async` pipe. Pick one.

---

## Q3. Operators you should name?

**Answer:**

`map`, `filter`, `tap`.

`switchMap`: cancels the old call when a new one starts (search box).

`concatMap`: runs them one after another.

`exhaustMap`: ignores new ones until the current one finishes (save button).

`debounceTime`, `catchError`, `takeUntilDestroyed`.

**Cross-question:** Typeahead: switchMap or mergeMap?

**Cross-answer:**

`switchMap`. An old, slow search should not overwrite the result of a newer one.

---

## Q4. Subject vs BehaviorSubject?

**Answer:**

A Subject has no last value, so late subscribers miss what was already sent.

A BehaviorSubject always has a current value, and new subscribers get it straight away. That makes it good for something like “current user”.

I keep the subject private and expose `asObservable()` or a signal.

**Cross-question:** ReplaySubject?

**Cross-answer:**

It replays the last N values to new subscribers.

---

## Q5. Guards?

**Answer:**

`canActivate`: am I allowed to open this page?

`canMatch`: does this route even exist for me?

`canDeactivate`: checks for unsaved changes before leaving.

**Cross-question:** Is a guard security?

**Cross-answer:**

No. It only blocks navigation in the browser. The API must still do its own checks.

---

## Q6. Lazy load and preload?

**Answer:**

Lazy loading loads a part of the app only when you navigate to it. Preloading downloads extra chunks in the background after the first page is up. That makes later pages faster, but uses more bandwidth now.

**Cross-question:** Preload everything?

**Cross-answer:**

That is fine for a desktop intranet app. On mobile, I preload only the areas people use most.

---

## Q7. `patchValue` vs `setValue`? Disabled controls?

**Answer:**

`setValue` needs every key. `patchValue` lets you update just some of them. I disable a control in TypeScript with `control.disable()`, not with `[disabled]` on a reactive control. Disabled fields are left out of `.value`, so I use `getRawValue()` if I still need them.

**Cross-question:** FormArray?

**Cross-answer:**

It is a list of controls, like extra phone numbers. I push and remove controls as the user adds or deletes rows.

---

## Q8. Async validator?

**Answer:**

It is a function that returns an Observable of errors, or null if valid. A typical case is checking that an email is unique. I debounce it so we do not hit the API on every key press.

**Cross-question:** Form stuck `pending`?

**Cross-answer:**

That means the async validator never completed. The Observable has to complete or error.

---

## Q9. OnPush and mutation?

**Answer:**

If I do `user.name = 'A'` and `user` is still the same object, OnPush may not update the view. So I replace the object with a new one, or I use a signal.

**Cross-question:** Signals plus OnPush?

**Cross-answer:**

That is a good default for lists. Signals tell Angular exactly what changed.

---

## Q10. Interceptor refresh race?

**Answer:**

If two requests get a 401 at the same time, they should not start two refresh calls. They should share one refresh. If the refresh fails, I log the user out.

**Cross-question:** Add the token to every HTTP call on the internet?

**Cross-answer:**

No. Only to calls going to my own API origin.

---

## Q11. NgRx or a service?

**Answer:**

For most apps, a service plus signals is enough. NgRx helps when many screens share complex events. I would not add NgRx for six CRUD pages.

**Cross-question:** Overkill sign?

**Cross-answer:**

When there is more boilerplate than features. When there are actions for every keystroke. When the team spends more time fighting the store than building the product.

---

## Q12. Content projection?

**Answer:**

`<ng-content>` takes HTML from the parent and puts it inside the child. For named slots, I use `select="[actions]"`. For optional layouts, I use templates (`ng-template`).

**Cross-question:** Projection vs `@Input()`?

**Cross-answer:**

An input is for data. Projection is for markup. A card title as an input is fine. A whole toolbar works nicer as projection.

---

## Q13. Dynamic component vs `@if`?

**Answer:**

For a few variants, I use `@if` or `NgComponentOutlet`. For a widget host, I use `createComponent`. I prefer templates when I can.

**Cross-question:** How do you set inputs on a dynamic component?

**Cross-answer:**

With `setInput('title', value)`, so Angular goes through its normal input system. Don’t just set the field directly.

---

## Q14. Performance checklist?

**Answer:**

Track list items by id. Use OnPush or signals. Lazy-load routes. Use virtual scroll for huge lists. Keep heavy work out of the template. Use `NgOptimizedImage`.

**Cross-question:** Why virtual scroll?

**Cross-answer:**

Rendering 10,000 real rows is slow. With virtual scroll we only render what you can see, plus a small buffer.

---

## Q15. How do you test a component?

**Answer:**

I use `TestBed`, click things, and expect some text or an output. I fake HTTP with `HttpTestingController`. I test behavior, not private methods.

**Cross-question:** Shallow vs deep?

**Cross-answer:**

Shallow tests mock the child components, so they are faster. But for a tricky widget, I still want a few tests with the real children.

---

## Q16. `switchMap` vs `exhaustMap` on save?

**Answer:**

For a save button, `exhaustMap`, so a double click does not fire two saves. For search, `switchMap`.

**Cross-question:** `mergeMap` on save?

**Cross-answer:**

You get two overlapping saves racing each other. That is usually wrong.

---

## Q17. Route params — snapshot vs paramMap Observable?

**Answer:**

The snapshot is fine if the component is created fresh each time. But if the same component is reused when going from `/user/1` to `/user/2`, I have to listen to `paramMap`.

**Cross-question:** Resolver?

**Cross-answer:**

A resolver loads data before the component is shown. It is less common now, but fine if you really need the data before the first paint.

---

## Q18. `ChangeDetectorRef.markForCheck` vs `detectChanges`?

**Answer:**

`markForCheck` flags the component to be checked in the next cycle (for OnPush). `detectChanges` runs the check right now. I use markForCheck more often. Using detectChanges in tests is common.

**Cross-question:** Call detectChanges in production code a lot?

**Cross-answer:**

That is a code smell. The data should flow in a way that Angular can see on its own.

---

## Q19. i18n?

**Answer:**

With Angular’s built-in i18n, you build the app once per locale. Libraries like ngx-translate let you switch language at runtime. For dates, I use locale-aware pipes, not hand-built strings.

**Cross-question:** Hard-coded English in templates?

**Cross-answer:**

That is fine for an internal, English-only app. But a product for many countries needs a plan early, because adding it later is painful.

---

## Q20. How do you share a header user name across pages?

**Answer:**

I keep it in an auth service, as a signal or a BehaviorSubject. The header reads it, login sets it, and logout clears it. I do not copy the name into ten components.

**Cross-question:** Refresh the page — user gone?

**Cross-answer:**

I restore it from a session cookie/token on startup (`APP_INITIALIZER`, or an auth guard that waits).

---

## Q21. `providedIn: 'root'` vs route providers?

**Answer:**

`root` gives one instance for the whole app. Route providers live only as long as that route. That is good for a feature store you want to throw away when the user leaves.

**Cross-question:** Two “singletons”?

**Cross-answer:**

`providedIn: 'any'` in lazy modules used to do that. I stick to root unless I have a real reason.

---

## Q22. What is a pure pipe vs impure?

**Answer:**

A pure pipe runs only when the input reference changes. An impure pipe runs very often (the `async` pipe is one). I write pure pipes unless I really can’t.

**Cross-question:** Pipe that sorts an array in place?

**Cross-answer:**

That is bad, because it mutates the input. Return a new array instead.

---

## Q23. How do you cancel an HTTP call on leave?

**Answer:**

With `takeUntilDestroyed`, the async pipe, or `switchMap` from the route. If I `subscribe` by hand and forget to clean up, the call can finish later and set state on a component that is already gone.

**Cross-question:** Memory leak sign?

**Cross-answer:**

A `subscribe` in `ngOnInit` with no teardown. I look for that in code reviews.

---

## Q24. Reactive forms: `valueChanges`?

**Answer:**

It is an Observable of the form’s value. I debounce it, then run the search. I unsubscribe with `takeUntilDestroyed`.

**Cross-question:** `updateOn: 'blur'`?

**Cross-answer:**

It validates when the user leaves the field, not on every key press. That is a better experience for something like email.

---

## Q25. What is `NgZone.run` for?

**Answer:**

When third-party code runs outside Angular, the UI does not update. `run` brings that work back into Angular’s zone so change detection happens. With zoneless Angular and signals, I notify through signals instead.

**Cross-question:** `runOutsideAngular`?

**Cross-answer:**

I use it for noisy events like mousemove, where I do not want 60 change detections a second. I only step back into Angular when I actually need to update the UI.


---
## Additional Questions
### Q26. `switchMap`, `concatMap`, `mergeMap`, `exhaustMap` in one example?
**Answer:** Search uses switchMap, saves that must stay in order can use concatMap, independent parallel work can use mergeMap, and a login or submit button often uses exhaustMap to ignore duplicate clicks.
### Q27. How do you handle multiple 401 responses?
**Answer:** I share one refresh request. The other failed requests wait for it, then retry with the new token. If the refresh fails, I clear the auth state and send the user to login.
### Q28. How do you build a dynamic FormArray?
**Answer:** I create a FormArray, push a FormGroup for each row, and remove that group by index when the user deletes a row. The API still validates every row.
