# Angular — Situational questions

These are “what would you do if…” questions. Each one has:

1. A **proper answer** you can speak
2. A **code example**
3. A **real-world example** from a product (dashboard, login, incidents)

---

## Q1. Component A and Component B have no parent-child relation. How do you connect them?

**Answer:**

If they are not parent and child, `@Input` and `@Output` cannot talk to each other. I put the shared data in a **service** that both inject. The service holds a signal or a `BehaviorSubject`. A writes. B reads. Both components stay dumb. The service is the relation.

Do not use a global variable. Do not reach into the other component with `document.getElementById`.

**Example:**

```typescript
@Injectable({ providedIn: 'root' })
export class SelectedIncidentService {
  private readonly _id = signal<number | null>(null);
  readonly id = this._id.asReadonly();

  select(id: number) {
    this._id.set(id);
  }
}

// Component A — list (no parent of B)
export class IncidentListComponent {
  private selected = inject(SelectedIncidentService);

  onRowClick(id: number) {
    this.selected.select(id);
  }
}

// Component B — header badge (no child of A)
export class HeaderBadgeComponent {
  selectedId = inject(SelectedIncidentService).id;
}
```

```html
<!-- B template -->
@if (selectedId(); as id) {
  <span>Open incident #{{ id }}</span>
}
```

**Real-world example:**

Incident **list** is on the left. A **notification bell** is in the top header. They are not parent-child. When the user clicks a row, the bell should show “1 selected”. The list cannot `@Output` to the header. A root `SelectedIncidentService` is the link.

**Cross-question:** Why not `providedIn` on the list component?

**Cross-answer:**

If the service is provided on A, B gets a **different** instance. They will not share data. Use `providedIn: 'root'` (or a parent route that wraps both) so there is one object.

---

## Q2. A is parent, B is child. How should they talk?

**Answer:**

Down with `@Input`. Up with `@Output`. Parent owns the data. Child only displays or asks to change.

**Example:**

```typescript
// child
@Component({
  selector: 'app-incident-card',
  standalone: true,
  template: `
    <h3>{{ title() }}</h3>
    <button (click)="closed.emit(id())">Close</button>
  `,
})
export class IncidentCardComponent {
  title = input.required<string>();
  id = input.required<number>();
  closed = output<number>();
}

// parent
@Component({
  template: `
    <app-incident-card
      [title]="item.title"
      [id]="item.id"
      (closed)="close($event)" />
  `,
})
export class IncidentPageComponent {
  item = { id: 12, title: 'VPN down' };

  close(id: number) {
    // call API, then update list
  }
}
```

**Real-world example:**

Incident **page** owns the ticket. **Card** only shows title and a Close button. The card does not call HTTP. The page closes it so one place owns the API.

**Cross-question:** Can the child inject the parent class?

**Cross-answer:**

Technically yes. Do not. It ties the card to one parent and makes tests hard. Input/output stays reusable.

---

## Q3. A and B are siblings under the same parent. How do they talk?

**Answer:**

Lift the state to the **parent**. A emits an event. Parent stores it. Parent passes it to B as input. If they later move apart, switch to a service.

**Example:**

```html
<app-filter (changed)="status = $event" />
<app-list [status]="status" />
```

```typescript
export class IncidentsShellComponent {
  status = 'Open';
}
```

**Real-world example:**

Filter chips and the table sit on one **Incidents** page. Filter does not import the table. The shell page holds `status`. Both stay simple.

**Cross-question:** When do siblings need a service instead?

**Cross-answer:**

When they are not always under the same parent — for example filter in a sidebar layout and table in `router-outlet`. Then a service (or query params) is better.

---

## Q4. How do you share data when A and B are on different routes?

**Answer:**

They do not share a parent view. Options:

1. A **root service** (current user, selected tenant)
2. **Router** — id in the URL (`/incidents/12`)
3. **Query params** — filters (`?status=Open`)

If the user should be able to refresh or share the link, put it in the **URL**, not only in memory.

**Example:**

```typescript
// A — list
this.router.navigate(['/incidents', id]);

// B — detail (another route)
incidentId = toSignal(this.route.paramMap.pipe(map(p => Number(p.get('id')))));
```

**Real-world example:**

User clicks incident 12 on the list (`/incidents`). Detail is `/incidents/12`. Refresh still works. A service-only selected id would be lost on refresh.

**Cross-question:** Selected row highlight plus deep link?

