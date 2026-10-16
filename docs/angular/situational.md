# Angular — Situational questions

These are “what would you do if…” questions. Each one has:

1. A **proper answer** you can speak
2. A **code example**
3. A **real-world example** from a product (dashboard, login, incidents)

---

## Q1. Component A and Component B have no parent-child relation. How do you connect them?

**Answer:**

I put the shared data in a **service** that both of them inject. Since they aren't parent and child, `@Input` and `@Output` can't connect them. The service holds a signal or a `BehaviorSubject`. A writes to it, B reads from it, and both components stay dumb. The service is the relation between them.

In plain words, a service is a small shared object that Angular creates and hands to any component that asks for it ("injects" it). A signal or a `BehaviorSubject` is a box that holds the current value and tells everyone who is watching when that value changes. So A doesn't need to know B exists. A just puts a value in the box, and B, which is watching the box, updates itself. It's like a shared whiteboard in an office: one person writes on it, and everyone who looks at it sees the latest note, without the two people ever talking directly.

I don't use a global variable, and I don't reach into the other component with `document.getElementById`. A global variable doesn't tell anyone when it changes, so B would show old data. Reaching into the page by hand skips Angular completely, breaks easily when the HTML changes, and makes the code very hard to test.

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

Think of it like two people each getting their own notebook instead of sharing one. A writes in its notebook, B reads its own, and B's notebook stays empty. Providing the service at the root means the whole app shares one notebook.

---

## Q2. A is parent, B is child. How should they talk?

**Answer:**

Data goes down with `@Input`, and events go up with `@Output`. The parent owns the data. The child only displays it or asks for a change.

An input is a value the parent hands to the child, like passing a note down. An output is a signal the child sends back up, like raising a hand to say "the user clicked Close". The child doesn't change the data itself. It just tells the parent what happened, and the parent decides what to do.

This matters because when only one place owns and changes the data, it's easy to find bugs and easy to know where the truth lives. If the child changed the data on its own, you'd have two places editing the same thing, and they could get out of step. It also keeps the child reusable: the same card can sit on any page, because it doesn't care where the data comes from.

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

If the card injects `IncidentPageComponent`, it only works inside that page. Put it on another page and it breaks. In a test, I'd also have to build the whole parent just to test a small card. With inputs and outputs, I just pass in a title and an id and check what it emits.

---

## Q3. A and B are siblings under the same parent. How do they talk?

**Answer:**

I lift the state up to the **parent**. A emits an event, the parent stores the value, and the parent passes it to B as an input. If they later move apart, I switch to a service.

"Lifting state up" means the shared value lives in the nearest common parent instead of inside one of the siblings. Siblings can't talk to each other directly, but they both talk to the parent. It's like two kids at a table passing a message through their mom: one tells her, and she tells the other.

The benefit is that A and B don't know about each other at all. The filter only says "the user picked Open", and the list only says "show me the rows for this status". The parent wires them together. That keeps both components simple and easy to reuse or test on their own.

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

The `router-outlet` is the slot where Angular swaps pages in and out as you navigate. Whatever sits in it isn't a normal child that the layout can pass inputs to, so the "go through the parent" trick no longer works. A shared service holds the value for both. Query params (like `?status=Open` in the URL) are a good choice when the filter should also survive a refresh or be shared as a link.

---

## Q4. How do you share data when A and B are on different routes?

**Answer:**

They don't share a parent view, so I have a few options:

1. A **root service** (current user, selected tenant). This is one shared object for the whole app that stays alive while the user moves between pages.
2. The **router** — the id goes in the URL (`/incidents/12`). The next page reads the id from the address bar.
3. **Query params** — for filters (`?status=Open`). These are the extra bits after the `?` in a URL, good for optional settings like filters and sorting.

If the user should be able to refresh or share the link, it belongs in the **URL**, not only in memory.

The reason is simple: anything kept only in memory is wiped when the page reloads. The URL is the one thing the browser keeps. It's like writing the page number on a bookmark instead of just remembering it: close the book, open it again, and you can still find your place. It also means a colleague can open the exact same screen from a link you send them.

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

