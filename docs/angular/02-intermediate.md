# Angular — Intermediate

RxJS, forms, routing, speed.

---

## Q1. Observable vs Promise vs Signal?

**Answer:**

A Promise gives you one value. It starts right away, and it is hard to cancel.

An Observable gives you zero or more values over time. It only starts when you subscribe, and you can cancel it.

A Signal is a value that Angular can track in the template. It is great for local state. HTTP calls usually still return an Observable, and then I convert it with `toSignal()`.

An everyday way to picture them: a Promise is like ordering a parcel. One delivery comes, and once it's on its way you can't really stop it. An Observable is like a newspaper subscription: nothing arrives until you sign up, papers keep coming, and you can cancel any time. A Signal is like a scoreboard: it always shows the current value, and everyone watching sees it change.

**Cross-question:** Can I `await` an Observable?

**Cross-answer:**

Not directly. `firstValueFrom` subscribes for you and gives back a Promise. But I prefer the async pipe or signals, so cancelling still works. Once it's turned into a Promise, I can't cancel it if the user leaves the page.

---

## Q2. Cold vs hot Observable?

**Answer:**

A cold Observable starts new work for each subscriber. With `HttpClient.get`, two subscribers means two calls.

A hot Observable is already running, like a `Subject`. `shareReplay(1)` lets several subscribers share one HTTP result.

A cold Observable is like a video on demand: each viewer presses play and gets their own copy from the start. A hot Observable is like live TV: it's already playing, and if you tune in late you miss what already happened. This matters because a cold HTTP Observable that gets subscribed twice quietly sends two requests to the server.

**Cross-question:** API fired twice, once in `ngOnInit` and once in the template?

**Cross-answer:**

That happens when you subscribe in code and also use the `async` pipe. Each one is a separate subscription, and each subscription to a cold HTTP Observable sends its own request. Pick one.

---

## Q3. Operators you should name?

**Answer:**

Operators are small functions you chain onto an Observable to change, filter or control the values that come through it, like steps on an assembly line.

`map`, `filter`, `tap`.

`switchMap`: cancels the old call when a new one starts (search box).

`concatMap`: runs them one after another.

`exhaustMap`: ignores new ones until the current one finishes (save button).

`debounceTime`, `catchError`, `takeUntilDestroyed`.

In plain words: `map` changes each value, `filter` lets only some values through, and `tap` lets me do something on the side (like logging) without changing the value. `debounceTime` waits until the user stops typing for a moment, `catchError` handles a failure so the stream doesn't just die, and `takeUntilDestroyed` stops the stream when the component goes away.

**Cross-question:** Typeahead: switchMap or mergeMap?

**Cross-answer:**

`switchMap`. An old, slow search should not overwrite the result of a newer one.

If the user types "ang" and then "angular", the "ang" request might come back last. With `mergeMap`, both results arrive and the stale one can win. `switchMap` cancels the "ang" request as soon as "angular" starts, so only the latest search can show.

---

## Q4. Subject vs BehaviorSubject?

**Answer:**

A Subject is something I can push values into by hand, and everyone subscribed gets them.

A Subject has no last value, so late subscribers miss what was already sent.

A BehaviorSubject always has a current value, and new subscribers get it straight away. That makes it good for something like “current user”. A header that loads after login still needs to know who is logged in, and a plain Subject would leave it with nothing until the next change.

I keep the subject private and expose `asObservable()` or a signal. That way, only the service can change the value, and other parts of the app can only read it. It stops random components from pushing values in and makes bugs easier to trace.

**Cross-question:** ReplaySubject?

**Cross-answer:**

It replays the last N values to new subscribers. So a late subscriber gets a short history, not just the latest value. Unlike a BehaviorSubject, it doesn't need a starting value.

---

## Q5. Guards?

**Answer:**

A guard is a check the router runs before it lets the user go to, or leave, a page.

`canActivate`: am I allowed to open this page?

`canMatch`: does this route even exist for me?

`canDeactivate`: checks for unsaved changes before leaving.

For example, `canActivate` sends a logged-out user to the login page. `canMatch` can make an admin route act as if it isn't there for normal users, and it also stops that lazy-loaded code from being downloaded for them. `canDeactivate` shows "You have unsaved changes, leave anyway?" so nobody loses a half-filled form.

**Cross-question:** Is a guard security?

