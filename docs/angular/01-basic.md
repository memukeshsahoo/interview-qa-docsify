# Angular — Basic

Components, templates, services. Talk like you built a screen yesterday.

---

## Q1. What is Angular? Different from AngularJS? Different from React?

**Answer:**

Angular is a TypeScript framework for building single-page apps. A single-page app loads one HTML page and then swaps screens inside it, instead of asking the server for a new page on every click. Out of the box it gives you components, a router, forms, HTTP and DI (dependency injection, where Angular creates services and hands them to you).

AngularJS is the old 1.x version. It is a completely different thing. The names are close, but the code, the concepts and the upgrade path are all different, so knowing one does not mean you know the other.

React is a UI library, so you pick the rest yourself. Angular comes with a lot more built in. I think of it like this: Angular is a fully furnished flat, React is an empty flat where you buy your own furniture. Both are fine. One saves decisions, the other gives you more choice.

**Cross-question:** What does “opinionated” mean?

**Cross-answer:**

It means there is a usual way to do things: where files go, how you call an API, how you build forms. That helps a team stay consistent, because a new developer can open any Angular project and find their way around. The downside is that the first week feels heavier, since you have to learn the "Angular way" before you get going.

---

## Q2. What is a component?

**Answer:**

A component is a class plus HTML plus CSS. The class holds the data, and the template shows it. The `@Component` decorator is what marks it as a component. A decorator is just a label on top of the class that tells Angular "treat this as a component, and here is its HTML and CSS".

Components are the building blocks of the screen. A page is made of smaller components, like a header, a search box and a product card. It is like building with Lego: each piece is small and does one job, and you can reuse the same piece in many places.

**Cross-question:** Smart vs dumb component?

**Cross-answer:**

A dumb component just has inputs and outputs, with no HTTP. It gets data from its parent, shows it, and tells the parent when something happens (like a click). A smart component talks to services and the router, so it knows where the data comes from.

I keep things like list items dumb when I can. Dumb components are easier to test and reuse, because they don't care where the data came from. The smart one is like a manager who fetches the work, and the dumb ones just do what they're given.

---

## Q3. NgModule vs standalone?

**Answer:**

Older apps wrap everything in an `NgModule`. An NgModule is a box where you list which components belong together and what they can use. Newer apps use standalone components, where each component imports only what it needs.

The module approach meant an extra file to keep in sync, and it was easy to get "this component is not known" errors when something was declared in the wrong module. With standalone, each component lists its own imports right next to it, so it is clearer what it depends on. For new projects, I go standalone.

**Cross-question:** Do I still need `AppModule`?

**Cross-answer:**

Not in the new project template. The app starts directly from a standalone root component instead. But you will still see it in older repos, so it is worth knowing both.

---

## Q4. Binding types?

**Answer:**

Binding is how the template and the component class stay connected, so data on the screen matches the data in code.

`{{ name }}` shows text.

`[value]="name"` sets a property.

`(click)="save()"` listens to an event.

`[(ngModel)]="name"` binds both ways, mostly in forms.

An easy way to remember the brackets: square brackets `[ ]` send data from the class to the screen, round brackets `( )` send events from the screen back to the class, and `[( )]` (the "banana in a box") does both. So when a user types in an input with `[(ngModel)]`, the variable updates, and when code changes the variable, the input updates too.

**Cross-question:** `[value]` vs `value` vs `[attr.value]`?

**Cross-answer:**

`[value]` binds to the DOM property. Plain `value="..."` is just a static attribute. `[attr.*]` sets the HTML attribute. They are not always the same thing.

The attribute is what was written in the HTML at the start. The property is the live value the browser keeps in memory. For an input box, the attribute is the starting text, but the property changes as the user types. So if you want the current value, you bind the property. You use `[attr.*]` for things that have no matching property, like `aria-*` or `colspan`.

---

## Q5. `*ngIf` / `*ngFor` vs `@if` / `@for`?

**Answer:**

