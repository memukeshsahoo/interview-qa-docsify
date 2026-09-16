# Angular — Advanced

Signals, zoneless, SSR, DI tree. Keep it human.

---

## Q1. Signals, `computed`, `effect`?

**Answer:**

A signal holds a value and knows who read it. `computed` derives another value and caches it. `effect` runs a side effect when inputs change — log, write localStorage. If I need a value, I use computed, not an effect that copies state.

**Cross-question:** Effect that `set`s another signal?

**Cross-answer:**

Easy to loop. Smell. Prefer computed or `linkedSignal`.

---

## Q2. `resource` / HTTP as signals?

**Answer:**

Newer API: params are signals, status is loading/error/value, you can reload. `rxResource` wraps Observables.

**Cross-question:** Does this replace NgRx?

**Cross-answer:**

For one screen’s server data, often yes. For a big event-driven app, maybe not. I pick the smallest tool.

---

## Q3. Zoneless Angular?

**Answer:**

Without Zone.js, Angular does not magically see `setTimeout`. Updates come from signals, events, and async pipe. Smaller and more predictable. Third-party libs that poke the DOM must tell Angular.

**Cross-question:** Does OnPush still matter?

**Cross-answer:**

Yes, it still limits who gets checked. Signals already mark consumers.

---

## Q4. SSR and hydration?

**Answer:**

SSR: first HTML comes from the server. Hydration: the client reuses that HTML instead of throwing it away. Faster first paint, better SEO.

**Cross-question:** Hydration mismatch?

**Cross-answer:**

Server HTML and client first draw differ — `Date.now()`, `Math.random()`, `window` in the constructor. First render must be the same. Browser-only code goes in `afterNextRender`.

---

## Q5. Why `@for` requires `track`?

**Answer:**

So the team cannot forget. Reusing rows is not optional on real lists.

**Cross-question:** Track by index?

**Cross-answer:**

Better than nothing, still bad if the list sorts or inserts. Track by id.

---

## Q6. Injector tree?

**Answer:**

DI walks up. Component providers are local. `providedIn: 'root'` is the root environment injector. Route injectors last as long as the route.

**Cross-question:** Circular DI?

**Cross-answer:**

A needs B needs A. I extract a third type. Cycles are a design smell.

---

## Q7. App initializer?

**Answer:**

Run async work before the app starts: load config, load current user. If it fails, I handle it or the app may not boot.

**Cross-question:** Runtime config.json vs environment.ts?

**Cross-answer:**

Runtime config: one build, many environments. environment.ts needs a rebuild per env.

---

## Q8. XSS and `bypassSecurityTrustHtml`?

**Answer:**

Angular escapes `{{ }}`. `[innerHTML]` is sanitized. Bypass turns safety off. Only for trusted static HTML. Never for user text.

**Cross-question:** Is interpolation XSS-safe?

**Cross-answer:**

It is escaped. Danger is innerHTML, `javascript:` URLs, and iframes. The server must still encode.

---

## Q9. CSRF vs JWT header?

**Answer:**

Cookie login can need CSRF tokens. A JWT in a header is not sent by random other sites the same way. XSS stealing the token is the bigger worry for SPA storage.

**Cross-question:** `withCredentials: true`?

**Cross-answer:**

Sends cookies cross-origin. CORS cannot be `*`. Easy to misconfigure.

---

## Q10. Micro-frontends?

**Answer:**

Split the UI so teams deploy alone. Cost: two Angular copies, shared library pain, uneven UX. One app is simpler.

**Cross-question:** When would you split?

**Cross-answer:**

Big org, different release trains, a legacy island. Not a five-person product.

---

## Q11. Monorepo libraries?

**Answer:**

Shared UI and util libs. Rules: app → feature → ui → util. Apps should not import each other’s guts.

**Cross-question:** Publish to npm?

**Cross-answer:**

Not until someone outside the repo needs it. Path imports are enough inside a monorepo.

---

## Q12. Ivy / AOT?

**Answer:**

Templates compile at build time. Faster runtime, template errors at build. We ship AOT. Old JIT-in-the-browser is not how we deploy.

**Cross-question:** Local compilation?

**Cross-answer:**

Ivy can compile a component using only what it sees. Faster builds.

---

## Q13. `ExpressionChangedAfterItHasBeenCheckedError`?

**Answer:**

I changed a bound value after Angular already checked it in the same turn. Parent and child fighting is a common cause. Fix the data flow. Do not “fix” it by turning off the error.

