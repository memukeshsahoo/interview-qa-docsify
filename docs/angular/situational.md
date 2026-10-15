# Angular — Situational questions

These are “what would you do if…” questions. Each one has:

1. A **proper answer** you can speak
2. A **code example**
3. A **real-world example** from a product (dashboard, login, incidents)

---

## Q1. Component A and Component B have no parent-child relation. How do you connect them?

**Answer:**

I put the shared data in a **service** that both of them inject. Since they aren't parent and child, `@Input` and `@Output` can't connect them. The service holds a signal or a `BehaviorSubject`. A writes to it, B reads from it, and both components stay dumb. The service is the relation between them.

I don't use a global variable, and I don't reach into the other component with `document.getElementById`.

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

The incident **list** is on the left, and a **notification bell** is in the top header. They aren't parent and child. When the user clicks a row, the bell should show “1 selected”. The list can't `@Output` to the header, so a root `SelectedIncidentService` is the link.

**Cross-question:** Why not `providedIn` on the list component?

**Cross-answer:**

If the service is provided on A, B gets a **different** instance, so they won't share data. I use `providedIn: 'root'` (or a parent route that wraps both) so there's only one object.

---

## Q2. A is parent, B is child. How should they talk?

**Answer:**

Data goes down with `@Input`, and events go up with `@Output`. The parent owns the data. The child only displays it or asks for a change.

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

The incident **page** owns the ticket. The **card** only shows the title and a Close button. The card doesn't call HTTP. The page does the closing, so only one place owns the API call.

**Cross-question:** Can the child inject the parent class?

**Cross-answer:**

Technically yes, but I don't. It ties the card to one parent and makes testing hard. With input/output, the card stays reusable.

---

## Q3. A and B are siblings under the same parent. How do they talk?

**Answer:**

I lift the state up to the **parent**. A emits an event, the parent stores the value, and the parent passes it to B as an input. If they later move apart, I switch to a service.

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

The filter chips and the table sit on one **Incidents** page. The filter doesn't import the table. The shell page holds `status`, and both components stay simple.

**Cross-question:** When do siblings need a service instead?

**Cross-answer:**

When they aren't always under the same parent — for example, the filter is in a sidebar layout and the table is in the `router-outlet`. Then a service (or query params) works better.

---

## Q4. How do you share data when A and B are on different routes?

**Answer:**

They don't share a parent view, so I have a few options:

1. A **root service** (current user, selected tenant)
2. The **router** — the id goes in the URL (`/incidents/12`)
3. **Query params** — for filters (`?status=Open`)

If the user should be able to refresh or share the link, it belongs in the **URL**, not only in memory.

**Example:**

```typescript
// A — list
this.router.navigate(['/incidents', id]);

// B — detail (another route)
incidentId = toSignal(this.route.paramMap.pipe(map(p => Number(p.get('id')))));
```

**Real-world example:**

The user clicks incident 12 on the list (`/incidents`). The detail page is `/incidents/12`. A refresh still works. If the selected id lived only in a service, it would be lost on refresh.

**Cross-question:** Selected row highlight plus deep link?

**Cross-answer:**

The URL holds the id. The list **also** reads the same param to highlight the row. That way there's one source of truth: the route.

---

## Q5. Login page sets the user. Header is far away. How does the header get the name?

**Answer:**

Through an `AuthService` that holds a signal or a `BehaviorSubject`. Login writes the user, and the header reads it. Logout sets it back to null. When the app starts, I restore the user once from the token or cookie.

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

Login is a full-screen route. The top bar lives in `AppComponent`. They never nest. But after login, you still see “Hi, Mukesh” in the bar, because both use `AuthService`.

**Cross-question:** Refresh the browser — name gone?

**Cross-answer:**

After a refresh, memory is empty. So on startup I read the stored token, call `/api/me` (or decode the safe claims), and then call `loginSucceeded`. The header waits until that's done, using a guard or an initializer.

---

## Q6. After creating an incident on page A, the dashboard counts on page B must update. How?

**Answer:**

I don't count on B still being alive. I have a few options:

1. **Re-fetch** when B becomes active (`OnInit` / `router` events)
2. A **shared store**: A calls the API, then `incidentsCreated.next()`, and B listens and reloads
3. If both are on screen (shell + dashboard widget), a service signal that holds the counts

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

“New incident” is a modal, and the KPI cards sit on the home dashboard. After saving, the cards must go from 4 to 5 without reloading the whole app. The create flow publishes `changed`, and the dashboard reloads the counts.

**Cross-question:** Why not `window.location.reload()`?

