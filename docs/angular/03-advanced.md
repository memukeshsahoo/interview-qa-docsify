# Angular — Advanced

Signals, zoneless, SSR, DI tree. Keep it human.

---

## Q1. Signals, `computed`, `effect`?

**Answer:**

A signal holds a value, and it also knows who read that value. `computed` builds another value from signals and caches it. `effect` runs a side effect when its inputs change, like logging or writing to localStorage. If what I need is a value, I use computed. I don't write an effect that just copies state around.

**Cross-question:** Effect that `set`s another signal?

**Cross-answer:**

That makes it easy to end up in a loop, so I treat it as a smell. I'd rather use computed or `linkedSignal`.

---

## Q2. `resource` / HTTP as signals?

**Answer:**

It's a newer API. The params are signals, the status tells you if it's loading, has an error, or has a value, and you can reload it. `rxResource` does the same thing but wraps Observables.

**Cross-question:** Does this replace NgRx?

**Cross-answer:**

For the server data of one screen, often yes. For a big event-driven app, maybe not. I pick the smallest tool that does the job.

---

## Q3. Zoneless Angular?

**Answer:**

Without Zone.js, Angular doesn't magically notice a `setTimeout` anymore. Updates come from signals, events, and the async pipe. The app gets smaller and more predictable. The catch is that third-party libraries that poke the DOM have to tell Angular when something changed.

**Cross-question:** Does OnPush still matter?

**Cross-answer:**

Yes. It still limits which components get checked. Signals already mark the components that read them.

---

## Q4. SSR and hydration?

**Answer:**

With SSR, the first HTML comes from the server. Hydration means the browser reuses that HTML instead of throwing it away and drawing it again. You get a faster first paint and better SEO.

**Cross-question:** Hydration mismatch?

**Cross-answer:**

That's when the server HTML and the client's first draw are different. Common causes are `Date.now()`, `Math.random()`, or using `window` in the constructor. The first render has to be the same on both sides. Browser-only code goes in `afterNextRender`.

---

## Q5. Why `@for` requires `track`?

**Answer:**

So the team can't forget it. On real lists, reusing rows isn't optional.

**Cross-question:** Track by index?

**Cross-answer:**

It's better than nothing, but it still breaks down if the list gets sorted or items get inserted. I track by id.

---

## Q6. Injector tree?

**Answer:**

DI looks up the tree until it finds a provider. Providers on a component are local to that component. `providedIn: 'root'` puts the service in the root environment injector. Route injectors live as long as the route does.

**Cross-question:** Circular DI?

**Cross-answer:**

That's when A needs B and B needs A. I pull the shared part out into a third type. A cycle usually means the design needs another look.

---

## Q7. App initializer?

**Answer:**

It runs async work before the app starts, like loading config or the current user. If it fails, I have to handle that, or the app may not boot at all.

**Cross-question:** Runtime config.json vs environment.ts?

**Cross-answer:**

With a runtime config, I build once and deploy the same build to many environments. environment.ts needs a separate build for each environment.

---

## Q8. XSS and `bypassSecurityTrustHtml`?

**Answer:**

Angular escapes `{{ }}`, and `[innerHTML]` gets sanitized. The bypass turns that safety off. I only use it for trusted, static HTML, never for text that came from a user.

**Cross-question:** Is interpolation XSS-safe?

**Cross-answer:**

Yes, it's escaped. The danger is in innerHTML, `javascript:` URLs, and iframes. The server still has to encode its output too.

---

## Q9. CSRF vs JWT header?

**Answer:**

If login uses cookies, you may need CSRF tokens. A JWT in a header isn't sent by other random sites the same way a cookie is. For an SPA that stores the token, the bigger worry is XSS stealing it.

**Cross-question:** `withCredentials: true`?

**Cross-answer:**

It sends cookies on cross-origin requests. Then CORS can't be `*`. It's easy to get this wrong.

---

## Q10. Micro-frontends?

**Answer:**

You split the UI so each team can deploy on its own. The cost is real: two copies of Angular, pain with shared libraries, and a UX that doesn't feel consistent. One app is simpler.

**Cross-question:** When would you split?

**Cross-answer:**

In a big org with different release trains, or when there's a legacy island. Not for a five-person product.

---

## Q11. Monorepo libraries?

**Answer:**

I keep shared UI and util libraries. The rule is app → feature → ui → util. Apps shouldn't import each other's internals.

**Cross-question:** Publish to npm?

**Cross-answer:**

Not until someone outside the repo needs it. Inside a monorepo, path imports are enough.

---

## Q12. Ivy / AOT?

**Answer:**

With AOT, templates compile at build time. The app runs faster, and template errors show up at build time instead of in the browser. We ship AOT. The old JIT-in-the-browser approach isn't how we deploy.

**Cross-question:** Local compilation?

**Cross-answer:**

Ivy can compile a component using only what it sees in that component. That makes builds faster.

---

## Q13. `ExpressionChangedAfterItHasBeenCheckedError`?

**Answer:**

It means I changed a bound value after Angular had already checked it in the same turn. A common cause is a parent and child fighting over a value. The fix is to fix the data flow. I don't "fix" it by hiding the error.

**Cross-question:** Why only in dev?