**Cross-answer:**

URL holds the id. The list **also** reads the same param to highlight the row. One source of truth: the route.

---

## Q5. Login page sets the user. Header is far away. How does the header get the name?

**Answer:**

An `AuthService` with a signal or `BehaviorSubject`. Login writes the user. Header reads it. Logout sets it to null. On app start, restore from token/cookie once.

**Example:**

```typescript
@Injectable({ providedIn: 'root' })
export class AuthService {
  private readonly _user = signal<User | null>(null);
  readonly user = this._user.asReadonly();

  loginSucceeded(user: User) {
    this._user.set(user);
  }

  logout() {
    this._user.set(null);
  }
}
```

```html
<!-- header, no relation to login component -->
@if (auth.user(); as u) {
  <span>{{ u.name }}</span>
}
```

**Real-world example:**

Login is a full-screen route. The top bar is in `AppComponent`. They never nest. After login you still see “Hi, Mukesh” in the bar because both use `AuthService`.

**Cross-question:** Refresh the browser — name gone?

**Cross-answer:**

Memory is empty. On startup, read the stored token, call `/api/me` (or decode safe claims), then `loginSucceeded`. Header waits until that finishes (guard or initializer).

---

## Q6. After creating an incident on page A, the dashboard counts on page B must update. How?

**Answer:**

Do not hope B is still alive. Options:

1. **Re-fetch** when B becomes active (`OnInit` / `router` events)
2. A **shared store**: A calls API, then `incidentsCreated.next()`, B listens and reloads
3. If both are on screen (shell + dashboard widget), a service signal of counts

**Example:**

```typescript
@Injectable({ providedIn: 'root' })
export class IncidentEvents {
  readonly changed = new Subject<void>();
}

// A after save
this.api.create(dto).subscribe(() => this.events.changed.next());

// B
this.events.changed.pipe(startWith(void 0), switchMap(() => this.api.counts()))
  .subscribe(c => this.counts.set(c));
```

**Real-world example:**

“New incident” is a modal. KPI cards sit on the home dashboard. After save, cards must go from 4 to 5 without a full app reload. The create flow publishes `changed`. Dashboard reloads counts.

**Cross-question:** Why not `window.location.reload()`?

**Cross-answer:**

Slow, loses other UI state, looks amateur. Reload only the number.

---

## Q7. Can you use `ViewChild` to call a method on an unrelated component?

**Answer:**

No. `ViewChild` only sees a child in **this** template. Unrelated A and B are not in each other’s template. Use a service. `ViewChild` is for “I need the child map widget to `resize()` after the side panel opens.”

**Example:**

```typescript
// works — B is in A's template
@ViewChild(MapWidgetComponent) map?: MapWidgetComponent;

onSidebarToggle() {
  this.map?.resize();
}
```

**Real-world example:**

A Leaflet map inside a ticket page. Opening the notes drawer changes width. The page calls `map.resize()`. The **header** cannot `ViewChild` that map. If the header needed to trigger resize, use a service event `layout.changed`.

**Cross-question:** `document.querySelector` from A to B?

**Cross-answer:**

No. Breaks encapsulation, SSR, and tests.

---

## Q8. Should you use an “event bus” (`Subject` in a service) for everything?

**Answer:**

Use it for **rare events** (“incident saved”, “toast”). Do not use it for all state. If B always needs the **current** value (selected id, current user), use a `signal` or `BehaviorSubject`, not a plain `Subject` (late subscribers miss it).

**Example:**

```typescript
// current value — BehaviorSubject / signal
readonly user = signal<User | null>(null);

// fire-and-forget event
readonly saved$ = new Subject<number>();
```

**Real-world example:**

Toast “Saved” is an event. Current user in the header is state. Mixing them in one giant `AppEvents` service becomes spaghetti.

**Cross-question:** NgRx vs service?

**Cross-answer:**

Two components and one selected id: a service is enough. Many screens, undo, and complex events: consider a store. Do not start with NgRx for this.

---

## Q9. A has a reactive form. B has the Submit button in another part of the layout. How?

**Answer:**

The form model lives in a **service** or the parent shell, not inside A only. B calls `service.submit()`. Or wrap both in a parent that holds the `FormGroup`.

**Example:**

