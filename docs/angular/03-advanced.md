# Angular — Advanced

Signals, zoneless, SSR, DI tree. Keep it human.

---

## Q1. Signals, `computed`, `effect`?

**Answer:**

A signal holds a value, and it also knows who read that value. `computed` builds another value from signals and caches it. `effect` runs a side effect when its inputs change, like logging or writing to localStorage. If what I need is a value, I use computed. I don't write an effect that just copies state around.

Think of a signal as a box with a value in it, plus a list of everyone who looked inside. When the value changes, Angular knows exactly who needs to hear about it, so it only updates the parts of the screen that use it. A `computed` is like a spreadsheet cell with a formula: if I have `price` and `quantity` signals, `total = computed(() => price() * quantity())` updates by itself, and it only recalculates when one of them actually changes.

An `effect` is different. It doesn't give me a value back. It just does something when the signals it reads change, like saving the cart to localStorage. If I use an effect to copy one signal into another, I end up with two sources of truth and extra update rounds, which is harder to follow and easier to break. A computed gives the same result with less code.

**Cross-question:** Effect that `set`s another signal?

**Cross-answer:**

That makes it easy to end up in a loop, so I treat it as a smell. I'd rather use computed or `linkedSignal`.

The loop happens when the effect writes a signal that, directly or through other code, causes the effect to run again. Even when there's no loop, the data flow gets hard to trace, because a value changes "somewhere in an effect" instead of in one clear formula. With computed or `linkedSignal`, the relationship is written in one place and anyone reading the code can see it.

---

## Q2. `resource` / HTTP as signals?

**Answer:**

It's a newer API. The params are signals, the status tells you if it's loading, has an error, or has a value, and you can reload it. `rxResource` does the same thing but wraps Observables.

In plain words, a resource is a signal for data that has to be fetched. I tell it what to fetch based on some signals, for example the selected user id. When that id changes, it fetches the new data for me. The component can then read whether it's still loading, whether it failed, or the actual value, all as signals.

Why it helps: without it, I'd write my own `isLoading` and `error` flags and a subscription for every screen, and it's easy to forget one of them. With a resource, the loading spinner, the error message and the data all come from one place. `rxResource` is for when the fetch code already returns an Observable, like Angular's `HttpClient` does.

**Cross-question:** Does this replace NgRx?

**Cross-answer:**

For the server data of one screen, often yes. For a big event-driven app, maybe not. I pick the smallest tool that does the job.

NgRx is a full store with actions, reducers and effects. It shines when many parts of a large app react to the same events and you want a clear history of what happened. If one screen just needs to load a list and show it, that whole setup is extra code to write and maintain, and a resource does the job.

---

## Q3. Zoneless Angular?

**Answer:**

Without Zone.js, Angular doesn't magically notice a `setTimeout` anymore. Updates come from signals, events, and the async pipe. The app gets smaller and more predictable. The catch is that third-party libraries that poke the DOM have to tell Angular when something changed.

Zone.js is a library that wraps browser async APIs like timers, clicks and HTTP calls. Every time one of them finishes, it tells Angular "something might have changed, check the screen". That's convenient, but it's a guess: Angular often checks when nothing changed, and Zone.js itself adds code to the bundle.

In zoneless mode, Angular only updates when it gets a clear signal that something changed: a signal was set, a template event fired, or the async pipe got a new value. It's like a shop assistant who only comes over when you ring the bell, instead of walking past every few seconds to check. If a library changes data in its own callback that Angular doesn't know about, the screen won't update until something tells Angular, for example by setting a signal.

**Cross-question:** Does OnPush still matter?

**Cross-answer:**

Yes. It still limits which components get checked. Signals already mark the components that read them.

So OnPush and signals work together. When a check runs, OnPush lets Angular skip whole parts of the component tree that weren't marked as changed. Signals take care of the marking, so the components that read a changed signal get checked and the rest are left alone.

---

## Q4. SSR and hydration?

**Answer:**

With SSR, the first HTML comes from the server. Hydration means the browser reuses that HTML instead of throwing it away and drawing it again. You get a faster first paint and better SEO.

SSR means server-side rendering. Normally an Angular app sends an almost empty page and the browser builds everything with JavaScript, so the user stares at a blank screen for a moment. With SSR, the server runs the app first and sends real HTML, so the user sees content right away. Search engines also get proper content to read, which helps SEO.

