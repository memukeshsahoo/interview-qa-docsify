# Angular — Intermediate

RxJS, forms, routing, speed.

---

## Q1. Observable vs Promise vs Signal?

**Answer:**

Promise: one value, starts now, hard to cancel.

Observable: zero or more values over time, starts when you subscribe, can cancel.

Signal: a value Angular can track in the template. Great for local state. HTTP is often still an Observable, then `toSignal()`.

**Cross-question:** Can I `await` an Observable?

**Cross-answer:**

Not directly. `firstValueFrom` subscribes. I prefer async pipe or signals so cancel still works.

---

## Q2. Cold vs hot Observable?

**Answer:**

Cold: each subscriber starts new work. `HttpClient.get` — two subscribers, two calls.

Hot: already running. `Subject`. `shareReplay(1)` shares one HTTP result.

**Cross-question:** API fired twice, once in `ngOnInit` and once in the template?

**Cross-answer:**

You subscribed and also used `async` pipe. Pick one.

---

## Q3. Operators you should name?

**Answer:**

`map`, `filter`, `tap`.

`switchMap`: cancel the old call (search box).

`concatMap`: one after another.

`exhaustMap`: ignore new until current finishes (save button).

`debounceTime`, `catchError`, `takeUntilDestroyed`.

**Cross-question:** Typeahead: switchMap or mergeMap?

**Cross-answer:**

`switchMap`. An old slow search should not overwrite a new one.

---

## Q4. Subject vs BehaviorSubject?

**Answer:**

Subject: no last value. Late subscribers miss it.

BehaviorSubject: always has a current value. New subscribers get it now. Good for “current user”.

Keep the subject private. Expose `asObservable()` or a signal.

**Cross-question:** ReplaySubject?

**Cross-answer:**

Replays the last N values to new subscribers.

---

## Q5. Guards?

**Answer:**

`canActivate`: may I open this page?

`canMatch`: does this route even exist for me?

`canDeactivate`: unsaved changes.

**Cross-question:** Is a guard security?

**Cross-answer:**

No. It only blocks navigation. The API must still check.

---

## Q6. Lazy load and preload?

**Answer:**

Lazy: load on navigation. Preload: download extra chunks after the first page is up. Faster later, more bandwidth now.

**Cross-question:** Preload everything?

**Cross-answer:**

Fine on desktop intranet. On mobile I preload only the common areas.

---

## Q7. `patchValue` vs `setValue`? Disabled controls?

**Answer:**

`setValue` needs every key. `patchValue` is partial. Disable in TypeScript with `control.disable()`, not `[disabled]` on a reactive control. Disabled fields are missing from `.value`. Use `getRawValue()` if you still need them.

**Cross-question:** FormArray?

**Cross-answer:**

A list of controls, like extra phone numbers. I push and remove controls as the user adds rows.

---

## Q8. Async validator?

**Answer:**

A function that returns an Observable of errors or null. Unique email check. Debounce so we do not hit the API every key.

**Cross-question:** Form stuck `pending`?

**Cross-answer:**

The async validator never completed. I must complete or error the Observable.

---

## Q9. OnPush and mutation?

**Answer:**

If I do `user.name = 'A'` and `user` is the same object, OnPush may not update. I replace the object or I use a signal.

**Cross-question:** Signals plus OnPush?

**Cross-answer:**

A good default for lists. Signals tell Angular exactly what changed.

---

## Q10. Interceptor refresh race?

**Answer:**

Two 401s should not start two refresh calls. Share one refresh. If refresh fails, log out.

**Cross-question:** Add the token to every HTTP call on the internet?

**Cross-answer:**

No. Only my API origin.

---

## Q11. NgRx or a service?

**Answer:**

A service plus signals is enough for most apps. NgRx helps when many screens share complex events. I do not add NgRx for six CRUD pages.

**Cross-question:** Overkill sign?

**Cross-answer:**

More boilerplate than features. Actions for every keystroke. The team fights the store more than the product.

---

## Q12. Content projection?

**Answer:**

`<ng-content>` puts parent HTML into the child. Named slots: `select="[actions]"`. Templates (`ng-template`) for optional layouts.

**Cross-question:** Projection vs `@Input()`?

**Cross-answer:**

Input is data. Projection is markup. A card title as input is fine. A whole toolbar as projection is nicer.

---

## Q13. Dynamic component vs `@if`?