Both add or remove elements from the DOM (the live page structure in the browser). `*ngIf` shows something only when a condition is true, and `*ngFor` repeats something for each item in a list. Newer Angular prefers `@if` and `@for`, which are built into the template syntax, so they read more like normal code and you don't need to import anything for them.

In a list, I always `track` a stable id so Angular can reuse the rows.

**Cross-question:** Why track?

**Cross-answer:**

Without it, a refresh can rebuild every row. You lose focus, and Angular does extra work for nothing.

With a stable id, Angular can tell "this is the same order #42 as before" and keep that row as it is, only changing what really changed. Think of a teacher taking attendance by roll number instead of asking everyone to leave and come back in.

---

## Q6. What is a service? How do you inject it?

**Answer:**

A service is a class marked with `@Injectable` that holds logic or HTTP calls. I inject it through the constructor or with `inject()`. `providedIn: 'root'` means there is one instance for the whole app.

"Inject" means I don't create the service myself with `new`. I just ask for it, and Angular hands me the shared instance. This keeps components small: the component handles the screen, and the service handles things like calling the API or holding the logged-in user. It also makes testing easy, because in a test I can hand the component a fake service instead.

A simple comparison: a service is like the office printer. Everyone uses the same one, nobody brings their own from home.

**Cross-question:** Provide on a component instead?

**Cross-answer:**

Then each component instance gets its own copy. That is useful for something like a wizard’s local state, where each open wizard should keep its own steps and answers, and the data should go away when the wizard closes. I do not re-provide `HttpClient` myself, though.

---

## Q7. CLI commands?

**Answer:**

The Angular CLI is a command-line tool that creates, runs and builds Angular projects for you, so you don't set it all up by hand.

`ng serve` to run locally, `ng build` for a production build, `ng generate component` to create a component, and `ng test` to run tests.

`ng generate` also saves typing and keeps files named and placed the same way across the team.

**Cross-question:** Deploy `ng serve`?

**Cross-answer:**

No. That is only a dev server. It is built for fast reloads while coding, not for speed or safety with real users. In production, you deploy the output of `ng build`, which is a set of small, optimized static files that any web server can host.

---

## Q8. Router?

**Answer:**

The router maps a URL to a component. `router-outlet` is the slot where the page shows up. `routerLink` changes the URL without a full page reload.

So `/products` shows the product list component and `/cart` shows the cart component. The user still gets normal URLs they can bookmark and share, and the back button works, but the browser never reloads the whole app. It feels fast, like switching tabs instead of opening a new window.

**Cross-question:** `href` vs `routerLink`?

**Cross-answer:**

`href` reloads the whole app. `routerLink` stays inside Angular. With `href`, the browser throws away everything in memory and downloads the app again, so you lose state and it feels slow. I use `href` only for links to outside sites.

---

## Q9. Template forms vs reactive forms?

**Answer:**

Template forms use `ngModel` and are quick to set up. Most of the form is described in the HTML, which is fine for a small login or contact form.

Reactive forms build a `FormGroup` in TypeScript. They are better for testing, dynamic fields and bigger forms.

With reactive forms, the whole form (its fields, starting values and rules) lives in code. That means I can add or remove fields at runtime, react to a value changing, and test the form without rendering any HTML. A simple rule: small and simple, template form; big or changing, reactive form.

**Cross-question:** Where do validators run?

**Cross-answer:**

On the form control. The UI shows the errors. But the API still validates on its side too.

Browser checks are for a nicer experience, like telling the user "email is required" right away. Anyone can skip them by calling the API directly, so the server must never trust them.

---

## Q10. How do you call an API?

**Answer:**

I inject `HttpClient` and call `get`/`post`. That gives me back an Observable. An Observable is like a subscription to a stream: nothing happens until someone subscribes, and then values arrive when they're ready.

I prefer the `async` pipe or `toSignal`, so I do not forget to unsubscribe. If I subscribe by hand and forget to clean up, the subscription can stay alive after the user leaves the page and cause bugs or memory leaks. The `async` pipe and `toSignal` clean up for me.

**Cross-question:** Why not `fetch` in the component?

**Cross-answer:**