Hydration is the second half. Once the JavaScript loads, Angular has to take over that page so buttons and forms work. Without hydration, it would wipe the server HTML and draw the same thing again, which can cause a flicker and wastes work. With hydration, Angular walks through the existing HTML and attaches itself to it, like moving into a furnished house instead of emptying it and buying the same furniture again.

**Cross-question:** Hydration mismatch?

**Cross-answer:**

That's when the server HTML and the client's first draw are different. Common causes are `Date.now()`, `Math.random()`, or using `window` in the constructor. The first render has to be the same on both sides. Browser-only code goes in `afterNextRender`.

For example, if the template shows `Date.now()`, the server prints one time and the browser calculates a slightly later time, so the two don't match. `window` doesn't exist on the server at all. `afterNextRender` only runs in the browser after Angular has rendered, so it's a safe place for code that needs browser things like `window` or measuring elements.

---

## Q5. Why `@for` requires `track`?

**Answer:**

So the team can't forget it. On real lists, reusing rows isn't optional.

`track` tells Angular how to recognise each item in a list, usually by its id. When the list changes, Angular uses that to work out which rows are the same item as before. It keeps and moves those rows instead of destroying them and building new ones.

Without good tracking, every refresh of the data can rebuild every row. That's slow on long lists, and it also loses things the user cares about, like text typed into an input inside a row, focus, or a row that was expanded. With the old `*ngFor`, `trackBy` was optional and people often forgot it. Making `track` required in `@for` means every list has to answer the question "how do I know which item is which?".

**Cross-question:** Track by index?

**Cross-answer:**

It's better than nothing, but it still breaks down if the list gets sorted or items get inserted. I track by id.

The index is just a position, not the item. If I insert a new item at the top, every item moves down one place, so Angular thinks row 0 is still row 0 and reuses it for different data. The id stays with the item wherever it moves, so Angular can match rows correctly.

---

## Q6. Injector tree?

**Answer:**

DI looks up the tree until it finds a provider. Providers on a component are local to that component. `providedIn: 'root'` puts the service in the root environment injector. Route injectors live as long as the route does.

DI (dependency injection) means a class asks for what it needs in its constructor or with `inject()`, and Angular hands it over instead of the class creating it. An injector is the thing that holds and creates those services. Angular has injectors at different levels, and they form a tree.

When a component asks for a service, Angular first checks that component's own injector, then its parent's, and keeps going up until it finds one that provides it. It's like asking for a stapler: first you check your desk, then your team's cupboard, then the office supply room. Where I provide a service decides how many copies exist and how long they live. In root, the whole app shares one copy. On a component, each instance of that component gets its own copy, which is handy for things like the state of one form.

**Cross-question:** Circular DI?

**Cross-answer:**

That's when A needs B and B needs A. I pull the shared part out into a third type. A cycle usually means the design needs another look.

Angular can't create A without B, and can't create B without A, so it throws an error. Usually both services need one small shared piece. If I move that piece into a third service C, then A and B both depend on C and neither depends on the other.

---

## Q7. App initializer?

**Answer:**

It runs async work before the app starts, like loading config or the current user. If it fails, I have to handle that, or the app may not boot at all.

An app initializer is a function Angular runs at startup, and Angular waits for it to finish before it shows the app. It's useful when everything else depends on some data, for example the API base URL from a config file. Without it, components could start making calls before they even know where the API is.

Because Angular waits on it, a failure here is serious. If the config request fails and I don't catch it, the user can be stuck on a blank screen. So I catch the error and either fall back to safe defaults or show a clear "couldn't start, please retry" message. I also keep the work small, because the user is waiting the whole time it runs.

**Cross-question:** Runtime config.json vs environment.ts?

**Cross-answer:**

With a runtime config, I build once and deploy the same build to many environments. environment.ts needs a separate build for each environment.

environment.ts is baked into the JavaScript when you build, so the dev, test and production values each need their own build. A `config.json` file is loaded when the app starts, so I can deploy the exact same build everywhere and only swap the JSON file. That means the build I tested is the build that goes to production.

---

## Q8. XSS and `bypassSecurityTrustHtml`?