**Cross-answer:**

In dev, Angular checks twice to catch this. In prod it skips the extra check. The flicker can still be there, though.

---

## Q14. Design an incident dashboard feature.

**Answer:**

I'd make it a lazy route with an `IncidentApi` service. The list uses `@for` with track by id. Filters use debounce plus switchMap. I guard the route and handle loading, empty, and error states. I use DTO types and no `any`. An auth interceptor adds the token. The numbers come from the API, so tenants can't see each other's data.

**Cross-question:** Where do you map snake_case to camelCase?

**Cross-answer:**

In one place: the interceptor or the API layer. That way components only ever see one shape.

---

## Q15. Tree-shaking services?

**Answer:**

With `providedIn: 'root'`, the bundler can drop a service if nobody injects it. A module `providers` array can keep extra code in the bundle. Standalone plus `providedIn` is friendlier for the bundler.

**Cross-question:** Side-effect imports?

**Cross-answer:**

An `import 'foo'` with no symbols might be kept or dropped, depending on the config. I avoid random side-effect imports.

---

## Q16. `linkedSignal` in simple words?

**Answer:**

It's a writable signal that also resets when something else changes. For example, the selected id becomes invalid when the list reloads.

**Cross-question:** vs computed?

**Cross-answer:**

Computed is read-only. A linked signal is for "derived, but the user can also change it".

---

## Q17. Control flow vs structural directives — migration pain?

**Answer:**

`@if` narrows types better. `*ngIf` still works. `as` and else templates map over to `@else`. I migrate screen by screen, not in one big bang.

**Cross-question:** Mix both?

**Cross-answer:**

Yes, during the migration. I just don't mix them in the same small template for no reason.

---

## Q18. How do you keep bundle size down?

**Answer:**

I use lazy routes, I don't import all of lodash, and I check the bundle with source-map-explorer. I watch PrimeNG/Material imports, and I avoid the huge moment.js (I use date-fns or native Intl instead).

**Cross-question:** First paint still slow?

**Cross-answer:**

Then I look at SSR, fewer fonts, `NgOptimizedImage`, and less JS on the home page.

---

## Q19. Secondary router outlets / matrix params — do you use them?

**Answer:**

Rarely. Most products just need a simple URL. Fancy outlets confuse people and analytics. For filters, I use query params.

**Cross-question:** Filters in query params vs in a service only?

**Cross-answer:**

With query params, you can share the URL and refresh the page. With a service only, a refresh loses the filters. For lists, I prefer the URL.

---

## Q20. How would you handle a 401 in a SPA?

**Answer:**

An interceptor catches the 401 and tries a refresh once, then retries the call or redirects to login. I clear the in-memory user state. I don't let it loop on refresh forever.

**Cross-question:** 403?

**Cross-answer:**

That means the user is logged in but not allowed. I show a "no access" message, not the login page.

---

## Q21. Signals in a service as the store?

**Answer:**

I keep a private `_items = signal<Item[]>([])` and expose a public `items = this._items.asReadonly()`. Methods change it with `.update`. Components only read. It's simple and easy to test.

**Cross-question:** Mutate the array in place?

**Cross-answer:**

No. I replace the array so consumers notice the change. It's the same rule as with OnPush.

---

## Q22. What is hydration skipping / incremental hydration?

**Answer:**

You don't hydrate a heavy widget until the user needs it. That saves main-thread work on the first load.

**Cross-question:** Risk?

**Cross-answer:**

Until it hydrates, it may not respond to the user. So I skip things below the fold, not the main button.

---

## Q23. Testing signals?

**Answer:**

I set inputs, call methods, and assert on `component.count()`. For effects, I flush the testing scheduler if needed. For important UX, I still test through the template.

**Cross-question:** Fake time?

**Cross-answer:**

Yes, for debounce. Otherwise the tests get slow or flaky.

---

## Q24. How do you think about accessibility in Angular?

**Answer:**

Labels on inputs, moving focus after errors, keyboard support on custom widgets, `alt` on images, and not relying on color alone. The CDK has a11y helpers for this. I don't ship a click-only div as a button.

**Cross-question:** `div (click)` as a button?

**Cross-answer:**

Use a `<button>`. Then screen readers and Enter/Space just work. CSS can still make it look like a tile.

---

## Q25. Standalone + provideHttpClient + interceptors?

**Answer:**

I call `bootstrapApplication` with `provideHttpClient(withInterceptors([authInterceptor]))`. Functional interceptors are just functions, which is easier than the old DI classes.

**Cross-question:** Order of interceptors?

**Cross-answer:**

The first one listed is the outer one. Auth then logging, or the other way round — I pick one order and stay consistent. Errors should still be handled in only one place.


---
## Additional Questions
### Q26. When would you use `computed` instead of `effect`?
**Answer:** If I need a value derived from other signals, I use computed. I use effect for side effects, like logging or syncing with browser APIs.
### Q27. What is hydration good for?
**Answer:** The server sends useful HTML first, and the browser attaches Angular behavior to it. For the right kind of app, that can improve first paint and SEO.
### Q28. What is a signal equality function for?
**Answer:** It decides when Angular treats a new signal value as different. I use it carefully, because hiding real changes behind custom equality can leave the UI stale.