**Cross-answer:**

It's slow, it loses other UI state, and it looks amateur. I reload only the number.

---

## Q7. Can you use `ViewChild` to call a method on an unrelated component?

**Answer:**

No. `ViewChild` only sees a child in **this** component's template. Unrelated A and B aren't in each other's template, so I use a service. `ViewChild` is for cases like “I need the child map widget to `resize()` after the side panel opens.”

**Example:**

```typescript
// works — B is in A's template
@ViewChild(MapWidgetComponent) map?: MapWidgetComponent;

onSidebarToggle() {
  this.map?.resize();
}
```

**Real-world example:**

There's a Leaflet map inside a ticket page. Opening the notes drawer changes its width, so the page calls `map.resize()`. The **header** can't `ViewChild` that map. If the header needed to trigger a resize, I'd use a service event like `layout.changed`.

**Cross-question:** `document.querySelector` from A to B?

**Cross-answer:**

No. It breaks encapsulation, SSR, and tests.

---

## Q8. Should you use an “event bus” (`Subject` in a service) for everything?

**Answer:**

No. I use it for **rare events**, like “incident saved” or a toast. I don't use it for all state. If B always needs the **current** value (selected id, current user), I use a `signal` or a `BehaviorSubject`, not a plain `Subject`, because late subscribers miss what a plain `Subject` already sent.

**Example:**

```typescript
// current value — BehaviorSubject / signal
readonly user = signal<User | null>(null);

// fire-and-forget event
readonly saved$ = new Subject<number>();
```

**Real-world example:**

A “Saved” toast is an event. The current user in the header is state. If you mix them in one giant `AppEvents` service, it turns into spaghetti.

**Cross-question:** NgRx vs service?

**Cross-answer:**

For two components and one selected id, a service is enough. With many screens, undo, and complex events, I'd consider a store. But I wouldn't start with NgRx for this.

---

## Q9. A has a reactive form. B has the Submit button in another part of the layout. How?

**Answer:**

The form model lives in a **service** or the parent shell, not only inside A. B calls `service.submit()`. Another option is to wrap both in a parent that holds the `FormGroup`.

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

Think of a wizard: the step 1 fields are in the main pane, and **Save** is in the sticky footer. The footer isn't a child of the form component. A route-level `IncidentFormService` holds the `FormGroup`.

**Cross-question:** Provide the form service in `root`?

**Cross-answer:**

Then the next visit might still show the old values. I provide it on the **route**, so leaving the wizard destroys it.

---

## Q10. List does not update after you push an item. What happened?

**Answer:**

Usually the array was **mutated** (`list.push`) while using `OnPush` or a signal that expects a **new** array. The fix is to replace the array. Another cause is updating a nested object but keeping the same reference.

**Example:**

```typescript
// bad with OnPush / signals
this.items.push(newItem);

// good
this.items = [...this.items, newItem];
this.itemsSignal.update(list => [...list, newItem]);
```

**Real-world example:**

A user adds a comment, and the thread uses `OnPush`. `comments.push` keeps the same array reference, so the new comment doesn't show until you click somewhere else. Spread into a new array and it shows up right away.

**Cross-question:** `detectChanges()` to force it?

**Cross-answer:**

That can hide the bug. I fix the data flow instead.

---

## Q11. HTTP in A runs twice when B also uses the same Observable. Why?

**Answer:**

`HttpClient` Observables are **cold**, which means every subscribe makes its own call. Maybe A subscribed in `ngOnInit` and the template also used the `async` pipe. Or A and B both called `get()` without sharing it.

**Example:**

```typescript
// share one in-flight/cached call
readonly incidents$ = this.http.get<Incident[]>('/api/incidents').pipe(
  shareReplay({ bufferSize: 1, refCount: true })
);
```

Or use one service method that caches the result in a signal.

**Real-world example:**

The dashboard cards and the table both need incidents. Two `get('/api/incidents')` calls in `ngOnInit` means two network rows in DevTools. I put the call in a service with `shareReplay`, or load it once into a signal.

**Cross-question:** `shareReplay` forever?

**Cross-answer:**

I use `refCount: true` so it unsubscribes when nobody is listening. If I need the latest data, I reload on an explicit refresh instead of keeping an endless cache.

---

## Q12. User types in a search box. Old results overwrite new ones. How do you fix it?

**Answer:**

I use `switchMap`, so the previous HTTP call gets cancelled when the text changes. I also add `debounceTime` so we don't hit the API on every key press.

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

The user types `inc`, then `incident`. The slow `inc` call comes back last and overwrites the `incident` results. `switchMap` drops the old call, so the search box behaves correctly.