Because with `HttpClient` I get interceptors, easier tests and typed JSON. It is the default choice in Angular.

Interceptors let me add things like the auth header in one place for every call. In tests, Angular gives me a way to fake `HttpClient` responses without a real server. With raw `fetch`, I would have to build all of that myself.

---

## Q11. What is an interceptor?

**Answer:**

It is a hook that wraps every HTTP call. I use it to add the auth header, catch a 401, or show a spinner.

Every request passes through it on the way out, and every response passes through it on the way back. So instead of adding the token in fifty places, I add it once. It is like the security desk at an office building: everyone going in and out passes through the same desk, so the checks happen in one place.

**Cross-question:** JWT in `localStorage`?

**Cross-answer:**

A JWT is the login token the server gives you, which you send back on each request to prove who you are.

A lot of apps do it, but an XSS attack can steal it from there. XSS means someone manages to run their own script on your page, and any script on the page can read `localStorage`. Other options are httpOnly cookies, or keeping it in memory plus a refresh token. An httpOnly cookie can't be read by JavaScript at all, so a script can't copy it. Either way, never log the token and never put it in the URL, because logs and URLs get saved and shared in places you don't control.

---

## Q12. Pipes?

**Answer:**

Pipes change how a value is displayed, like `{{ date | date:'short' }}`. The data stays the same; only what the user sees is formatted. So a raw date becomes something readable, or a number becomes a currency amount.

Pure pipes run only when the input changes. Impure pipes run very often, so I avoid them. An impure pipe runs on every check of the screen, which can be many times a second, so a slow one makes the whole page slow.

**Cross-question:** Pipe vs `{{ format(x) }}`?

**Cross-answer:**

A method call in the template can run on every change detection cycle. A pure pipe is cheaper.

Angular can't know if a method's result has changed, so it calls it again each time it checks the screen. A pure pipe remembers its last input, and if the input is the same, it reuses the last result.

---

## Q13. Change detection in one sentence?

**Answer:**

Angular updates the screen when it thinks the data has changed. Zone.js used to patch timers and HTTP so Angular would know when to check. You can also use signals.

Change detection is how Angular keeps the screen in sync with your data. Zone.js watches things like clicks, timers and HTTP responses, and after any of them, Angular checks the components to see what needs updating. A signal is a value that tells Angular directly when it changes, so Angular doesn't have to guess and can update just the parts that use it.

**Cross-question:** What is `OnPush`?

**Cross-answer:**

It tells Angular to check this component less often: only when an input changes by reference, when an event fires, or when a signal or the async pipe says something changed.

"By reference" means a new object, not the same object edited in place. On a big page, this saves a lot of work, because Angular skips components whose data hasn't changed.

---

## Q14. Environment files?

**Answer:**

I use them for things like different API URLs in dev and prod. When I build for production, Angular swaps in the production file, so the same code points to the right server without me editing it by hand.

But anything in the Angular bundle is public, so no real secrets go there. The bundle is the JavaScript that gets downloaded to every user's browser, and anyone can read it.

**Cross-question:** Hide an API key in environment.ts?

**Cross-answer:**

No. Anyone can open DevTools and see it. Secret keys stay on the server. If the frontend needs something that uses a secret key, the frontend calls my own backend, and the backend calls the other service with the key.

---

## Q15. Lifecycle: `ngOnInit` vs constructor?

**Answer:**

Lifecycle hooks are methods Angular calls at certain moments in a component's life: when it's created, when its inputs change, and when it's removed.

The constructor is for injecting dependencies. `ngOnInit` is where I start loading data. `ngOnDestroy` is where I unsubscribe and clear timers. `ngOnChanges` runs when inputs change.

A simple way to picture it: the constructor is like moving into a new flat and bringing your tools. `ngOnInit` is when the flat is ready and you start working. `ngOnDestroy` is moving out, so you switch off the lights and hand back the keys.

**Cross-question:** HTTP in the constructor?

**Cross-answer:**