A deep link is a link that opens a specific item directly, like `/incidents/12`, instead of the home page. If the highlight came from a separate variable, it could say "row 10" while the URL says "12". Reading both from the URL means they can never disagree.

---

## Q5. Login page sets the user. Header is far away. How does the header get the name?

**Answer:**

Through an `AuthService` that holds a signal or a `BehaviorSubject`. Login writes the user, and the header reads it. Logout sets it back to null. When the app starts, I restore the user once from the token or cookie.

The login page and the header have nothing to do with each other in the page structure, so they can't pass data directly. `AuthService` is one shared object for the whole app that remembers who is logged in. The login page puts the user in, and the header simply watches it. Because a signal tells its readers when the value changes, the header updates the moment the user logs in or out, with no extra code in the header.

This also keeps "who is logged in" in exactly one place. Any other part of the app, like a menu that hides admin links, can read the same service and always agree with the header.

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

A token is the proof of login the browser keeps after you sign in, so it survives a refresh even though the signal doesn't. On startup I use it to ask the server "who am I?" and fill the service again. A guard (code that runs before a route opens) or an app initializer (code that runs before the app starts) makes sure the screen doesn't show "logged out" for a moment while that check is still running.

---

## Q6. After creating an incident on page A, the dashboard counts on page B must update. How?

**Answer:**

I don't count on B still being alive. I have a few options:

1. **Re-fetch** when B becomes active (`OnInit` / `router` events). When the user comes back to the dashboard, it simply loads fresh numbers.
2. A **shared store**: A calls the API, then `incidentsCreated.next()`, and B listens and reloads. `next()` is how you send a new message on a `Subject`, so it works like A ringing a bell that B is listening for.
3. If both are on screen (shell + dashboard widget), a service signal that holds the counts

When the user moves away from a page, Angular usually destroys it, so page B may not even exist while A is saving. That's why I don't try to "reach into" B. Either B loads fresh data when it opens, or, if it's on screen at the same time, it listens for a "something changed" message and reloads itself.

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

A full reload downloads and starts the whole app again, so the user waits, the screen flashes, and anything they had open, like a filter, a scroll position or a half-typed form, is lost. Asking the API for just the new counts is fast and nothing else on the screen moves.

---

## Q7. Can you use `ViewChild` to call a method on an unrelated component?

**Answer:**

No. `ViewChild` only sees a child in **this** component's template. Unrelated A and B aren't in each other's template, so I use a service. `ViewChild` is for cases like “I need the child map widget to `resize()` after the side panel opens.”

`ViewChild` is a way for a component to grab a direct handle on something placed inside its own HTML. It's like being able to reach things on your own desk, but not things on a colleague's desk in another room. If B isn't written inside A's template, A simply has no way to find it with `ViewChild`. For components that aren't related, a shared service is the right tool.

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

`document.querySelector` searches the raw page by hand. Encapsulation means each component keeps its insides private, and this ignores that, so a small change to B's HTML silently breaks A. SSR (server-side rendering, where Angular builds the page on the server) has no real browser `document` to search. And in unit tests the other component often isn't on the page at all, so the search finds nothing.

---

## Q8. Should you use an “event bus” (`Subject` in a service) for everything?

**Answer:**

No. I use it for **rare events**, like “incident saved” or a toast. I don't use it for all state. If B always needs the **current** value (selected id, current user), I use a `signal` or a `BehaviorSubject`, not a plain `Subject`, because late subscribers miss what a plain `Subject` already sent.

An event bus is a shared channel where any part of the app can shout a message and anyone listening hears it. A plain `Subject` is like a radio announcement: if you tune in after it was said, you missed it. A `BehaviorSubject` or a signal is more like a notice board: it always shows the latest value, so someone who walks up later still sees it.

That's the key difference between an **event** ("something just happened") and **state** ("this is how things are right now"). A toast saying "Saved" is an event, and it's fine if a component that loads later never sees it. The current user is state, and a header that loads a second later must still know who it is.

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

NgRx is a library that keeps all app state in one central store with strict rules for how it changes. That structure helps in a big app with lots of screens touching the same data, but it adds a lot of extra files and setup. For a small case like this, a plain service is easier to read and does the job.

---

## Q9. A has a reactive form. B has the Submit button in another part of the layout. How?

