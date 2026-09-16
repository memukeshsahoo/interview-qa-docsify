# Angular — Basic

Components, templates, services. Talk like you built a screen yesterday.

---

## Q1. What is Angular? Different from AngularJS? Different from React?

**Answer:**

Angular is a TypeScript framework for single-page apps. It gives you components, a router, forms, HTTP, and DI.

AngularJS is the old 1.x. Different thing.

React is a UI library. You pick the rest. Angular comes with more batteries.

**Cross-question:** What does “opinionated” mean?

**Cross-answer:**

There is a usual way to do things. That helps a team. The first week is heavier.

---

## Q2. What is a component?

**Answer:**

A class plus HTML plus CSS. The class holds data. The template shows it. `@Component` marks it.

**Cross-question:** Smart vs dumb component?

**Cross-answer:**

Dumb: inputs and outputs, no HTTP. Smart: talks to services and the router. I keep list items dumb when I can.

---

## Q3. NgModule vs standalone?

**Answer:**

Old apps wrap things in `NgModule`. New apps use standalone components that import what they need. New projects should be standalone.

**Cross-question:** Do I still need `AppModule`?

**Cross-answer:**

Not in the new template. You will still see it in older repos. Know both.

---

## Q4. Binding types?

**Answer:**

`{{ name }}` shows text.

`[value]="name"` sets a property.

`(click)="save()"` listens.

`[(ngModel)]="name"` is both ways, mostly forms.

**Cross-question:** `[value]` vs `value` vs `[attr.value]`?

**Cross-answer:**

`[value]` is the DOM property. Plain `value="..."` is a static attribute. `[attr.*]` sets the attribute. They are not always the same.

---

## Q5. `*ngIf` / `*ngFor` vs `@if` / `@for`?

**Answer:**

They add or remove DOM. New Angular prefers `@if` and `@for`. Always `track` a stable id in a list so rows are reused.

**Cross-question:** Why track?

**Cross-answer:**

Without it, a refresh can rebuild every row. You lose focus and extra work happens.

---

## Q6. What is a service? How do you inject it?

**Answer:**

A class with `@Injectable` that holds logic or HTTP. I inject it in the constructor or with `inject()`. `providedIn: 'root'` means one for the app.

**Cross-question:** Provide on a component instead?

**Cross-answer:**

Then each component instance gets its own copy. Useful for a wizard’s local state. I do not re-provide `HttpClient` myself.

---

## Q7. CLI commands?

**Answer:**

`ng serve` for local. `ng build` for prod. `ng generate component`. `ng test`.

**Cross-question:** Deploy `ng serve`?

**Cross-answer:**

No. That is a dev server. Production is `ng build` output.

---

## Q8. Router?

**Answer:**

URL → component. `router-outlet` is the hole where the page appears. `routerLink` changes the URL without a full reload.

**Cross-question:** `href` vs `routerLink`?

**Cross-answer:**

`href` reloads the whole app. `routerLink` stays inside Angular. Use `href` for outside links.

---

## Q9. Template forms vs reactive forms?

**Answer:**

Template: `ngModel`, quick.

Reactive: `FormGroup` in TypeScript. Better for tests, dynamic fields, and bigger forms.

**Cross-question:** Where do validators run?

**Cross-answer:**

On the control. The UI shows errors. The API still validates.

---

## Q10. How do you call an API?

**Answer:**

Inject `HttpClient`, `get`/`post`. You get an Observable. I prefer `async` pipe or `toSignal` so I do not forget to unsubscribe.

**Cross-question:** Why not `fetch` in the component?

**Cross-answer:**

Interceptors, tests, and typed JSON. `HttpClient` is the default.

---

## Q11. What is an interceptor?

**Answer:**

A hook around HTTP: add the auth header, catch 401, show a spinner.

**Cross-question:** JWT in `localStorage`?

**Cross-answer:**

Many apps do it. XSS can steal it. Other options: httpOnly cookies, or memory plus refresh. Never log the token. Never put it in the URL.

---

## Q12. Pipes?

**Answer:**

Change how something looks: `{{ date | date:'short' }}`. Pure pipes run when the input changes. Impure pipes run a lot — avoid them.

**Cross-question:** Pipe vs `{{ format(x) }}`?

**Cross-answer:**

A method in the template can run on every change detection. A pure pipe is cheaper.