**Answer:**

A few variants: `@if` or `NgComponentOutlet`. A widget host: `createComponent`. I prefer templates when I can.

**Cross-question:** How do you set inputs on a dynamic component?

**Cross-answer:**

`setInput('title', value)` so Angular runs the input system. Do not only poke the field.

---

## Q14. Performance checklist?

**Answer:**

Track by id. OnPush or signals. Lazy routes. Virtual scroll for huge lists. No heavy work in the template. `NgOptimizedImage`.

**Cross-question:** Why virtual scroll?

**Cross-answer:**

10,000 real rows is slow. We only render what you see plus a little buffer.

---

## Q15. How do you test a component?

**Answer:**

`TestBed`, click, expect text or an output. Fake HTTP with `HttpTestingController`. I test behavior, not private methods.

**Cross-question:** Shallow vs deep?

**Cross-answer:**

Shallow mocks children. Faster. I still want a few tests with real children for a tricky widget.

---

## Q16. `switchMap` vs `exhaustMap` on save?

**Answer:**

Save button: `exhaustMap` so a double click does not fire two saves. Search: `switchMap`.

**Cross-question:** `mergeMap` on save?

**Cross-answer:**

Two overlapping saves. Race. Usually wrong.

---

## Q17. Route params — snapshot vs paramMap Observable?

**Answer:**

Snapshot is fine if the component is created each time. If the same component is reused for `/user/1` then `/user/2`, I must listen to `paramMap`.

**Cross-question:** Resolver?

**Cross-answer:**

Loads data before the component. Less common now. Fine if you really need data before first paint.

---

## Q18. `ChangeDetectorRef.markForCheck` vs `detectChanges`?

**Answer:**

`markForCheck` flags the component for the next cycle (OnPush). `detectChanges` runs now. I use markForCheck more. detectChanges in tests is common.

**Cross-question:** Call detectChanges in production code a lot?

**Cross-answer:**

A smell. Data should flow so Angular sees it.

---

## Q19. i18n?

**Answer:**

Built-in i18n: build per locale. Libraries like ngx-translate: switch language at runtime. Dates use locale pipes, not hand-built strings.

**Cross-question:** Hard-coded English in templates?

**Cross-answer:**

Fine for an internal English-only app. A product for many countries needs a plan early. Retrofit is painful.

---

## Q20. How do you share a header user name across pages?

**Answer:**

An auth service with a signal or BehaviorSubject. Header reads it. Login sets it. Logout clears it. I do not copy the name into ten components.

**Cross-question:** Refresh the page — user gone?

**Cross-answer:**

I restore from a session cookie/token on startup (`APP_INITIALIZER` or an auth guard that waits).

---

## Q21. `providedIn: 'root'` vs route providers?

**Answer:**

Root: one instance. Route providers: live as long as that route. Good for a feature store you want to drop when you leave.

**Cross-question:** Two “singletons”?

**Cross-answer:**

`providedIn: 'any'` in lazy modules used to do that. I stick to root unless I have a reason.

---

## Q22. What is a pure pipe vs impure?

**Answer:**

Pure: run when the input reference changes. Impure: run often (like `async` pipe). I write pure pipes unless I must not.

**Cross-question:** Pipe that sorts an array in place?

**Cross-answer:**

Bad. It mutates. Return a new array.

---

## Q23. How do you cancel an HTTP call on leave?

**Answer:**

`takeUntilDestroyed`, async pipe, or `switchMap` from the route. If I `subscribe` by hand and forget, the call can finish and set state on a dead component.

**Cross-question:** Memory leak sign?

**Cross-answer:**

`subscribe` in `ngOnInit` with no teardown. I look for that in reviews.

---

## Q24. Reactive forms: `valueChanges`?

**Answer:**

An Observable of the form value. I debounce, then search. I unsubscribe with `takeUntilDestroyed`.

**Cross-question:** `updateOn: 'blur'`?

**Cross-answer:**

Validate when the user leaves the field, not on every key. Better UX for email.

---

## Q25. What is `NgZone.run` for?

**Answer:**

Third-party code ran outside Angular, UI did not update. `run` puts the work back in the zone. With zoneless + signals I notify via signals instead.

**Cross-question:** `runOutsideAngular`?

**Cross-answer:**

For noisy events (mousemove) I do not want 60 change detections a second. I re-enter only when I must update the UI.