**Cross-answer:**

No. It only blocks navigation in the browser. The user controls the browser, so they can change the code or call the API directly with a tool. The API must still do its own checks. A guard is like a "Staff only" sign on a door; the real lock has to be on the server.

---

## Q6. Lazy load and preload?

**Answer:**

Lazy loading loads a part of the app only when you navigate to it. Preloading downloads extra chunks in the background after the first page is up. That makes later pages faster, but uses more bandwidth now.

So the two work together. Lazy loading keeps the first load small, so the app opens quickly. Preloading then uses the quiet time after that to fetch the rest, so when the user clicks to another page, the code is already there. It's like a waiter bringing your starter first, while the kitchen gets the main course ready.

**Cross-question:** Preload everything?

**Cross-answer:**

That is fine for a desktop intranet app, where the network is fast and data is free. On mobile, I preload only the areas people use most, because users may be on slow or paid data and most of them won't open every page.

---

## Q7. `patchValue` vs `setValue`? Disabled controls?

**Answer:**

`setValue` needs every key. `patchValue` lets you update just some of them. I disable a control in TypeScript with `control.disable()`, not with `[disabled]` on a reactive control. Disabled fields are left out of `.value`, so I use `getRawValue()` if I still need them.

So if the form has name, email and phone, `setValue` with only name and email throws an error, while `patchValue` updates those two and leaves phone alone. `setValue` is strict on purpose: it catches the case where I forgot a field.

For disabling, a reactive form keeps its state in TypeScript, so I change it there too. Putting `[disabled]` in the template fights with the form's own state, and Angular warns about it. The `getRawValue()` part catches people out: a disabled "customer id" field won't be in `.value`, so if I send `.value` to the API, that field quietly goes missing.

**Cross-question:** FormArray?

**Cross-answer:**

It is a list of controls, like extra phone numbers. I push and remove controls as the user adds or deletes rows. It's for when I don't know up front how many fields there will be.

---

## Q8. Async validator?

**Answer:**

It is a function that returns an Observable of errors, or null if valid. A typical case is checking that an email is unique. I debounce it so we do not hit the API on every key press.

A normal validator can answer right away, like "is this field empty?". Some checks need the server, like "is this email already registered?", and the answer takes time to come back. That's what an async validator is for. While it waits, the form's status is `pending`. Debouncing means waiting until the user pauses typing before asking, so typing a 20-letter email doesn't send 20 requests.

**Cross-question:** Form stuck `pending`?

**Cross-answer:**

That means the async validator never completed. The Observable has to complete or error. Angular waits for it to finish before it decides the field is valid, so a stream that keeps running, like one built on a Subject that never ends, leaves the form waiting forever.

---

## Q9. OnPush and mutation?

**Answer:**

If I do `user.name = 'A'` and `user` is still the same object, OnPush may not update the view. So I replace the object with a new one, or I use a signal.

With OnPush, Angular checks whether an input is a different object, not whether something inside it changed. Changing `user.name` is like changing a page inside a folder: the folder looks the same from the outside, so Angular doesn't open it. Making a new object, like `{ ...user, name: 'A' }`, hands Angular a new folder, so it knows to look.

**Cross-question:** Signals plus OnPush?

**Cross-answer:**

That is a good default for lists. Signals tell Angular exactly what changed, so it doesn't have to guess, and OnPush makes sure it skips everything else.

---

## Q10. Interceptor refresh race?

**Answer:**

A 401 means "your access token is no longer valid". The usual fix is to call a refresh endpoint to get a new token and then retry the request.

If two requests get a 401 at the same time, they should not start two refresh calls. They should share one refresh. If the refresh fails, I log the user out.

Why it matters: a dashboard can fire five calls at once, and if the token just expired, all five fail together. If each starts its own refresh, the server may rotate the refresh token on the first call and reject the others, and the user gets logged out for no good reason. So the first 401 starts the refresh, and the others wait for that same result and then retry with the new token.

**Cross-question:** Add the token to every HTTP call on the internet?

**Cross-answer:**

No. Only to calls going to my own API origin. If I send it to a third-party site, like a public CDN or another company's API, I'm handing my user's login token to someone else.

---

## Q11. NgRx or a service?

**Answer:**

NgRx is a library for keeping app state in one central store, where every change goes through named actions. It's based on the Redux pattern.