I avoid it. Inputs are not ready yet, and it makes tests harder. If the call needs an input value, like a user id, that value doesn't exist yet in the constructor. And in tests, the call fires the moment the component is created, before I have set anything up. I load in `ngOnInit` (or with signals/resources in the newer style).

---

## Q16. What is a directive?

**Answer:**

A directive adds behavior to an element. Structural directives add or remove DOM. Attribute directives change how an element looks or behaves, like a tooltip.

`*ngIf` is a structural directive, because it decides whether an element exists at all. A tooltip or "highlight on hover" directive is an attribute directive: the element is still there, it just gets extra behavior. The nice part is reuse: I write the tooltip once and put it on any element.

**Cross-question:** Component vs directive?

**Cross-answer:**

A component has a template; a directive usually does not. Components are the pages and widgets. Directives add extra behavior on top. In fact, a component is a directive that comes with its own template.

---

## Q17. `@Input` and `@Output`?

**Answer:**

With an input, the parent passes data in. With an output, the child raises an event (`EventEmitter` or output()). The parent listens with `(saved)="..."`.

This is how components talk to each other. Data goes down through inputs, and events come up through outputs. For example, a parent passes a `product` into a product card, and when the user clicks "Add to cart", the card raises an event and the parent decides what to do. It's like a manager giving a task to someone, and that person calling back when it's done.

**Cross-question:** Two-way binding on a child?

**Cross-answer:**

An input called `value` plus an output called `valueChange` gives you `[(value)]`. Still, I prefer one-way binding plus events when that is clearer, because it's easier to see who changed the data and when.

---

## Q18. `ngClass` / `ngStyle`?

**Answer:**

They bind CSS classes or inline styles from data. For example, I can make a row red when an order is overdue, without writing code that touches the DOM.

I prefer classes over a big style object. The styling stays in the CSS file where it belongs, and it's easier to change the look later in one place.

**Cross-question:** `[class.active]="isOn"` vs `ngClass`?

**Cross-answer:**

For a single class, I use `[class.active]`. It adds the class when `isOn` is true and removes it when false. For many, `ngClass`. It is the same idea.

---

## Q19. What is `async` pipe?

**Answer:**

It subscribes to an Observable or Promise right in the template, shows the value, and unsubscribes when the view is destroyed.

Without it, I would subscribe in the class, store the value in a variable, and remember to unsubscribe when the component closes. The `async` pipe does all three for me, so there's less code and no forgotten cleanup.

**Cross-question:** Two `async` pipes on the same HTTP Observable?

**Cross-answer:**

That creates two subscriptions, which can mean two HTTP calls. Each subscription starts the request again.

I use `as` once, or `shareReplay`, or a signal. With `as`, I subscribe once and give the result a name I can use in the rest of the template. `shareReplay` makes all subscribers share one request. A signal holds the value once and any part of the template can read it.

---

## Q20. Lazy loading a route?

**Answer:**

It means a page, like admin, is not in the first bundle. `loadComponent` / `loadChildren` loads it only when you navigate there.

Without it, every user downloads the code for every page up front, even pages they'll never open. With lazy loading, the app is split into smaller files, and each one is downloaded only when needed. It's like a restaurant that cooks your dish when you order it, instead of cooking the whole menu before you arrive.

**Cross-question:** Why bother?

**Cross-answer:**

The first load is faster. Users who never open admin never download it. This matters most on slow mobile networks, where every extra file adds waiting time.

---

## Q21. What is `CommonModule`?

**Answer:**

It is the old shared module that gives you `ngIf`, `ngFor` and the common pipes. With standalone components and the new control flow, you need it less. But you will still see it everywhere.

The new `@if` and `@for` are built into the template syntax, so they don't need an import. Standalone components can also import just the one pipe or directive they use, instead of the whole module.

**Cross-question:** SharedModule with everything?

**Cross-answer:**

A giant SharedModule is a beginner trap. Every component that imports it pulls in everything, even what it doesn't use, and it gets hard to see what depends on what. I import only what each component needs.

---

## Q22. How do you show a loading spinner?

**Answer:**

