# Angular — Basic

Components, templates, services. Talk like you built a screen yesterday.

---

## Q1. What is Angular? Different from AngularJS? Different from React?

**Answer:**

Angular is a TypeScript framework for building single-page apps. Out of the box it gives you components, a router, forms, HTTP and DI (dependency injection, where Angular creates services and hands them to you).

AngularJS is the old 1.x version. It is a completely different thing.

React is a UI library, so you pick the rest yourself. Angular comes with a lot more built in.

**Cross-question:** What does “opinionated” mean?

**Cross-answer:**

It means there is a usual way to do things. That helps a team stay consistent. The downside is that the first week feels heavier.

---

## Q2. What is a component?

**Answer:**

A component is a class plus HTML plus CSS. The class holds the data, and the template shows it. The `@Component` decorator is what marks it as a component.

**Cross-question:** Smart vs dumb component?

**Cross-answer:**

A dumb component just has inputs and outputs, with no HTTP. A smart component talks to services and the router. I keep things like list items dumb when I can.

---

## Q3. NgModule vs standalone?

**Answer:**

Older apps wrap everything in an `NgModule`. Newer apps use standalone components, where each component imports only what it needs. For new projects, I go standalone.

**Cross-question:** Do I still need `AppModule`?

**Cross-answer:**

Not in the new project template. But you will still see it in older repos, so it is worth knowing both.

---

## Q4. Binding types?

**Answer:**

`{{ name }}` shows text.

`[value]="name"` sets a property.

`(click)="save()"` listens to an event.

`[(ngModel)]="name"` binds both ways, mostly in forms.

**Cross-question:** `[value]` vs `value` vs `[attr.value]`?

**Cross-answer:**

`[value]` binds to the DOM property. Plain `value="..."` is just a static attribute. `[attr.*]` sets the HTML attribute. They are not always the same thing.

---

## Q5. `*ngIf` / `*ngFor` vs `@if` / `@for`?

**Answer:**

Both add or remove elements from the DOM. Newer Angular prefers `@if` and `@for`. In a list, I always `track` a stable id so Angular can reuse the rows.

**Cross-question:** Why track?

**Cross-answer:**

Without it, a refresh can rebuild every row. You lose focus, and Angular does extra work for nothing.

---

## Q6. What is a service? How do you inject it?

**Answer:**

A service is a class marked with `@Injectable` that holds logic or HTTP calls. I inject it through the constructor or with `inject()`. `providedIn: 'root'` means there is one instance for the whole app.

**Cross-question:** Provide on a component instead?

**Cross-answer:**

Then each component instance gets its own copy. That is useful for something like a wizard’s local state. I do not re-provide `HttpClient` myself, though.

---

## Q7. CLI commands?

**Answer:**

`ng serve` to run locally, `ng build` for a production build, `ng generate component` to create a component, and `ng test` to run tests.

**Cross-question:** Deploy `ng serve`?

**Cross-answer:**

No. That is only a dev server. In production, you deploy the output of `ng build`.

---

## Q8. Router?

**Answer:**

The router maps a URL to a component. `router-outlet` is the slot where the page shows up. `routerLink` changes the URL without a full page reload.

**Cross-question:** `href` vs `routerLink`?

**Cross-answer:**

`href` reloads the whole app. `routerLink` stays inside Angular. I use `href` only for links to outside sites.

---

## Q9. Template forms vs reactive forms?

**Answer:**

Template forms use `ngModel` and are quick to set up.

Reactive forms build a `FormGroup` in TypeScript. They are better for testing, dynamic fields and bigger forms.

**Cross-question:** Where do validators run?

**Cross-answer:**

On the form control. The UI shows the errors. But the API still validates on its side too.

---

## Q10. How do you call an API?

**Answer:**

I inject `HttpClient` and call `get`/`post`. That gives me back an Observable. I prefer the `async` pipe or `toSignal`, so I do not forget to unsubscribe.

**Cross-question:** Why not `fetch` in the component?

**Cross-answer:**

Because with `HttpClient` I get interceptors, easier tests and typed JSON. It is the default choice in Angular.

---

## Q11. What is an interceptor?

**Answer:**

It is a hook that wraps every HTTP call. I use it to add the auth header, catch a 401, or show a spinner.

**Cross-question:** JWT in `localStorage`?

**Cross-answer:**

A lot of apps do it, but an XSS attack can steal it from there. Other options are httpOnly cookies, or keeping it in memory plus a refresh token. Either way, never log the token and never put it in the URL.

---

## Q12. Pipes?

**Answer:**

Pipes change how a value is displayed, like `{{ date | date:'short' }}`. Pure pipes run only when the input changes. Impure pipes run very often, so I avoid them.

**Cross-question:** Pipe vs `{{ format(x) }}`?

**Cross-answer:**

A method call in the template can run on every change detection cycle. A pure pipe is cheaper.

---

## Q13. Change detection in one sentence?

**Answer:**

Angular updates the screen when it thinks the data has changed. Zone.js used to patch timers and HTTP so Angular would know when to check. You can also use signals.

**Cross-question:** What is `OnPush`?