For most apps, a service plus signals is enough. NgRx helps when many screens share complex events. I would not add NgRx for six CRUD pages.

NgRx gives you a clear history of what changed and why, which helps in a big app where lots of screens react to the same events. But it comes with a lot of extra files and setup. For simple pages that just load, edit and save data, a service holding a signal does the same job with far less code.

**Cross-question:** Overkill sign?

**Cross-answer:**

When there is more boilerplate than features. When there are actions for every keystroke. When the team spends more time fighting the store than building the product.

---

## Q12. Content projection?

**Answer:**

`<ng-content>` takes HTML from the parent and puts it inside the child. For named slots, I use `select="[actions]"`. For optional layouts, I use templates (`ng-template`).

It lets me build a "shell" component that doesn't know its contents in advance. A card component can own the border, padding and shadow, and whoever uses it decides what goes inside: text, an image, buttons. Named slots let the parent put different pieces in different places, like content in the body and buttons in the footer. It's like a picture frame: the frame is the same, the picture changes.

**Cross-question:** Projection vs `@Input()`?

**Cross-answer:**

An input is for data. Projection is for markup. A card title as an input is fine. A whole toolbar works nicer as projection, because passing a bunch of buttons and their click handlers through inputs gets messy fast.

---

## Q13. Dynamic component vs `@if`?

**Answer:**

A dynamic component is one I create from code at runtime, instead of writing it in the template.

For a few variants, I use `@if` or `NgComponentOutlet`. For a widget host, I use `createComponent`. I prefer templates when I can.

If a page can show one of three known layouts, `@if` is the simplest: anyone reading the template sees all the options. `NgComponentOutlet` lets me pass in which component to show. `createComponent` is for cases like a dashboard where widgets are picked from a config at runtime and I don't know them in advance. Templates are easier to read and debug, so I only go fully dynamic when I have to.

**Cross-question:** How do you set inputs on a dynamic component?

**Cross-answer:**

With `setInput('title', value)`, so Angular goes through its normal input system. Don’t just set the field directly. If I write to the field directly, things that depend on inputs, like `ngOnChanges` or OnPush checks, may not notice the change.

---

## Q14. Performance checklist?

**Answer:**

Track list items by id. Use OnPush or signals. Lazy-load routes. Use virtual scroll for huge lists. Keep heavy work out of the template. Use `NgOptimizedImage`.

In plain words:

- Tracking by id lets Angular keep rows it already drew instead of rebuilding them.
- OnPush or signals mean Angular only checks the parts of the screen whose data changed.
- Lazy-loading routes keeps the first download small.
- Virtual scroll draws only the rows you can see.
- Heavy work in the template (like sorting inside a method call) can run on every check, so I do it once in code instead.
- `NgOptimizedImage` helps images load in a smarter order and size, so the page appears faster.

**Cross-question:** Why virtual scroll?

**Cross-answer:**

Rendering 10,000 real rows is slow. With virtual scroll we only render what you can see, plus a small buffer. As you scroll, the same few row elements are reused with new data. It's like a train window: the view changes, but the window is the same size.

---

## Q15. How do you test a component?

**Answer:**

I use `TestBed`, click things, and expect some text or an output. I fake HTTP with `HttpTestingController`. I test behavior, not private methods.

`TestBed` is Angular's way to create a component in a test, like a small fake app. Then I act like a user: click a button, type in a field, and check what shows on screen or which event fired. `HttpTestingController` lets me catch the HTTP call and answer it with fake data, so the test doesn't need a real server.

Testing behavior means that if I rename or rewrite a private method later, the test still passes as long as the user sees the same result. Tests tied to private methods break on every refactor and don't prove the screen works.

**Cross-question:** Shallow vs deep?

**Cross-answer:**

Shallow tests mock the child components, so they are faster. But for a tricky widget, I still want a few tests with the real children, because some bugs only show up when the parent and the real child work together.

---

## Q16. `switchMap` vs `exhaustMap` on save?

**Answer:**

For a save button, `exhaustMap`, so a double click does not fire two saves. For search, `switchMap`.

`exhaustMap` ignores new clicks while a save is still running, so the user can't create the same order twice by clicking quickly. `switchMap` would be wrong here, because it cancels the first save on the browser side, but the server may already have received it. For search it's the other way around: I only care about the latest text, so cancelling the old search is exactly what I want.