**Answer:**

Angular escapes `{{ }}`, and `[innerHTML]` gets sanitized. The bypass turns that safety off. I only use it for trusted, static HTML, never for text that came from a user.

XSS (cross-site scripting) is when an attacker gets their own script to run inside your page, for example by typing `<script>` into a comment box. If that runs, it can read the user's data or act as them. Escaping means Angular shows `<script>` as plain text instead of running it. Sanitizing means Angular keeps safe HTML like bold text but strips the dangerous parts.

`bypassSecurityTrustHtml` tells Angular "don't check this, I promise it's safe". If I use it on something a user typed, I've removed the one guard that would have stopped the attack. So I keep it for HTML my own team wrote, like a fixed piece of help text.

**Cross-question:** Is interpolation XSS-safe?

**Cross-answer:**

Yes, it's escaped. The danger is in innerHTML, `javascript:` URLs, and iframes. The server still has to encode its output too.

`{{ }}` always treats the value as text, so a script tag just shows up as characters on the screen. The risky places are the ones that treat a value as HTML or as a link to load, like a link whose address starts with `javascript:`. And the front end isn't the only defence: other clients or emails may use the same data, so the server has to handle it safely as well.

---

## Q9. CSRF vs JWT header?

**Answer:**

If login uses cookies, you may need CSRF tokens. A JWT in a header isn't sent by other random sites the same way a cookie is. For an SPA that stores the token, the bigger worry is XSS stealing it.

CSRF (cross-site request forgery) works like this: the browser sends cookies automatically with every request to your site, even when the request was started by another site. So a bad page could make your browser send "transfer money" to your bank, and the cookie would make it look like you. A CSRF token is an extra secret value the bad site can't know, so the server can reject those requests.

A JWT is a signed token that says who the user is. If my app puts it in the `Authorization` header with code, the browser never adds it on its own, so another site can't trigger a request that carries it. The risk moves somewhere else: if the token sits in browser storage and an attacker gets a script into my page, that script can read it. That's why XSS protection matters so much for an SPA.

**Cross-question:** `withCredentials: true`?

**Cross-answer:**

It sends cookies on cross-origin requests. Then CORS can't be `*`. It's easy to get this wrong.

By default, the browser doesn't send cookies when the app calls an API on a different domain. `withCredentials: true` turns that on. The server then has to name the exact allowed origin and allow credentials. A wildcard `*` isn't accepted together with credentials, because that would let any site send requests with the user's cookies.

---

## Q10. Micro-frontends?

**Answer:**

You split the UI so each team can deploy on its own. The cost is real: two copies of Angular, pain with shared libraries, and a UX that doesn't feel consistent. One app is simpler.

A micro-frontend setup is like a shopping centre where each shop is run by a different owner. Each team builds and releases its own part of the UI, and a shell app puts the parts together on one page. The benefit is independence: the payments team can release without waiting for the search team.

The price is paid in other places. The user may download Angular more than once, which makes the app slower to load. Shared code like a design system has to stay in sync across teams. And small differences in buttons, spacing or behaviour make the product feel stitched together. For most teams, one well-organised app gives the same structure without these problems.

**Cross-question:** When would you split?

**Cross-answer:**

In a big org with different release trains, or when there's a legacy island. Not for a five-person product.

A release train is a team's own schedule for shipping. If many teams keep blocking each other's releases, splitting can be worth the cost. A legacy island is an old part of the app, maybe in an older framework, that I want to keep running while new parts are built next to it. For a small team, the coordination cost is bigger than the benefit.

---

## Q11. Monorepo libraries?

**Answer:**

I keep shared UI and util libraries. The rule is app → feature → ui → util. Apps shouldn't import each other's internals.

A monorepo is one Git repository that holds several apps and the libraries they share. Inside it, I split code into libraries by type. Feature libraries hold whole screens and their logic. UI libraries hold reusable visual pieces like buttons and tables. Util libraries hold plain helpers like date formatting.

The arrow shows which way imports are allowed: an app can use features, a feature can use UI, and UI can use utils, but never backwards. That keeps the base layers simple and reusable. If a button component imported a whole feature, every app using that button would pull in that feature too. It's like a building: upper floors rest on lower floors, never the other way round.

**Cross-question:** Publish to npm?