I use a `loading` flag, or the status of a resource/signal. In the template: `@if (loading()) { ... }`. I also handle the error and empty states, not only the happy path.

So a screen has four states: loading, loaded with data, loaded but empty, and failed. If I only build the happy path, a failed call leaves a spinner going forever, or an empty list looks like a broken page. A short "No orders yet" or "Couldn't load, try again" message tells the user what is going on.

**Cross-question:** Spinner in the interceptor for every call?

**Cross-answer:**

That can flicker. Quick calls make the spinner flash on and off, and it doesn't tell the user which part of the screen is waiting. I prefer a loading state per screen for the main data, and a small global one only for long calls if needed.

---

## Q23. What is `router-outlet`?

**Answer:**

It is the place where the router inserts the current page. Child routes can have their own nested outlets.

Think of the app layout as a picture frame. The header and menu stay put, and the `router-outlet` is the space inside the frame where the picture changes as you move between pages. A nested outlet is a smaller frame inside a page, for example the tabs inside a "Settings" page.

**Cross-question:** More than one outlet?

**Cross-answer:**

Named outlets exist, but most apps need just one. I do not use named outlets unless the layout really needs something like a side panel route, because they make the URLs and the routing setup harder to follow.

---

## Q24. Path `''` vs `**` in the router?

**Answer:**

`''` is the default child or the home route. `**` is the wildcard, for “not found”. The wildcard must come last.

The router checks routes from top to bottom and uses the first one that matches. `**` matches anything, so if it came first, every URL would land on the "not found" page.

**Cross-question:** Redirect?

**Cross-answer:**

I use `redirectTo: 'home'` with `pathMatch: 'full'`, so it does not swallow every URL.

An empty path `''` is a prefix of every URL. Without `pathMatch: 'full'`, the redirect could catch URLs it shouldn't. With it, the redirect only happens when the URL is really empty.

---

## Q25. Why TypeScript with Angular?

**Answer:**

Types catch typos before the code runs. Angular’s templates and DI work well with them. After compiling, the browser still runs plain JavaScript.

A type says what shape a value has, like "this is a user with a name and an email". If I misspell `user.emial`, the editor flags it right away instead of me finding out from a blank screen in production. It also gives better autocomplete and makes refactoring safer, because renaming something shows every place that breaks.

**Cross-question:** `any` everywhere?

**Cross-answer:**

Then you have thrown away the benefit. `any` tells TypeScript to stop checking, so the typos come back. I type my HTTP DTOs (the shapes of the data the API sends and receives). If I truly do not know the type yet, `unknown` is better than `any`, because TypeScript makes me check what it is before I use it.


---
## Additional Questions
### Q26. Signals vs BehaviorSubject?
**Answer:** Signals are Angular’s built-in way to hold reactive state, and they work naturally with templates. BehaviorSubject comes from RxJS and is useful when I need an Observable stream. I pick based on how the data flows, instead of mixing both everywhere.

"Reactive state" means a value that tells whoever is using it when it changes. A signal is simple: you read it, set it, and the template updates. A BehaviorSubject also holds a current value, but you work with it through subscriptions and RxJS operators, which is useful for things like combining streams or adding a debounce. Mixing both for the same data makes the code harder to follow, so I keep each piece of data in one style.
### Q27. Why use `takeUntilDestroyed`?
**Answer:** It cleans up subscriptions automatically when the component is destroyed. It is safer than keeping a Subject around by hand just for teardown.

Without cleanup, a subscription can keep running after the user leaves the page, which wastes memory and can cause odd bugs. The old way was to create a Subject, fire it in `ngOnDestroy`, and pipe every subscription through it. That's easy to forget. `takeUntilDestroyed` does the same job in one line.
### Q28. What should an Angular interceptor not do?
**Answer:** It should not hold business logic for one particular screen. Auth headers, common error handling and other request-level concerns fit well there; feature logic belongs in services and components.

The interceptor runs on every HTTP call in the app. If I put screen-specific rules there, every request pays for that check, and the logic is hidden in a place nobody looks when they work on that screen. Keeping it in the feature's own service makes it easy to find and test.