**Cross-question:** `mergeMap` on save?

**Cross-answer:**

You get two overlapping saves racing each other. Both requests run at the same time, and whichever finishes last decides the final state. That is usually wrong.

---

## Q17. Route params — snapshot vs paramMap Observable?

**Answer:**

The snapshot is fine if the component is created fresh each time. But if the same component is reused when going from `/user/1` to `/user/2`, I have to listen to `paramMap`.

The snapshot is a one-time photo of the URL at the moment the component was created. When the user goes from user 1 to user 2, Angular can keep the same component on screen and just change the URL. The snapshot still says "1", so the page keeps showing the old user. `paramMap` is an Observable, so it sends the new id each time and I can load the right user.

**Cross-question:** Resolver?

**Cross-answer:**

A resolver loads data before the component is shown. It is less common now, but fine if you really need the data before the first paint. The downside is that the user sees nothing happen until the data arrives, so many apps prefer to show the page right away with a loading state.

---

## Q18. `ChangeDetectorRef.markForCheck` vs `detectChanges`?

**Answer:**

`markForCheck` flags the component to be checked in the next cycle (for OnPush). `detectChanges` runs the check right now. I use markForCheck more often. Using detectChanges in tests is common.

`markForCheck` is like putting a note on the door: "check this room on your next round". Angular picks it up at its normal time, together with everything else. `detectChanges` is like calling someone to check the room right now, outside the normal routine. In tests that's handy, because I want the screen updated before I check it.

**Cross-question:** Call detectChanges in production code a lot?

**Cross-answer:**

That is a code smell, meaning a sign that something underneath is wrong. The data should flow in a way that Angular can see on its own, through inputs, the async pipe or signals. Forcing checks by hand usually hides that problem instead of fixing it.

---

## Q19. i18n?

**Answer:**

i18n is short for "internationalization" (18 letters between the "i" and the "n"). It means making the app work in more than one language and region.

With Angular’s built-in i18n, you build the app once per locale. Libraries like ngx-translate let you switch language at runtime. For dates, I use locale-aware pipes, not hand-built strings.

So with the built-in way, the English and Hindi versions are separate builds, and switching language means loading a different version of the app. With a runtime library, the user can switch from a dropdown without a reload. For dates and numbers, countries write them differently (for example, day-month order or the decimal separator), and locale-aware pipes handle that for me.

**Cross-question:** Hard-coded English in templates?

**Cross-answer:**

That is fine for an internal, English-only app. But a product for many countries needs a plan early, because adding it later is painful. Every piece of text across every template has to be found and pulled out, and layouts often break when a translated word is much longer.

---

## Q20. How do you share a header user name across pages?

**Answer:**

I keep it in an auth service, as a signal or a BehaviorSubject. The header reads it, login sets it, and logout clears it. I do not copy the name into ten components.

Since the service is a single shared instance, there's one source of truth. When the user logs in or updates their name, everything reading it updates on its own. If I copied the name into each component, one of them would sooner or later show an old value.

**Cross-question:** Refresh the page — user gone?

**Cross-answer:**

A page refresh restarts the whole Angular app, so anything kept only in memory is lost. I restore it from a session cookie/token on startup (`APP_INITIALIZER`, or an auth guard that waits). That way, the app asks "who is logged in?" before showing pages that need the user.

---

## Q21. `providedIn: 'root'` vs route providers?

**Answer:**

`root` gives one instance for the whole app. Route providers live only as long as that route. That is good for a feature store you want to throw away when the user leaves.

For example, a service for app-wide things like the logged-in user goes in root, because the whole app needs it all the time. But a store holding the filters and results for a reports section only matters while the user is in that section. Providing it on the route keeps that data from hanging around in memory when it's no longer needed.

**Cross-question:** Two “singletons”?

**Cross-answer:**

`providedIn: 'any'` in lazy modules used to do that. Each lazy-loaded module got its own instance, so a service you expected to be shared could quietly exist twice with different data. I stick to root unless I have a real reason.

---

## Q22. What is a pure pipe vs impure?

**Answer:**

A pure pipe runs only when the input reference changes. An impure pipe runs very often (the `async` pipe is one). I write pure pipes unless I really can’t.

"Reference changes" means a different object or value, not the same object edited inside. Pure pipes are cheap, because Angular can skip them when nothing new came in. An impure pipe runs on every check of the screen, which is fine for something light like `async`, but a heavy impure pipe, like one that filters a big list, slows down the whole page.