**Cross-answer:**

Not until someone outside the repo needs it. Inside a monorepo, path imports are enough.

Publishing means versioning, release steps and keeping old versions working for others. Inside the repo, every app already uses the latest code through a path alias, so a change and its users can be updated in the same commit. I only take on the publishing work when another repo actually needs the library.

---

## Q12. Ivy / AOT?

**Answer:**

With AOT, templates compile at build time. The app runs faster, and template errors show up at build time instead of in the browser. We ship AOT. The old JIT-in-the-browser approach isn't how we deploy.

Ivy is Angular's current compiler and rendering engine. AOT (ahead-of-time) means Angular turns the HTML templates into JavaScript instructions while building, before anything reaches the user. JIT (just-in-time) means the browser has to download the compiler and do that work on every load.

With AOT, the browser gets ready-to-run code, so the app starts faster and the bundle doesn't carry the compiler. It also catches mistakes early. If I misspell a property in a template, the build fails on my machine instead of the page breaking for a user. It's like a printed book versus handing someone your notes and asking them to typeset it before they can read.

**Cross-question:** Local compilation?

**Cross-answer:**

Ivy can compile a component using only what it sees in that component. That makes builds faster.

This is often called locality. Because each component's compiled output depends only on that component's own information, changing one component means only that one has to be recompiled, not everything that uses it.

---

## Q13. `ExpressionChangedAfterItHasBeenCheckedError`?

**Answer:**

It means I changed a bound value after Angular had already checked it in the same turn. A common cause is a parent and child fighting over a value. The fix is to fix the data flow. I don't "fix" it by hiding the error.

Angular updates the screen from top to bottom in one pass. Once it has read a value and put it on the screen, it expects that value to stay the same until the pass ends. If a child component then changes something the parent already displayed, for example by setting a parent's title in its own setup hook, the screen is now showing an old value. Angular throws this error to warn me.

The real fix is to make data flow in one direction: the parent owns the value and passes it down, or the value is set before the check runs. Wrapping the change in a `setTimeout` makes the error go away, but it just pushes the update to a later round. The bad flow is still there, and the user may see the value jump.

**Cross-question:** Why only in dev?

**Cross-answer:**

In dev, Angular checks twice to catch this. In prod it skips the extra check. The flicker can still be there, though.

The second check costs time, so Angular only does it in development to help me find the problem. In production the error isn't thrown, but the underlying issue hasn't gone away. The screen can still show an out-of-date value until the next update.

---

## Q14. Design an incident dashboard feature.

**Answer:**

I'd make it a lazy route with an `IncidentApi` service. The list uses `@for` with track by id. Filters use debounce plus switchMap. I guard the route and handle loading, empty, and error states. I use DTO types and no `any`. An auth interceptor adds the token. The numbers come from the API, so tenants can't see each other's data.

Here's why each piece is there:

- **Lazy route:** the dashboard code only downloads when someone opens it, so the rest of the app loads faster.
- **`IncidentApi` service:** all HTTP calls live in one place, so components don't need to know URLs.
- **Track by id:** rows are reused when the list refreshes instead of rebuilt.
- **Debounce plus switchMap:** debounce waits until the user stops typing for a moment before searching. switchMap cancels the old request when a new one starts, so an old slow answer can't overwrite a newer one.
- **Route guard:** only users who are allowed in can open the page.
- **Loading, empty and error states:** the user always knows what's happening, instead of seeing a blank table.
- **DTO types and no `any`:** a DTO (data transfer object) describes the exact shape the API returns, so TypeScript can catch mistakes like a misspelled field.
- **Auth interceptor:** it adds the login token to every request, so no one forgets it on one call.

**Cross-question:** Where do you map snake_case to camelCase?

**Cross-answer:**

In one place: the interceptor or the API layer. That way components only ever see one shape.

Many back ends send names like `created_at`, while TypeScript code usually uses `createdAt`. If every component did its own conversion, some would forget, and the same field would have two names in different parts of the app. Converting once, at the edge where data comes in, keeps the rest of the code clean.

---

## Q15. Tree-shaking services?

**Answer:**

With `providedIn: 'root'`, the bundler can drop a service if nobody injects it. A module `providers` array can keep extra code in the bundle. Standalone plus `providedIn` is friendlier for the bundler.