**Answer:**

The form model lives in a **service** or the parent shell, not only inside A. B calls `service.submit()`. Another option is to wrap both in a parent that holds the `FormGroup`.

A reactive form in Angular is a form whose fields and rules live in TypeScript as a `FormGroup` object, not just in the HTML. Because it's just an object, it doesn't have to live inside the component that shows the fields. If I put it in a shared service, A can show the inputs and B can press Submit, and both work on the same form object.

If the form lived only inside A, B would have no way to read the values or check if the form is valid. Moving the form one level up, into a service or the parent, gives both of them access to the same thing.

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

A root service lives as long as the app does, so the form would keep whatever the user typed last time. That's like a paper form that still has the previous person's answers on it. Providing the service on the route means Angular creates a fresh one when the user enters the wizard and throws it away when they leave.

---

## Q10. List does not update after you push an item. What happened?

**Answer:**

Usually the array was **mutated** (`list.push`) while using `OnPush` or a signal that expects a **new** array. The fix is to replace the array. Another cause is updating a nested object but keeping the same reference.

"Mutated" means the same array was changed in place. `OnPush` is a setting that tells Angular "only re-check this component when its inputs get a new value". Angular checks this by comparing references, which is like asking "is this the same box?" rather than "has anything inside the box changed?". When you `push` into the array, it's still the same box, so Angular thinks nothing changed and skips the update. Signals work the same way: setting the same array again doesn't count as a change.

Creating a new array with the spread syntax (`[...this.items, newItem]`) gives Angular a brand-new box, so it sees the change and redraws the list. The same idea applies to objects inside the array: if you edit a field on an object but keep the same object, Angular may not notice.

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

`detectChanges()` tells Angular to re-check the screen right now, by hand. It may make this one list show up, but the real problem, changing data in place, is still there and will cause the same bug in the next place that reads that data. Replacing the array fixes it once, everywhere.

---

## Q11. HTTP in A runs twice when B also uses the same Observable. Why?

**Answer:**

`HttpClient` Observables are **cold**, which means every subscribe makes its own call. Maybe A subscribed in `ngOnInit` and the template also used the `async` pipe. Or A and B both called `get()` without sharing it.

An Observable is a stream of values you can listen to; subscribing is how you start listening. A "cold" one doesn't do anything until someone subscribes, and then it does all its work again for each new subscriber. It's like a recipe, not a cooked meal: every person who asks for it cooks it from scratch. So two subscribers means two network calls for the same data.

That wastes bandwidth, puts extra load on the server, and can even show two slightly different results if the data changed between calls. The fix is to share one call between everyone who needs it.

**Example:**

```typescript
// share one in-flight/cached call
readonly incidents$ = this.http.get<Incident[]>('/api/incidents').pipe(
  shareReplay({ bufferSize: 1, refCount: true })
);
```

Or use one service method that caches the result in a signal.

`shareReplay` makes all subscribers share a single call and remembers the last result (`bufferSize: 1`), so someone who subscribes a bit later gets the same answer without a new request.

**Real-world example:**

The dashboard cards and the table both need incidents. Two `get('/api/incidents')` calls in `ngOnInit` means two network rows in DevTools. I put the call in a service with `shareReplay`, or load it once into a signal.

**Cross-question:** `shareReplay` forever?

**Cross-answer:**

I use `refCount: true` so it unsubscribes when nobody is listening. If I need the latest data, I reload on an explicit refresh instead of keeping an endless cache.

`refCount: true` keeps a count of who is still listening. When that count drops to zero, the shared subscription is closed, so it doesn't stay open in the background for no reason. And because a cache can go out of date, I give the user a clear refresh (or reload after a save) rather than serving the same old answer forever.

---

## Q12. User types in a search box. Old results overwrite new ones. How do you fix it?

**Answer:**

I use `switchMap`, so the previous HTTP call gets cancelled when the text changes. I also add `debounceTime` so we don't hit the API on every key press.

The problem happens because network calls don't always come back in the order they were sent. A slow, older request can finish after a newer one and replace the right results with stale ones. `switchMap` fixes this by only caring about the latest search: when a new value arrives, it drops the old request. It's like changing your order at a café: the barista throws away the first order and makes only the new one.