---

## Q13. Change detection in one sentence?

**Answer:**

Angular updates the screen when it thinks data changed. Zone.js used to patch timers and HTTP so Angular knows. You can also use signals.

**Cross-question:** What is `OnPush`?

**Cross-answer:**

Check this component less often: when inputs change by reference, when an event fires, or when a signal/async pipe says so.

---

## Q14. Environment files?

**Answer:**

Different API URLs for dev and prod. Anything in the Angular bundle is public. No real secrets here.

**Cross-question:** Hide an API key in environment.ts?

**Cross-answer:**

No. Users can open DevTools. Secret keys stay on the server.

---

## Q15. Lifecycle: `ngOnInit` vs constructor?

**Answer:**

Constructor: inject. `ngOnInit`: start loading. `ngOnDestroy`: unsubscribe and clear timers. `ngOnChanges`: inputs changed.

**Cross-question:** HTTP in the constructor?

**Cross-answer:**

Inputs are not ready. Tests get harder. I load in `ngOnInit` (or with signals/resources in newer style).

---

## Q16. What is a directive?

**Answer:**

A way to add behavior to an element. Structural: add/remove DOM. Attribute: change look or behavior, like a tooltip.

**Cross-question:** Component vs directive?

**Cross-answer:**

A component has a template. A directive usually does not. Components are the pages and widgets. Directives are extra behavior.

---

## Q17. `@Input` and `@Output`?

**Answer:**

Input: parent passes data in. Output: child raises an event (`EventEmitter` or output()). Parent listens with `(saved)="..."`.

**Cross-question:** Two-way binding on a child?

**Cross-answer:**

Input `value` plus output `valueChange` gives `[(value)]`. I still prefer one-way plus events when it is clearer.

---

## Q18. `ngClass` / `ngStyle`?

**Answer:**

Bind CSS classes or inline styles from data. Prefer classes over a big style object.

**Cross-question:** `[class.active]="isOn"` vs `ngClass`?

**Cross-answer:**

Single class: `[class.active]`. Many: `ngClass`. Same idea.

---

## Q19. What is `async` pipe?

**Answer:**

It subscribes to an Observable or Promise in the template, shows the value, and unsubscribes when the view dies.

**Cross-question:** Two `async` pipes on the same HTTP Observable?

**Cross-answer:**

Two subscriptions, maybe two HTTP calls. Use `as` once, or `shareReplay`, or a signal.

---

## Q20. Lazy loading a route?

**Answer:**

The admin page is not in the first bundle. `loadComponent` / `loadChildren` loads it when you navigate.

**Cross-question:** Why bother?

**Cross-answer:**

Faster first load. Users who never open admin never download it.

---

## Q21. What is `CommonModule`?

**Answer:**

Old shared module with `ngIf`, `ngFor`, pipes. Standalone + new control flow needs less of it. You will still see it everywhere.

**Cross-question:** SharedModule with everything?

**Cross-answer:**

A giant SharedModule is a beginner trap. Import only what the component needs.

---

## Q22. How do you show a loading spinner?

**Answer:**

A `loading` flag or the status of a resource/signal. Template: `@if (loading()) { ... }`. I also handle error and empty. Not only the happy path.

**Cross-question:** Spinner in the interceptor for every call?

**Cross-answer:**

Can flicker. I prefer per-screen loading for the main data, and a small global one only for long calls if needed.

---

## Q23. What is `router-outlet`?

**Answer:**

The place the router inserts the current page. Child routes can have nested outlets.

**Cross-question:** More than one outlet?

**Cross-answer:**

Named outlets exist. Most apps need one. I do not use named outlets unless the layout really needs a side panel route.

---

## Q24. Path `''` vs `**` in the router?

**Answer:**

`''` is the default child or home. `**` is wildcard — not found. Wildcard must be last.

**Cross-question:** Redirect?

**Cross-answer:**

`redirectTo: 'home'` with `pathMatch: 'full'` so it does not swallow every URL.

---

## Q25. Why TypeScript with Angular?

**Answer:**

Types catch typos before runtime. Angular’s templates and DI work well with them. The browser still runs JavaScript after compile.

**Cross-question:** `any` everywhere?

**Cross-answer:**

Then you threw the benefit away. I type HTTP DTOs. `unknown` is better than `any` if I truly do not know yet.