Tree-shaking is the build step that removes code nobody uses, like shaking a tree so the dead leaves fall off. Smaller bundles mean faster downloads for users.

The difference is about who points to whom. With `providedIn: 'root'`, the service registers itself. If no component ever asks for it, nothing points to it and the bundler can drop it. When a service is listed in a module's `providers` array, the module points to it, so it stays in the bundle as long as that module is used, even if no one injects it.

**Cross-question:** Side-effect imports?

**Cross-answer:**

An `import 'foo'` with no symbols might be kept or dropped, depending on the config. I avoid random side-effect imports.

A side-effect import is one that runs a file just for what it does when loaded, like adding a polyfill or registering something globally. The bundler can't tell from the import line whether that code is needed, so the result depends on how the package and build are configured. Either the code is dropped and something silently breaks, or it's kept when no one needs it. Importing named things makes it clear what's used.

---

## Q16. `linkedSignal` in simple words?

**Answer:**

It's a writable signal that also resets when something else changes. For example, the selected id becomes invalid when the list reloads.

Picture a dropdown of products. The user picks one, so the selected product is something the user can change. But if the product list is reloaded, the old choice might not exist anymore. A `linkedSignal` handles both: the user can set it, and when the list signal changes, it recalculates a sensible value, like the first item in the new list.

Without it, I'd need a normal signal plus an effect that watches the list and resets the selection. That's the "effect that sets a signal" pattern I try to avoid.

**Cross-question:** vs computed?

**Cross-answer:**

Computed is read-only. A linked signal is for "derived, but the user can also change it".

With computed, the value always comes from the formula and I can't `set` it. With `linkedSignal`, the formula gives the starting value, and I can still `set` it when the user picks something. When its source changes, the formula runs again.

---

## Q17. Control flow vs structural directives — migration pain?

**Answer:**

`@if` narrows types better. `*ngIf` still works. `as` and else templates map over to `@else`. I migrate screen by screen, not in one big bang.

`@if`, `@for` and `@switch` are Angular's newer built-in template syntax. `*ngIf` and `*ngFor` are the older directives that do the same jobs. "Narrows types better" means that inside `@if (user)`, TypeScript knows `user` isn't null, so I get fewer false errors and don't need workarounds.

The old `else` needed a separate `<ng-template>` with a name, and the new syntax just uses an `@else` block, which is easier to read. I migrate one screen at a time because a small change is easy to review and test. If something breaks, I know exactly where to look.

**Cross-question:** Mix both?

**Cross-answer:**

Yes, during the migration. I just don't mix them in the same small template for no reason.

Both styles work side by side, so the app keeps running while it's half migrated. Within one template, though, mixing them makes it harder to read, so when I touch a template I usually convert the whole thing.

---

## Q18. How do you keep bundle size down?

**Answer:**

I use lazy routes, I don't import all of lodash, and I check the bundle with source-map-explorer. I watch PrimeNG/Material imports, and I avoid the huge moment.js (I use date-fns or native Intl instead).

The bundle is the JavaScript the user downloads before the app works. Every extra kilobyte is slower on a weak phone or a slow network. Lazy routes split the app so each page's code only loads when the user goes there. Importing one function from lodash, instead of the whole library, keeps unused helpers out.

source-map-explorer draws a map of the bundle, showing which library takes how much space, so I can see the real problem instead of guessing. For dates, moment.js is large and can't be trimmed well, while date-fns lets me import only the functions I use, and the browser's built-in Intl costs nothing to download.

**Cross-question:** First paint still slow?

**Cross-answer:**

Then I look at SSR, fewer fonts, `NgOptimizedImage`, and less JS on the home page.

First paint is the moment the user first sees something on the screen. SSR sends ready HTML so content shows before the JavaScript runs. Every font file is another download that can hold up text. `NgOptimizedImage` is Angular's image directive that helps with things like loading important images first and lazy-loading the rest. And the home page is what everyone sees first, so keeping its code small helps the most users.

---

## Q19. Secondary router outlets / matrix params — do you use them?

**Answer:**

Rarely. Most products just need a simple URL. Fancy outlets confuse people and analytics. For filters, I use query params.