**Cross-question:** Why only in dev?

**Cross-answer:**

Dev checks twice to catch this. Prod skips the extra check. The flicker can still be there.

---

## Q14. Design an incident dashboard feature.

**Answer:**

Lazy route. `IncidentApi` service. List with `@for` and track id. Filters with debounce + switchMap. Guard the route. Loading/empty/error. DTO types. No `any`. Auth interceptor. Numbers come from the API so tenants cannot see each other.

**Cross-question:** Where do you map snake_case to camelCase?

**Cross-answer:**

One place: interceptor or API layer. Components see one shape.

---

## Q15. Tree-shaking services?

**Answer:**

`providedIn: 'root'` can drop a service if nobody injects it. A module `providers` array can keep extra code. Standalone plus `providedIn` is friendlier for the bundler.

**Cross-question:** Side-effect imports?

**Cross-answer:**

`import 'foo'` with no symbols can be kept or dropped depending on config. I avoid random side-effect imports.

---

## Q16. `linkedSignal` in simple words?

**Answer:**

A writable signal that also resets when something else changes. Example: selected id becomes invalid when the list reloads.

**Cross-question:** vs computed?

**Cross-answer:**

Computed is read-only. Linked is for “derived but the user can also change it”.

---

## Q17. Control flow vs structural directives — migration pain?

**Answer:**

`@if` narrows types better. `*ngIf` still works. `as` and else templates map to `@else`. I migrate screen by screen, not a giant bang.

**Cross-question:** Mix both?

**Cross-answer:**

Yes during migration. I do not mix in the same tiny template for fun.

---

## Q18. How do you keep bundle size down?

**Answer:**

Lazy routes, do not import all of lodash, check source-map-explorer, watch PrimeNG/Material imports, avoid huge moment.js (use date-fns or native Intl).

**Cross-question:** First paint still slow?

**Cross-answer:**

Look at SSR, fewer fonts, `NgOptimizedImage`, less JS on home.

---

## Q19. Secondary router outlets / matrix params — do you use them?

**Answer:**

Rarely. Most products need a simple URL. Fancy outlets confuse people and analytics. I use query params for filters.

**Cross-question:** Filters in query params vs in a service only?

**Cross-answer:**

Query params: you can share the URL and refresh. Service only: refresh loses filters. I prefer the URL for lists.

---

## Q20. How would you handle a 401 in a SPA?

**Answer:**

Interceptor catches 401, try refresh once, retry the call, or redirect to login. I clear in-memory user state. I do not loop refresh forever.

**Cross-question:** 403?

**Cross-answer:**

User is logged in but not allowed. I show “no access”, not the login page.

---

## Q21. Signals in a service as the store?

**Answer:**

Private `_items = signal<Item[]>([])`. Public `items = this._items.asReadonly()`. Methods update with `.update`. Components only read. Simple and testable.

**Cross-question:** Mutate the array in place?

**Cross-answer:**

No. Replace the array so consumers notice. Same rule as OnPush.

---

## Q22. What is hydration skipping / incremental hydration?

**Answer:**

Do not hydrate a heavy widget until the user needs it. Saves main-thread work on first load.

**Cross-question:** Risk?

**Cross-answer:**

Until it hydrates, it may not be interactive. I skip below-the-fold stuff, not the main button.

---

## Q23. Testing signals?

**Answer:**

Set inputs, call methods, assert `component.count()`. For effects, I flush the testing scheduler if needed. I still test through the template for important UX.

**Cross-question:** Fake time?

**Cross-answer:**

For debounce, yes. Otherwise tests become slow or flaky.

---

## Q24. How do you think about accessibility in Angular?

**Answer:**

Labels on inputs, focus after errors, keyboard on custom widgets, `alt` on images, do not only use color. CDK a11y helpers exist. I do not ship a click-only div as a button.

**Cross-question:** `div (click)` as a button?

**Cross-answer:**

Use `<button>`. Screen readers and Enter/Space work. CSS can make it look like a tile.

---

## Q25. Standalone + provideHttpClient + interceptors?

**Answer:**

`bootstrapApplication` with `provideHttpClient(withInterceptors([authInterceptor]))`. Functional interceptors are just functions. Easier than old DI classes.

**Cross-question:** Order of interceptors?

**Cross-answer:**

First listed is the outer one. Auth then logging or the other way — I pick and stay consistent. Errors should still be handled once.