`debounceTime(300)` waits until the user has stopped typing for 300 milliseconds before searching. Without it, typing "incident" would send eight requests, one per letter. `distinctUntilChanged()` also skips the search if the text is the same as last time.

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

`mergeMap` lets every request run and passes along every result as it arrives, so the old-results-win problem comes back. `exhaustMap` does the opposite of `switchMap`: while one call is still running, it ignores new clicks. That's exactly what you want for Save, so a double-click doesn't create the same record twice.

---

## Q13. You left the page but `subscribe` still runs and throws. What was wrong?

**Answer:**

The subscription was made by hand and never unsubscribed. When the HTTP call returns, it writes to a component that's already destroyed.

When you call `subscribe`, you start listening, and that listening doesn't stop by itself just because the user left the page. It's like leaving a phone call on after you've walked out of the room: the other side keeps talking, but no one's there. When the answer finally arrives, the code tries to update a screen that no longer exists, which can cause errors and also leaks memory.

The fix is to tie the subscription to the component's life, so it stops automatically when the component is destroyed. `takeUntilDestroyed()` does exactly that, and `toSignal` handles subscribing and cleaning up for you.

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

With the manual way, I have to store each subscription in a field and remember to clean it up in `ngOnDestroy`. If someone adds a new subscription later and forgets that step, the leak comes back. `takeUntilDestroyed()` sits right next to the subscription, so the cleanup is written in the same place.

---

## Q14. Lazy feature has its own service instance. Header does not see updates. Why?

**Answer:**

The service was provided on the **lazy route or component**, not in `root`. The header sits outside the lazy injector, so it got a different instance (or none at all).

A lazy feature is a part of the app that's only downloaded when the user first goes there, which keeps the first load fast. An injector is what Angular uses to create services and hand them out. Each lazy route or component that lists `providers` gets its own injector, and services registered there belong to that area only. So the feature writes to its own copy of the service, while the header, which lives outside that area, reads a different copy and never sees the update.

The rule I follow: anything the whole app must share (toasts, current user) goes in `root`. Anything that should only exist inside one feature (like wizard state) goes in that feature's `providers`.

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

A singleton is supposed to be "only one of these in the app", but that's only true per injector. If the same service is provided in two places, you get two separate objects that just happen to have the same class, like two branches of a bank that don't share account records.

---

## Q15. How do you pass a large object from A to B without the URL?

**Answer:**

I don't put a big object in query params. I put an **id** in the URL, and B loads the data by that id from the API, which is the source of truth. A short-lived service cache is fine for the “just created, go to detail” case, but a refresh still has to load from the API.

A big object in the URL makes the link long and ugly, it can hit browser length limits, and anyone can see or edit it in the address bar. An id is short and safe to share. B then asks the server for the full record, so it always shows the real, current data rather than a copy A had a moment ago.

Think of it like a library: you don't carry the whole book around to tell someone which one you mean, you give them the shelf number and they fetch it.

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

`history.state` is a hidden bit of data you can attach to a navigation. It isn't part of the URL, so a shared link won't carry it, and you can't count on it being there when the page is opened fresh. That's fine for a nice-to-have like "Incident created" shown once, but the actual incident has to come from the API.

---

## Q16. Need the same data in a template-driven child and a chart that is not related. Pattern?

**Answer:**

One service, one signal. The child uses inputs if it's presentational. The chart either injects the service **or** gets the data from a shell. What I prefer: API → service signal → both views.

"Presentational" means the component only shows what it's given and doesn't fetch anything itself, so inputs are the natural way to feed it. The chart isn't related to the child, so it reads the same data from the shared service (or from the shell page that holds both).

The point is that the data is loaded once and kept in one place. Both views read from that same signal, so they always show the same numbers and there's only one network call. If each one loaded its own copy, you'd pay for extra calls, and the cards and the chart could briefly disagree.

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

If each widget needs different data, letting each one load its own is fine and keeps them independent. But four widgets asking the same endpoint for the same numbers is just waste, and the server does the same work four times.

---

## Q17. How do you keep two unrelated components in sync in real time (SignalR)?

**Answer:**