A secondary outlet lets one URL drive two separate areas of the page at once, like a main page and a side panel. Matrix params are extra values put inside a URL segment with semicolons, like `/products;color=red`. They're powerful, but the URLs end up looking strange, people don't trust them when sharing, and analytics tools often group them badly.

Query params, like `/products?color=red`, are what everyone already knows. They're easy to read, share, and track.

**Cross-question:** Filters in query params vs in a service only?

**Cross-answer:**

With query params, you can share the URL and refresh the page. With a service only, a refresh loses the filters. For lists, I prefer the URL.

A service lives in memory, so it's wiped when the page reloads. If filters are in the URL, a user can bookmark "open incidents for this week" or send it to a colleague, and it opens with the same filters. The back button also works as people expect.

---

## Q20. How would you handle a 401 in a SPA?

**Answer:**

An interceptor catches the 401 and tries a refresh once, then retries the call or redirects to login. I clear the in-memory user state. I don't let it loop on refresh forever.

A 401 means "I don't know who you are", usually because the access token has expired. An interceptor is code that sits between the app and every HTTP call, so I can handle this in one place instead of in every component. It asks the server for a new token using the refresh token. If that works, it repeats the original request and the user never notices.

If the refresh also fails, the session is really over. I clear the user data the app holds in memory so the screen doesn't keep showing someone who isn't logged in, and send them to the login page. The "only once" rule matters: if the refresh call itself gets a 401 and that triggers another refresh, the app can loop forever and hammer the server.

**Cross-question:** 403?

**Cross-answer:**

That means the user is logged in but not allowed. I show a "no access" message, not the login page.

A 403 is like a building pass that works at the front door but not for the server room. Logging in again won't help, because the server already knows who they are. Sending them to login would just confuse them, so a clear "you don't have access to this" message is better.

---

## Q21. Signals in a service as the store?

**Answer:**

I keep a private `_items = signal<Item[]>([])` and expose a public `items = this._items.asReadonly()`. Methods change it with `.update`. Components only read. It's simple and easy to test.

A store is the one place where shared app data lives. Here, the service holds the list in a private signal, so only the service can change it. Components get a read-only version. They can show the data and react to changes, but they can't change it directly.

To change the data, a component calls a method on the service, like `addItem()`. That means every change goes through a small set of methods I can find and test. If the list ends up wrong, I only have a few places to look, instead of searching every component that might have touched it.

**Cross-question:** Mutate the array in place?

**Cross-answer:**

No. I replace the array so consumers notice the change. It's the same rule as with OnPush.

By default, a signal decides whether something changed by checking if it's the same object. If I `push` into the existing array, it's still the same array, so the signal may not tell anyone. Creating a new array, like `[...items, newItem]`, gives a new reference, so everyone reading it updates. OnPush components work the same way with inputs.

---

## Q22. What is hydration skipping / incremental hydration?

**Answer:**

You don't hydrate a heavy widget until the user needs it. That saves main-thread work on the first load.

Normally hydration makes the whole server-rendered page interactive at once. Incremental hydration lets me mark some parts to wait, for example until they scroll into view or the user interacts with them. The HTML is already visible, so the user sees it. Angular just doesn't spend time wiring it up yet.

The main thread is where the browser runs JavaScript and also responds to clicks and scrolling. If it's busy hydrating a big chart at the bottom of the page, the page feels frozen. Delaying that work keeps the first load responsive.

**Cross-question:** Risk?

**Cross-answer:**

Until it hydrates, it may not respond to the user. So I skip things below the fold, not the main button.

"Below the fold" means the part of the page you only see after scrolling. Those parts can wait. The main call-to-action button is the first thing users try to click, so it has to work immediately.

---

## Q23. Testing signals?

**Answer:**

I set inputs, call methods, and assert on `component.count()`. For effects, I flush the testing scheduler if needed. For important UX, I still test through the template.

Signals are easy to test because reading one is just calling a function. I change something, then check the new value directly. Effects are different: they don't run instantly when a signal changes, Angular schedules them to run a bit later. In a test, I may need to tell Angular to run pending work, for example with `TestBed.flushEffects()` or by running change detection, before I check the result.

Checking a signal's value proves the logic is right, but not that the user sees it. So for important behaviour, like an error message showing up, I also check the rendered HTML.

**Cross-question:** Fake time?