```typescript
@Injectable()
export class IncidentFormService {
  readonly form = new FormGroup({
    title: new FormControl('', { nonNullable: true, validators: [Validators.required] }),
  });

  submit() {
    if (this.form.invalid) {
      this.form.markAllAsTouched();
      return;
    }
    return this.api.create(this.form.getRawValue());
  }
}

// provide on the route so A and B share one form
provide: [{ provide: IncidentFormService }]
```

**Real-world example:**

Wizard: step 1 fields in the main pane, **Save** in the sticky footer. Footer is not a child of the form component. Route-level `IncidentFormService` holds the `FormGroup`.

**Cross-question:** Provide the form service in `root`?

**Cross-answer:**

Then the next visit may still have old values. Provide it on the **route** so leaving the wizard destroys it.

---

## Q10. List does not update after you push an item. What happened?

**Answer:**

Usually you **mutated** the array (`list.push`) with `OnPush` or a signal that expects a **new** array. Replace the array. Or you updated a nested object with the same reference.

**Example:**

```typescript
// bad with OnPush / signals
this.items.push(newItem);

// good
this.items = [...this.items, newItem];
this.itemsSignal.update(list => [...list, newItem]);
```

**Real-world example:**

User adds a comment. The thread uses `OnPush`. `comments.push` leaves the same array reference. The new comment does not show until you click elsewhere. Spread a new array and it appears at once.

**Cross-question:** `detectChanges()` to force it?

**Cross-answer:**

It can hide the bug. Fix the data flow.

---

## Q11. HTTP in A runs twice when B also uses the same Observable. Why?

**Answer:**

`HttpClient` Observables are **cold**. Each subscribe = one call. A subscribed in `ngOnInit` and the template also used `async` pipe. Or A and B both called `get()` without sharing.

**Example:**

```typescript
// share one in-flight/cached call
readonly incidents$ = this.http.get<Incident[]>('/api/incidents').pipe(
  shareReplay({ bufferSize: 1, refCount: true })
);
```

Or one service method that caches a signal.

**Real-world example:**

Dashboard cards and the table both need incidents. Two `get('/api/incidents')` in `ngOnInit` = two network rows in DevTools. Put the call in a service with `shareReplay` or load once into a signal.

**Cross-question:** `shareReplay` forever?

**Cross-answer:**

Use `refCount: true` so it unsubscribes when nobody listens. For “always latest”, reload on an explicit refresh instead of an infinite cache.

---

## Q12. User types in a search box. Old results overwrite new ones. How do you fix it?

**Answer:**

Use `switchMap` so the previous HTTP call is cancelled when the text changes. Also `debounceTime` so you do not hit the API every key.

**Example:**

```typescript
this.search.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(q => this.api.search(q)),
  takeUntilDestroyed(),
).subscribe(rows => this.rows.set(rows));
```

**Real-world example:**

User types `inc` then `incident`. Slow `inc` returns last and overwrites `incident`. `switchMap` drops the old call. Search box feels correct.

**Cross-question:** `mergeMap`?

**Cross-answer:**

Keeps all calls. Wrong for typeahead. Save-button double-click: use `exhaustMap`.

---

## Q13. You left the page but `subscribe` still runs and throws. What was wrong?

**Answer:**

You subscribed by hand and did not unsubscribe. When the HTTP returns, it writes to a destroyed component.

**Example:**

```typescript
ngOnInit() {
  this.api.list().pipe(takeUntilDestroyed()).subscribe(x => this.rows = x);
}

// or
rows = toSignal(this.api.list(), { initialValue: [] });
```

**Real-world example:**

User opens Incidents, quickly goes to Settings. The list call returns and Angular errors “view destroyed”. `async` pipe, `takeUntilDestroyed`, or `toSignal` fixes it.

**Cross-question:** `unsubscribe` in `ngOnDestroy`?

**Cross-answer:**

Works. `takeUntilDestroyed()` is shorter and harder to forget.

---

## Q14. Lazy feature has its own service instance. Header does not see updates. Why?

**Answer:**

The service was provided on the **lazy route/component**, not `root`. Header is outside the lazy injector, so it got another instance (or none).

**Example:**

```typescript
// shared with the whole app
@Injectable({ providedIn: 'root' })
export class ToastService { }

// only inside the incidents feature
@Component({
  providers: [IncidentWizardState],
})
export class WizardComponent { }
```

**Real-world example:**

Toasts from “Create incident” (lazy) do not show in the root header because `ToastService` was in the feature `providers` array. Move toast to `root`. Keep wizard state on the wizard.