**Cross-question:** `mergeMap`?

**Cross-answer:**

It keeps all the calls, so it's wrong for typeahead. For a double-click on a Save button, I'd use `exhaustMap`.

---

## Q13. You left the page but `subscribe` still runs and throws. What was wrong?

**Answer:**

The subscription was made by hand and never unsubscribed. When the HTTP call returns, it writes to a component that's already destroyed.

**Example:**

```typescript
ngOnInit() {
  this.api.list().pipe(takeUntilDestroyed()).subscribe(x => this.rows = x);
}

// or
rows = toSignal(this.api.list(), { initialValue: [] });
```

**Real-world example:**

The user opens Incidents and quickly goes to Settings. The list call comes back and Angular throws a “view destroyed” error. The `async` pipe, `takeUntilDestroyed`, or `toSignal` fixes it.

**Cross-question:** `unsubscribe` in `ngOnDestroy`?

**Cross-answer:**

That works. `takeUntilDestroyed()` is just shorter and harder to forget.

---

## Q14. Lazy feature has its own service instance. Header does not see updates. Why?

**Answer:**

The service was provided on the **lazy route or component**, not in `root`. The header sits outside the lazy injector, so it got a different instance (or none at all).

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

Toasts from “Create incident” (a lazy feature) don't show in the root header, because `ToastService` was in the feature's `providers` array. I move the toast service to `root` and keep the wizard state on the wizard.

**Cross-question:** Two “singletons”?

**Cross-answer:**

Yes — two injectors means two objects. That's the usual cause of “I set it, but the other screen is empty.”

---

## Q15. How do you pass a large object from A to B without the URL?

**Answer:**

I don't put a big object in query params. I put an **id** in the URL, and B loads the data by that id from the API, which is the source of truth. A short-lived service cache is fine for the “just created, go to detail” case, but a refresh still has to load from the API.

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

Creating an incident returns id `88`. I navigate to `/incidents/88`. I don't pass the whole incident in the router `state` as the only copy, because a refresh would show a blank page.

**Cross-question:** `history.state`?

**Cross-answer:**

It's fine for passing something extra, like a flash message. Not for the record itself.

---

## Q16. Need the same data in a template-driven child and a chart that is not related. Pattern?

**Answer:**

One service, one signal. The child uses inputs if it's presentational. The chart either injects the service **or** gets the data from a shell. What I prefer: API → service signal → both views.

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

The home page has KPI cards and a pie chart, and both show “open vs closed”. The home page calls `DashboardStore.load()` once. The cards and the chart both read `counts()`. No second HTTP call.

**Cross-question:** Each widget loads itself?

**Cross-answer:**

Then you get 4 calls. That's fine if they're different APIs. If it's the same API, load it once.

---

## Q17. How do you keep two unrelated components in sync in real time (SignalR)?

**Answer:**

The **hub connection** lives in a root service, and components subscribe to its events. When a message arrives, the service updates a signal and all the listeners refresh. I don't open one WebSocket per component.

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

Agent A closes a ticket. Agent B's board, in another browser, should move the card. There's one hub service, and both the board and a small “live” toast read `live()`.

**Cross-question:** Tenant isolation?

**Cross-answer:**

The server must send only that tenant's events. The Angular service is not a security boundary.

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

When there's **no relation**, my default is a **root service**. If it has to survive a refresh, my default is the **URL**.

**Example:**

“Selected incident” on the same layout goes in a service. “Which incident is open” goes in the URL: `/incidents/12`.

**Real-world example:**

In the incident product, the header user comes from `AuthService`. List → detail uses the router id. List → right drawer on the same page uses the parent or a service. Mixing these is normal.

**Cross-question:** What is the wrong default?

**Cross-answer:**

Trying `@Input` between unrelated components, using `window` globals, or bringing in NgRx for two screens.


---
## Additional Questions
### Q26. A PrimeNG table becomes slow with thousands of rows. What do you do?
**Answer:** I avoid rendering everything. I use server-side pagination and filtering, stable tracking, and virtual scrolling when the UX needs it. I also check whether the API is sending more data than the screen needs.
### Q27. A component keeps making the same API call. How do you debug it?
**Answer:** I check subscriptions, effects, route events, and async pipes in the template. Then I look at the browser's Network tab. I make sure the data flow has one clear owner.
### Q28. The UI updates after one API call but not another. What do you check?
**Answer:** I check whether state was mutated in place, whether the component uses OnPush, and whether a signal was actually updated. I reproduce it with a small state change before I start changing change detection by hand.