**Cross-answer:**

Yes, for debounce. Otherwise the tests get slow or flaky.

If a search waits 300ms after typing, a real test would actually have to wait. That adds up across many tests, and timing on a busy build server can make tests pass or fail randomly. Fake time, like Angular's `fakeAsync` with `tick()`, lets the test jump the clock forward instantly and gives the same result every run.

---

## Q24. How do you think about accessibility in Angular?

**Answer:**

Labels on inputs, moving focus after errors, keyboard support on custom widgets, `alt` on images, and not relying on color alone. The CDK has a11y helpers for this. I don't ship a click-only div as a button.

Accessibility (often written a11y) means people can use the app however they use a computer: with a screen reader, only a keyboard, or with low vision or colour blindness. Some examples:

- A label lets a screen reader say "Email" instead of just "edit text".
- Moving focus to the first error after a failed submit tells keyboard and screen reader users what went wrong.
- `alt` text describes an image to someone who can't see it.
- If an error is shown only by a red border, a colour-blind user may miss it, so I add text or an icon too.

The Angular CDK (Component Dev Kit) has ready-made helpers, like keeping focus inside a dialog while it's open. That's hard to get right by hand.

**Cross-question:** `div (click)` as a button?

**Cross-answer:**

Use a `<button>`. Then screen readers and Enter/Space just work. CSS can still make it look like a tile.

A `div` with a click handler only works with a mouse. Keyboard users can't tab to it, pressing Enter does nothing, and a screen reader doesn't announce it as a button. A real `<button>` gets all of that from the browser for free. Styling doesn't depend on the tag, so it can look exactly the same.

---

## Q25. Standalone + provideHttpClient + interceptors?

**Answer:**

I call `bootstrapApplication` with `provideHttpClient(withInterceptors([authInterceptor]))`. Functional interceptors are just functions, which is easier than the old DI classes.

In a standalone app there's no root `AppModule`. Instead, `bootstrapApplication` starts the app and I pass in the providers it needs. `provideHttpClient` sets up Angular's HTTP client, and `withInterceptors` gives it a list of interceptors, the functions that see every request and response.

Earlier, an interceptor had to be a class that implemented an interface and was registered with a special DI token. Now it can be a plain function that takes the request and the next handler. There's less boilerplate, it's easier to read, and it's easier to test.

**Cross-question:** Order of interceptors?

**Cross-answer:**

The first one listed is the outer one. Auth then logging, or the other way round — I pick one order and stay consistent. Errors should still be handled in only one place.

Think of the interceptors as layers of an onion around the actual HTTP call. The request passes through them in the order they're listed, and the response comes back through them in reverse. The order changes what each one sees, for example whether the logger sees the auth header. If two interceptors both show error messages, the user gets two pop-ups for one failure, so only one of them should handle errors.


---
## Additional Questions
### Q26. When would you use `computed` instead of `effect`?
**Answer:** If I need a value derived from other signals, I use computed. I use effect for side effects, like logging or syncing with browser APIs.

A quick test I use: does it produce a value the template or other code reads? Then it's computed, for example a full name made from first and last name. Does it do something outside Angular, like writing to localStorage or calling an analytics tool? Then it's an effect. Computed is also lazy and cached, so it only recalculates when its inputs change and someone actually reads it.
### Q27. What is hydration good for?
**Answer:** The server sends useful HTML first, and the browser attaches Angular behavior to it. For the right kind of app, that can improve first paint and SEO.

"The right kind of app" is mostly public pages where speed of the first view and search ranking matter, like a product page or a blog. For an internal dashboard behind a login, search engines never see it, so the extra server setup may not be worth it. Hydration's own job is to avoid redrawing the server HTML, so there's no flicker and no wasted work when Angular takes over.
### Q28. What is a signal equality function for?
**Answer:** It decides when Angular treats a new signal value as different. I use it carefully, because hiding real changes behind custom equality can leave the UI stale.

By default, a signal compares the old and new value by reference, meaning "is it the very same object?". If I set a new object with identical contents, the signal still counts it as a change and updates everyone. An equality function lets me say, for example, "treat two users as equal if their id and name match", which avoids pointless updates. The risk is that if my function ignores a field that actually changed, the screen won't update and the user sees old data.