**Cross-question:** Pipe that sorts an array in place?

**Cross-answer:**

That is bad, because it mutates the input. Sorting in place changes the original array that other parts of the app also use, so data changes order in places that never asked for it. Return a new array instead.

---

## Q23. How do you cancel an HTTP call on leave?

**Answer:**

With `takeUntilDestroyed`, the async pipe, or `switchMap` from the route. If I `subscribe` by hand and forget to clean up, the call can finish later and set state on a component that is already gone.

All three stop listening when they're no longer needed. `takeUntilDestroyed` and the async pipe unsubscribe when the component is destroyed, and for an HTTP call, unsubscribing cancels the request. `switchMap` on the route params cancels the old call when the user moves to a new id. Without this, a slow response can arrive after the user left and run code nobody expects, like showing a stale message or keeping memory busy.

**Cross-question:** Memory leak sign?

**Cross-answer:**

A `subscribe` in `ngOnInit` with no teardown. Each time the user opens that page, a new subscription is added and the old ones never stop. I look for that in code reviews.

---

## Q24. Reactive forms: `valueChanges`?

**Answer:**

It is an Observable of the form’s value. It sends the new value every time the user changes something. I debounce it, then run the search. I unsubscribe with `takeUntilDestroyed`.

A common use is a filter form above a table: as the user types or picks options, the table updates on its own, with no "Search" button. Debouncing waits until the user pauses, so we don't call the API for every key.

**Cross-question:** `updateOn: 'blur'`?

**Cross-answer:**

It validates when the user leaves the field, not on every key press. That is a better experience for something like email. Otherwise, the user sees "invalid email" the moment they type the first letter, which feels like being told off before they've finished.

---

## Q25. What is `NgZone.run` for?

**Answer:**

NgZone is the part of Angular (built on Zone.js) that notices when async work like clicks, timers or HTTP calls finishes, so it knows when to update the screen.

When third-party code runs outside Angular, the UI does not update. `run` brings that work back into Angular’s zone so change detection happens. With zoneless Angular and signals, I notify through signals instead.

For example, a chart or map library may fire its own events in a way Angular doesn't notice. The data changes, but the screen stays the same until something else triggers a check. Wrapping the update in `NgZone.run` tells Angular "something happened, check the screen". With signals, setting the signal is already the notice, so `run` isn't needed.

**Cross-question:** `runOutsideAngular`?

**Cross-answer:**

I use it for noisy events like mousemove, where I do not want 60 change detections a second. Each event would make Angular check the screen, even when nothing visible changed. I only step back into Angular when I actually need to update the UI.


---
## Additional Questions
### Q26. `switchMap`, `concatMap`, `mergeMap`, `exhaustMap` in one example?
**Answer:** Search uses switchMap, saves that must stay in order can use concatMap, independent parallel work can use mergeMap, and a login or submit button often uses exhaustMap to ignore duplicate clicks.

A queue at a shop is a handy way to remember them. `switchMap` is a cashier who drops the current customer when a new one walks in, which is fine when only the newest request matters. `concatMap` serves customers one by one in order. `mergeMap` opens several counters and serves everyone at once. `exhaustMap` won't let anyone else in until the current customer is done, and the ones who came in the meantime are turned away, not queued.
### Q27. How do you handle multiple 401 responses?
**Answer:** I share one refresh request. The other failed requests wait for it, then retry with the new token. If the refresh fails, I clear the auth state and send the user to login.

A 401 means the access token is no longer accepted. If every failed request started its own refresh, the server could get several refresh calls at once, and some of them may fail or cancel each other out. With one shared refresh, the user doesn't notice anything: the calls pause for a moment and then go through. If even the refresh fails, the session is really over, so sending them to login is the honest next step.
### Q28. How do you build a dynamic FormArray?
**Answer:** I create a FormArray, push a FormGroup for each row, and remove that group by index when the user deletes a row. The API still validates every row.

Think of an invoice form where the user adds line items. Each line has the same fields (product, quantity, price), so each line is one FormGroup, and the FormArray is the list of lines. Clicking "Add item" pushes a new group, and the delete button on a row removes it at that position. Each row gets its own validation in the browser, but the server checks every row again, because browser checks can be skipped.