The **hub connection** lives in a root service, and components subscribe to its events. When a message arrives, the service updates a signal and all the listeners refresh. I don't open one WebSocket per component.

SignalR is a library that keeps an open connection between the browser and the server, so the server can push updates the moment something happens, instead of the browser asking again and again. The "hub" is the server endpoint that connection talks to, and a WebSocket is the kind of always-open connection it usually uses.

Keeping one connection in a root service is like having one radio in the office that everyone listens to, instead of each person carrying their own. If every component opened its own connection, the server would hold many connections per user, and each component might get messages at slightly different times. With one shared connection, every part of the screen reads the same signal and stays in sync.

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

A tenant is one customer organisation in an app that serves many. Anything in the browser can be read or changed by the user, so filtering events in Angular doesn't protect anything. If the server sends company X's events to company Y's browser, the data has already leaked. The server has to decide who receives what, for example by sending each event only to the users of that tenant.

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

The way I pick is by asking two questions. First, how are the two components related on the screen? The closer they are, the simpler the tool: inputs and outputs for parent and child, the parent for siblings, a shared service for anything further apart. Second, does the value need to survive a refresh or be shareable as a link? If yes, it goes in the URL. A store like NgRx and a SignalR hub are for bigger needs: lots of shared state across many screens, or updates pushed from the server.

**Example:**

“Selected incident” on the same layout goes in a service. “Which incident is open” goes in the URL: `/incidents/12`.

**Real-world example:**

In the incident product, the header user comes from `AuthService`. List → detail uses the router id. List → right drawer on the same page uses the parent or a service. Mixing these is normal.

**Cross-question:** What is the wrong default?

**Cross-answer:**

Trying `@Input` between unrelated components, using `window` globals, or bringing in NgRx for two screens.

`@Input` only works from a parent to its direct child, so it simply can't connect unrelated components. `window` globals don't tell anyone when they change and anything can overwrite them. NgRx for two screens adds a lot of setup for a problem a small service already solves.


---
## Additional Questions
### Q26. A PrimeNG table becomes slow with thousands of rows. What do you do?
**Answer:** I avoid rendering everything. I use server-side pagination and filtering, stable tracking, and virtual scrolling when the UX needs it. I also check whether the API is sending more data than the screen needs.

The browser gets slow because every row is a lot of HTML it has to build and keep updated, and nobody can read thousands of rows at once anyway. Server-side pagination means the server sends only one page, say 50 rows, and filtering happens on the server too, so the browser never holds the full list. Stable tracking (giving each row a unique id that Angular can track it by) lets Angular reuse rows it has already drawn instead of rebuilding the whole table when data changes. Virtual scrolling draws only the rows you can see on screen and swaps them as you scroll, like a window that only shows a slice of a very long list.

Checking the API matters too. If each row comes with fields the table never shows, the response is bigger and slower than it needs to be.
### Q27. A component keeps making the same API call. How do you debug it?
**Answer:** I check subscriptions, effects, route events, and async pipes in the template. Then I look at the browser's Network tab. I make sure the data flow has one clear owner.

These are the usual places a repeat call hides: a stream subscribed twice (for example in code and again with the `async` pipe), an effect that re-runs every time a signal it reads changes, or code that runs on every router event instead of once. The Network tab in the browser's developer tools shows each request as it happens, so I can see how many times it fires and when, which points me to the trigger.

"One clear owner" means one place, usually a service, is responsible for loading that data, and everyone else just reads the result. That way there's only one thing that can trigger the call, and it's easy to find.
### Q28. The UI updates after one API call but not another. What do you check?
**Answer:** I check whether state was mutated in place, whether the component uses OnPush, and whether a signal was actually updated. I reproduce it with a small state change before I start changing change detection by hand.

Most of the time the working call creates new data, while the broken one edits existing data in place, for example pushing into an array or changing a field on the same object. With `OnPush`, Angular only re-checks when it gets a new reference, so in-place edits go unnoticed. The same applies when a signal is set to the same array or object it already held, or when the code updated a plain variable and forgot to update the signal at all.

Reproducing it with a small, controlled change helps me find the real cause. If I jump straight to forcing change detection by hand, the screen may update, but the actual mistake in how the data is changed is still there.