**Cross-question:** Two “singletons”?

**Cross-answer:**

Yes — two injectors, two objects. That is the usual cause of “I set it but the other screen is empty.”

---

## Q15. How do you pass a large object from A to B without the URL?

**Answer:**

Do not put a big object in query params. Store an **id** in the URL. B loads by id from the API (source of truth). A short-lived service cache is OK for “just created, go to detail” but refresh must still load from API.

**Example:**

```typescript
// A
this.router.navigate(['/incidents', created.id]);

// B
ngOnInit() {
  const id = Number(this.route.snapshot.paramMap.get('id'));
  this.api.get(id).subscribe(x => this.item.set(x));
}
```

**Real-world example:**

Create incident returns id `88`. Navigate to `/incidents/88`. Do not pass the whole incident in router `state` as the only copy — a refresh would be blank.

**Cross-question:** `history.state`?

**Cross-answer:**

Can extra-pass a flash message. Not the record itself.

---

## Q16. Need the same data in a template-driven child and a chart that is not related. Pattern?

**Answer:**

One service, one signal. Child uses inputs if it is presentational. Chart injects the service **or** receives data from a shell. Prefer: API → service signal → both views.

**Example:**

```typescript
@Injectable({ providedIn: 'root' })
export class DashboardStore {
  readonly counts = signal<Counts | null>(null);
  load() {
    this.api.counts().subscribe(c => this.counts.set(c));
  }
}
```

**Real-world example:**

KPI cards and a pie chart on home. Both show “open vs closed”. One `DashboardStore.load()` in the home page. Cards and chart read `counts()`. No second HTTP.

**Cross-question:** Each widget loads itself?

**Cross-answer:**

Then you get 4 calls. Fine if they are different APIs. Same API: load once.

---

## Q17. How do you keep two unrelated components in sync in real time (SignalR)?

**Answer:**

The **hub connection** lives in a root service. Components subscribe to events. When a message arrives, update a signal. All listeners refresh. Do not open one WebSocket per component.

**Example:**

```typescript
@Injectable({ providedIn: 'root' })
export class IncidentHub {
  readonly live = signal<IncidentEvent | null>(null);

  constructor() {
    const conn = new signalR.HubConnectionBuilder().withUrl('/hubs/incidents').build();
    conn.on('updated', (e: IncidentEvent) => this.live.set(e));
    conn.start();
  }
}
```

**Real-world example:**

Agent A closes a ticket. Agent B’s board (another browser) should move the card. One hub service. Board and a small “live” toast both read `live()`.

**Cross-question:** Tenant isolation?

**Cross-answer:**

Server must send only that tenant’s events. The Angular service is not a security boundary.

---

## Q18. Interview wrap: name every way A can talk to B, and when to use it.

**Answer:**

| Relation | Mechanism | When |
| --- | --- | --- |
| Parent → child | `@Input` / `input()` | Data down |
| Child → parent | `@Output` / `output()` | Events up |
| Siblings | Parent state | Same screen |
| Unrelated / header | Root service + signal | User, toasts, selection |
| Different routes | URL params | Refresh + share link |
| Many complex events | Store (NgRx) | Large app |
| Live server push | Hub service | SignalR |

Default for **no relation**: **root service**. Default if it must survive refresh: **URL**.

**Example:**

“Selected incident” on the same layout: service. “Which incident is open”: `/incidents/12`.

**Real-world example:**

Incident product: header user = `AuthService`. List → detail = router id. List → right drawer on the same page = parent or service. That mix is normal.

**Cross-question:** What is the wrong default?

**Cross-answer:**

`@Input` between unrelated components, `window` globals, or NgRx for two screens.


---
## Additional Questions
### Q26. A PrimeNG table becomes slow with thousands of rows. What do you do?
**Answer:** I avoid rendering everything. I use server-side pagination/filtering, stable tracking, and virtual scrolling when the UX needs it. I also check whether the API is returning more data than the screen needs.
### Q27. A component keeps making the same API call. How do you debug it?
**Answer:** I check subscriptions, effects, route events and template async pipes. Then I see the browser Network tab. I make the data flow have one clear owner.
### Q28. The UI updates after one API call but not another. What do you check?
**Answer:** I check whether state was mutated in place, whether the component uses OnPush, and whether a signal was actually updated. I reproduce it with a small state change before changing change detection manually.