**Cross-answer:**

It tells Angular to check this component less often: only when an input changes by reference, when an event fires, or when a signal or the async pipe says something changed.

---

## Q14. Environment files?

**Answer:**

I use them for things like different API URLs in dev and prod. But anything in the Angular bundle is public, so no real secrets go there.

**Cross-question:** Hide an API key in environment.ts?

**Cross-answer:**

No. Anyone can open DevTools and see it. Secret keys stay on the server.

---

## Q15. Lifecycle: `ngOnInit` vs constructor?

**Answer:**

The constructor is for injecting dependencies. `ngOnInit` is where I start loading data. `ngOnDestroy` is where I unsubscribe and clear timers. `ngOnChanges` runs when inputs change.

**Cross-question:** HTTP in the constructor?

**Cross-answer:**

I avoid it. Inputs are not ready yet, and it makes tests harder. I load in `ngOnInit` (or with signals/resources in the newer style).

---

## Q16. What is a directive?

**Answer:**

A directive adds behavior to an element. Structural directives add or remove DOM. Attribute directives change how an element looks or behaves, like a tooltip.

**Cross-question:** Component vs directive?

**Cross-answer:**

A component has a template; a directive usually does not. Components are the pages and widgets. Directives add extra behavior on top.

---

## Q17. `@Input` and `@Output`?

**Answer:**

With an input, the parent passes data in. With an output, the child raises an event (`EventEmitter` or output()). The parent listens with `(saved)="..."`.

**Cross-question:** Two-way binding on a child?

**Cross-answer:**

An input called `value` plus an output called `valueChange` gives you `[(value)]`. Still, I prefer one-way binding plus events when that is clearer.

---

## Q18. `ngClass` / `ngStyle`?

**Answer:**

They bind CSS classes or inline styles from data. I prefer classes over a big style object.

**Cross-question:** `[class.active]="isOn"` vs `ngClass`?

**Cross-answer:**

For a single class, I use `[class.active]`. For many, `ngClass`. It is the same idea.

---

## Q19. What is `async` pipe?

**Answer:**

It subscribes to an Observable or Promise right in the template, shows the value, and unsubscribes when the view is destroyed.

**Cross-question:** Two `async` pipes on the same HTTP Observable?

**Cross-answer:**

That creates two subscriptions, which can mean two HTTP calls. I use `as` once, or `shareReplay`, or a signal.

---

## Q20. Lazy loading a route?

**Answer:**

It means a page, like admin, is not in the first bundle. `loadComponent` / `loadChildren` loads it only when you navigate there.

**Cross-question:** Why bother?

**Cross-answer:**

The first load is faster. Users who never open admin never download it.

---

## Q21. What is `CommonModule`?

**Answer:**

It is the old shared module that gives you `ngIf`, `ngFor` and the common pipes. With standalone components and the new control flow, you need it less. But you will still see it everywhere.

**Cross-question:** SharedModule with everything?

**Cross-answer:**

A giant SharedModule is a beginner trap. I import only what each component needs.

---

## Q22. How do you show a loading spinner?

**Answer:**

I use a `loading` flag, or the status of a resource/signal. In the template: `@if (loading()) { ... }`. I also handle the error and empty states, not only the happy path.

**Cross-question:** Spinner in the interceptor for every call?

**Cross-answer:**

That can flicker. I prefer a loading state per screen for the main data, and a small global one only for long calls if needed.

---

## Q23. What is `router-outlet`?

**Answer:**

It is the place where the router inserts the current page. Child routes can have their own nested outlets.

**Cross-question:** More than one outlet?

**Cross-answer:**

Named outlets exist, but most apps need just one. I do not use named outlets unless the layout really needs something like a side panel route.

---

## Q24. Path `''` vs `**` in the router?

**Answer:**

`''` is the default child or the home route. `**` is the wildcard, for “not found”. The wildcard must come last.

**Cross-question:** Redirect?

**Cross-answer:**

I use `redirectTo: 'home'` with `pathMatch: 'full'`, so it does not swallow every URL.

---

## Q25. Why TypeScript with Angular?

**Answer:**

Types catch typos before the code runs. Angular’s templates and DI work well with them. After compiling, the browser still runs plain JavaScript.

**Cross-question:** `any` everywhere?

**Cross-answer:**

Then you have thrown away the benefit. I type my HTTP DTOs. If I truly do not know the type yet, `unknown` is better than `any`.


---
## Additional Questions
### Q26. Signals vs BehaviorSubject?
**Answer:** Signals are Angular’s built-in way to hold reactive state, and they work naturally with templates. BehaviorSubject comes from RxJS and is useful when I need an Observable stream. I pick based on how the data flows, instead of mixing both everywhere.
### Q27. Why use `takeUntilDestroyed`?
**Answer:** It cleans up subscriptions automatically when the component is destroyed. It is safer than keeping a Subject around by hand just for teardown.
### Q28. What should an Angular interceptor not do?
**Answer:** It should not hold business logic for one particular screen. Auth headers, common error handling and other request-level concerns fit well there; feature logic belongs in services and components.
