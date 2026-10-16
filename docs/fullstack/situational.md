# Full-stack situational questions

Angular + ASP.NET Core together. Each one is a story you can tell in an interview.

---

## Q1. List is empty in Angular. API returns 200. Walk through debug.

**Answer:**

I open the Network tab and check the URL, the status, and the JSON.

- Body has rows but the UI is empty → it's the mapping, an `*ngIf`, or wrong property names (`title` vs `Title`).
- Body is `[]` → check the tenant, the filter, and the user. Debug the API next.
- CORS error → that's not an empty list; the call never succeeded.

The Network tab shows exactly what the server sent back, so it splits the problem in two straight away. If the data is there, the bug is in Angular, somewhere between the response and the screen. If the data isn't there, the bug is on the server, and changing the component won't help. A 200 status only means "the request worked"; it doesn't promise the body has what I expected. It's like checking whether a parcel arrived empty or whether you just haven't opened it yet.

**Example:**

The C# DTO has `Title`, but ASP.NET Core sends JSON in camelCase by default, so the API returns `title`. The Angular interface has to use `title` too. If it expects `Title`, the field comes back `undefined` and the list looks empty.

```csharp
public sealed class IncidentListDto
{
    public int Id { get; init; }
    public required string Title { get; init; }
}
```

```typescript
interface IncidentListDto { id: number; title: string; }
```

**Real-world example:**

Our incident dashboard cards showed 0. The Network tab showed `[]`. The tenant claim was missing after a refresh, and the API failed closed by returning an empty list instead of a 401. We changed it to return 401 when the tenant was missing, so the UI could log the user in again.

**Cross-question:** First tool?

**Cross-answer:**

The Network tab, not a rewrite of the component.

It takes seconds and tells me which side of the app has the bug. Rewriting code before I know where the problem is usually just moves the problem around.

---

## Q2. How do you establish a relation between two unrelated Angular components **and** keep the API honest?

**Answer:**

On the UI side, I connect them with a shared service or the URL. On the API side, I still authorize every call, because a service is not security. If A “selects” incident 12, B should show it only after GET `/api/incidents/12` succeeds for **this** user.

A shared service is one object both components can reach, so A can put a value in and B can read it. The URL works too, for example `/incidents/12`, and it has the bonus that a refresh or a shared link still works. But everything in the browser can be changed by the user with DevTools. So the browser can say "I want incident 12", and the server must decide whether this user is allowed to see it. It's like a receptionist who still checks your ID, even if a friend told them you're expected.

**Example:**

A writes `selectedId` to a service. B calls the API with that id. If it gets a 404, B shows an empty state, not cached data from A.

**Real-world example:**

A malicious user could type another tenant's id into a service (or the query string). B must not show a DTO that A stuffed into memory. B loads it from the API.

**Cross-question:** Pass the whole object from A to B in the service to skip HTTP?

**Cross-answer:**

That's OK as a cache for the same user and the same tenant. But a refresh or a deep link still needs a GET by id.

A service lives only in memory, so it's empty after a page refresh or when someone opens a link to that page directly. B has to be able to load the data on its own in those cases. Passing the object is just a speed-up for the normal click-through path, not a replacement for the API call.

---

## Q3. Login from Angular button to database. Where can it leak?

**Answer:**

It can leak through the password in a query string, in logs, or in error payloads, or a JWT in the URL. Also if HTTPS is off, if the token goes into `console.log`, or if the tenant is taken from the body.

A login sends secrets through many places: the browser, the network, maybe a gateway, the API, the logs and the database. Each one is a chance for them to be copied somewhere they shouldn't be. URLs get saved in browser history and server logs, so secrets never go there. Without HTTPS, anyone on the same network can read the traffic. Logs and error messages are often read by many people, so they must never contain passwords or tokens.

A JWT (JSON Web Token) is the signed token the server gives back after login; whoever holds it can act as that user until it expires, so it needs the same care as a password. And the tenant (which organisation the user belongs to) must come from the server, not from the request body, or a user could just type in someone else's.

**Example:**

```http
POST /api/auth/login
{ "email": "a@b.com", "password": "..." }
```

The server hashes and compares the password, then returns a token in JSON or in an httpOnly cookie. Angular stores it the way we agreed. Later calls send an `Authorization: Bearer` header, and the API uses the claims.

**Real-world example:**

A gateway logged the full request body, and passwords ended up in Seq. We redacted `password` and stopped logging `Authorization`.

**Cross-question:** JWT in localStorage?

**Cross-answer:**

It's common, but XSS can steal it. You should know the trade-off against httpOnly cookies, which then need CSRF protection.

`localStorage` can be read by any script running on the page. If an attacker manages to inject a script (that's XSS), it can read the token and send it away. An httpOnly cookie can't be read by scripts at all, which fixes that. But the browser sends cookies on its own with every request to that site, so another site could trick the browser into making a request with the user's cookie (that's CSRF). So with cookies I also need CSRF protection, like SameSite cookies or an anti-forgery token.

---

## Q4. Create incident in Angular, then dashboard counts. Both must be tenant-safe.

**Answer:**

The POST has `[Authorize]`. The server sets the tenant from the request context, not from the JSON. The counts query uses the same tenant. The Angular service can fire a `changed` event, and the dashboard refetches its counts. Cache keys include the tenant.

There are two jobs here. The first is safety: `[Authorize]` makes sure only logged-in users can create, and the server fills in the tenant from the user's verified login, so nobody can create a record inside another company's data. The counts query filters by that same tenant, so each company only counts its own incidents.

The second job is keeping the screen up to date. The create form and the dashboard are separate components, so the dashboard doesn't know a new incident exists. The service sends a `changed` signal after a successful create, and the dashboard reloads its numbers. Adding the tenant to cache keys means company A's cached counts can never be served to company B, like labelling each shelf with the owner's name.

**Example:**

```csharp
incident.TenantKey = _user.TenantKey; // ignore dto.TenantKey
```

**Real-world example:**

After a create, the KPI still showed “open = 4” instead of 5 until a refresh, because the dashboard never subscribed to `changed`. We emitted the event after the POST returned 201 and reloaded the counts. Putting the tenant key on the cache also stopped a later Redis mix-up.

**Cross-question:** Count in Angular from the list in memory?

**Cross-answer:**

That's wrong if the list is paged. I count on the server.

With paging, the browser only has one page of rows, say 20, even when there are 500 in the database. Counting those would give 20. The server can see all the rows, so it's the only place that can give the true total.

---

## Q5. File upload from Angular to .NET. Situational checks?

**Answer:**

`FormData` with `HttpClient`. The API checks size and extension, saves under a generated name, and checks permission on the parent incident. Private files don't go in git or in the public `wwwroot`.

`FormData` is the browser's way of packing a file into a request, the same format a normal HTML upload form uses. The rest is about not trusting the file. A size limit stops someone filling the disk. An extension check blocks types I don't want, like scripts. A generated name means the user's file name is never used as a path, so a name like `../../something` can't escape the upload folder or overwrite a real file. The permission check makes sure the user can actually edit the incident they're attaching to.

`wwwroot` is the folder ASP.NET Core serves to anyone who knows the URL. Private attachments go somewhere else, and the API hands them out only after checking who's asking.

**Example:**

```typescript
const fd = new FormData();
fd.append('file', file);
this.http.post(`/api/incidents/${id}/files`, fd);
```

**Real-world example:**

Users attached photos to a ticket. The first version used the original file name, and one user uploaded `../../web.config`. Now we save the file as `guid + ext` in a folder outside the web root and serve downloads through an authorized action.

**Cross-question:** Progress bar?

**Cross-answer:**

`HttpClient` with `reportProgress` and `observe: 'events'`. The security checks stay the same.

With those two options, `HttpClient` gives me a stream of events instead of just the final result. The upload progress events include how many bytes were sent so far and the total, so I can work out a percentage for the bar. It's only a display feature; the server checks don't change at all.

---

## Q6. Slow page: is it Angular or the API?

**Answer:**

If TTFB (time to first byte) is high in the Network tab, it's the API or SQL. A huge download means the payload or the bundle. High CPU in the Performance panel means list rendering (`trackBy`, virtual scroll). Guessing “Angular is slow” without looking at the waterfall wastes time.

The waterfall is the timeline in the Network tab that shows each request as a bar. TTFB is how long the browser waited before the server started replying, so a long wait there means the server is busy thinking, usually in the database. A long download part means the reply is simply too big. If the network is quick but the page still stutters, the browser itself is busy drawing too much on screen.

`trackBy` (or `track` in `@for`) tells Angular how to recognise each row, so it can reuse rows instead of rebuilding them all. Virtual scroll only draws the rows you can actually see. It's like a doctor checking which part hurts before giving medicine, instead of guessing.

**Example:**

Waterfall: TTFB 3s, download 20ms → fix SQL/projection. TTFB 40ms, download 8MB → DTO too fat. TTFB 40ms, download 30kb, FPS drops → `@for` track id / virtual scroll.

**Real-world example:**

Our incident board felt laggy, but the API took 50ms. The page had 3,000 DOM rows without `track`. Virtual scroll plus `track id` made it smooth. We almost added Redis for no reason.

**Cross-question:** First fix in a 30-minute interview task?

**Cross-answer:**

Measure first, then fix the biggest bar in the waterfall.

With little time, I want the fix that saves the most. The longest bar is where most of the time goes, so shaving it down helps more than polishing something that was already fast.

---

## Q7. 401 after 20 minutes while the user is typing a form.

**Answer:**

The access token expired. The interceptor should refresh **once** and retry the save, or, if you cached the form, redirect to login without losing it. Don't fire two refreshes.

An access token is a short-lived pass the API checks on each call. Keeping it short means a stolen one stops working soon. A refresh token is a longer-lived pass used only to get a new access token. An interceptor is Angular code that sits in the middle of every HTTP call, so it's the natural place to spot a 401, get a new token, and quietly repeat the request. The user shouldn't even notice.

"Once" matters because a page often makes several calls at the same time. If each one starts its own refresh, they race each other, and with token rotation the later ones can fail and log the user out. So all the waiting calls share one refresh. The worst outcome is losing a long form the user typed, so that has to be protected too.

**Example:**

Queue the 401s on one `refresh$`, then retry the original POST. If the refresh fails, store the form in `sessionStorage` (no passwords) and go to login.

**Real-world example:**

We had a long incident form and a 15-minute token TTL. Save returned 401 and wiped the form. We added refresh and a message: “session expired, log in, we kept your draft.”

**Cross-question:** Put the access token in the URL to avoid expiry?

**Cross-answer:**

No. URLs leak through logs and browser history.

Putting the token in the URL doesn't stop it expiring anyway. It just copies a secret into server logs, proxy logs and browser history, where other people can find it and use it.

---

## Q8. Angular shows 403, Swagger with the same user can call it. What is different?

**Answer:**

It could be a different token, a different header, a different tenant host, or Swagger pointing at another environment. I compare the `Authorization` header (without pasting secrets anywhere), and I compare the origin and the route.

A 403 means "I know who you are, but you're not allowed". If the same user works in Swagger, the user isn't the problem; the two requests are different in some way. So I put them side by side and compare: which server they went to, the full URL, and whether the same kind of token was sent. Usually one small difference shows up, like a missing path segment or a call to the wrong host. It's like two letters to the same person where only one arrives: you compare the addresses on the envelopes.

**Example:**

The Angular interceptor attached the token only to `/api` calls. A call to `/incidents` was missing the prefix, so it went to the Angular host and got a 403/404 from the SPA server, not from the API.

**Real-world example:**

`environment.apiUrl` was missing a slash: `https://api.site.com` + `incidents` instead of `https://api.site.com/api/incidents`. Swagger used the full path. We fixed `apiUrl` and the interceptor.

**Cross-question:** CORS only on Swagger?

**Cross-answer:**

Swagger runs on the same origin as the API, and Angular doesn't. That's a different issue from a 403.

Swagger is served by the API itself, so its calls never cross origins and CORS never applies. Angular runs on its own origin, so the browser does apply CORS. A CORS problem shows up as a blocked request in the console, while a 403 is a real reply from the server saying no. They point to different fixes.

---

## Q9. You must show live “ticket taken” on two agents’ boards.

**Answer:**

I put a SignalR hub on the API and one connection service in Angular. The server broadcasts only to the **tenant group**. The boards update a signal. The hub requires auth.

Normally the browser asks and the server answers. SignalR keeps a connection open, so the server can push a message the moment something happens, like a phone call instead of checking your letterbox. The hub is the server side of that connection. One shared connection service in Angular stops every component from opening its own connection.

A group is a named list of connections. Sending only to the tenant's group means one company never sees another company's tickets. Requiring auth on the hub means only logged-in users can connect at all. When a message arrives, the board updates a signal (Angular's reactive value), and the screen redraws on its own.

**Example:**

```csharp
await _hub.Clients.Group(tenantKey).SendAsync("taken", incidentId, agentName);
```

```typescript
conn.on('taken', (id, name) => this.board.markTaken(id, name));
```

**Real-world example:**

Two agents opened the same new ticket and both clicked Take. Without live updates, they both thought they owned it. The hub plus a row version on the API allowed only one Take, and the other agent's board moved the card.

**Cross-question:** Poll every 2 seconds instead?

**Cross-answer:**

That's OK for a small team. But a live Take needs faster feedback, and SignalR fits that.

Polling means asking the server again and again, "anything new?". It's simple, but it sends lots of requests that mostly come back empty, and there's still a delay of up to the polling interval. With many users that load adds up. SignalR only sends something when there's actually news, and it arrives straight away.

---

## Q10. Product asks “just send TenantId from Angular, easier.” What do you say?

**Answer:**

I say no. The tenant comes from the authenticated host or from a claim the **server** issued. A `TenantId` sent by the client can be faked. Unrelated components sharing a `TenantService` in Angular is just for the UI, not isolation.

Anything the browser sends can be edited by the user, with DevTools or a tool like Postman. If the API trusts a `TenantId` in the request, anyone can change one number and read another company's data. A claim is a piece of information inside the token the server signed at login, so the user can't change it without breaking the signature. The host (for example a company's own subdomain) is checked by the server too. It's like a hotel key card: the hotel decides which room it opens, not the guest.

**Example:**

```csharp
var tenant = _tenantContext.Slug; // from validated claim / host map
// never: var tenant = dto.TenantId;
```

**Real-world example:**

A tester changed `tenantId` in DevTools and saw another org's list, until we made the API ignore the body and use the token. The Angular service still stores the tenant, but only for the UI label.

**Cross-question:** Super-admin switching tenant?

**Cross-answer:**

That's a **server** endpoint, “act as tenant X”, behind a permission check, which then issues a new token or server-side session. It's not a free field on every DTO.

Switching is a deliberate action the server approves and can record, not something any request can quietly ask for. After the switch, the new tenant lives in the new token or session, so the normal rule still holds: the tenant always comes from the server.

---

## Q11. Wallet shows 500, booking says insufficient for 400. How do you debug?

**Answer:**

The card shows the **available** balance, which is `Balance - active holds`. The `Balance` column in the table is the gross amount. If Balance is 1000 and 600 is on hold, available is 400. I check `HoldBalances` for Active rows on that wallet, including expiry. Then I confirm whether the UI is bound to `available` or `balance`.

A hold is money set aside for a booking that isn't finished yet. It still sits in the wallet, but it can't be spent again. It's like a hotel blocking an amount on your card at check-in: the money is still in your account, but you can't use it until the hold is released. So there are two numbers, the total in the wallet and the part that's free to use. When two screens disagree, the first thing to check is which of the two numbers each one is showing, and which holds are being counted.

**Example:**

```csharp
Balance = result.WalletBalance - result.HoldAmount
```

**Real-world example:**

In NriCare, `GetWalletBalanceAsync` returns that subtracted number. The transactions page also returns `TotalHoldAmountCount`. A “wrong wallet” bug is often just a hold the UI didn't show.

**Cross-question:** Expired hold still blocking book?

**Cross-answer:**

It can. Booking create adds up all Active holds and ignores `ExpiresAt`, but the balance API skips expired ones. That mismatch makes a good real debug story.

So the two places use different rules for which holds count. The balance screen leaves out a hold that has passed its `ExpiresAt`, so the user sees more money free. But booking create still counts that hold while it's Active, so it sees less money and refuses. Both are reading the same data; they just filter it differently.

---

## Q12. Angular got 401 while the user was on the booking chat. What should happen?

**Answer:**

The access JWT lasts about 20 minutes, and the chat hub also used `access_token` on the socket. The interceptor should refresh once, retry the REST call, and reconnect SignalR with the new token. If the refresh fails, keep the typed message in memory and send the user to login.

There are two connections here, and both use the same short-lived token. Normal REST calls go through the HTTP interceptor, so it can refresh and retry them. But the chat's SignalR connection was opened with the old token, and the interceptor doesn't touch it. So after the refresh, the chat connection has to be restarted with the new token, or it will keep failing. If even the refresh fails, the user has to log in again, but the message they were typing shouldn't be lost.

**Example:**

`POST /api/Auth/refresh-token` with the refresh string. The old refresh row gets revoked, and a new pair comes back.

**Real-world example:**

In NriCare, refresh rotates the token inside a transaction. If two tabs refresh the same token, the second one should fail, because the first one already revoked it. That's what rotation means. The user has to log in again on the tab that lost.

**Cross-question:** Put refresh token on the SignalR URL?

**Cross-answer:**

No. Only the short-lived access token goes there, and I don't log the full hub URL.

The refresh token lasts much longer and can create new access tokens, so it's the most valuable secret. URLs end up in logs, so anything on them should be short-lived. If the access token leaks, it stops working soon. If a refresh token leaks, someone can keep getting new access tokens.

---

## Q13. Service provider marks complete on mobile. Association dashboard should show money moved. It does not. Where do you look?

**Answer:**

First the Network tab: did `change-status` return success? Then `ReleasePayment`: holds should be Released, the user's Balance should go down, and the SP/platform/MA ledgers should be created. Then the association filter: an MA user should see their own wallet, and SuperAdmin sees all. If FCM failed, the screen just didn't refresh, and pull-to-refresh should still load the new ledger from GET wallet.

I follow the money in the order it moves, and stop at the first step that didn't happen. Did the status change actually save? Did the payment release run and write the ledger rows (the records of who got what)? Is the dashboard filtering to the right wallet? Only then do I look at notifications. FCM (Firebase Cloud Messaging) only tells the app "something changed, refresh". If it fails, the data on the server is still fine; the screen just doesn't know yet. It's like a missed delivery text: the parcel still arrived.

**Example:**

Complete with no verifier → `Completed` + `ReleasePayment`. With a verifier → `UnderReview` and **no** payout yet.

**Real-world example:**

That verifier branch surprises people. The money moves later, when verification is approved. A reject goes through the refund path (`ReleaseHold`).

**Cross-question:** Count the payout in Angular from the old booking object?

**Cross-answer:**

No. Reload the wallet from the API. The in-memory booking DTO doesn't contain the split.

The booking object in the browser was loaded before the payout happened, and it never had the details of how the money was split between the parties. Working it out in Angular would just be a guess. The wallet API reads the real ledger rows, so it's the source of truth.


---
## Additional Questions
### Q14. A user sees data from another tenant. What is your first priority?
**Answer:** Stop the access, and check the server-side tenant filter and authorization. I don't rely on Angular hiding the screen. Then I add a regression test for cross-tenant access.

This is a data leak, so stopping it comes before anything else, even before finding the exact cause. The real protection must be on the server: every query should filter by the tenant from the user's verified login, and every endpoint should check permissions. Hiding a screen in Angular doesn't help, because anyone can call the API directly.

After the fix, a regression test is an automated test that logs in as one tenant, tries to read another tenant's data, and expects to be refused. It runs on every build, so if someone breaks the filter again later, we find out before users do.
### Q15. The API returns 200 but Angular shows nothing. How do you debug?
**Answer:** I look at the actual response shape and the TypeScript model, then check the mapping and template bindings. I also make sure the request is hitting the environment I expect.

A 200 only means the request worked, not that the data looks how Angular expects. So I open the Network tab and look at the real JSON: are the property names the same as in my TypeScript interface (for example `title` vs `Title`)? Is the list wrapped inside another object, like `{ items: [...] }`, when the code expects a plain array? TypeScript types don't check anything at runtime, so a mismatch just gives `undefined` with no error.

Then I check the template: is an `*ngIf` or `@if` hiding the list, or is it bound to the wrong field? And finally, the environment: if the app is pointing at a test server with no data, everything "works" but the list is empty.
### Q16. The API is correct but the UI feels slow. What do you check?
**Answer:** Network payload, how many rows are rendered, change detection, methods called in templates, images and third-party components. I profile before adding another library.

Each of these can make a page feel slow even when the API is fast. A big payload takes time to download and parse. Thousands of rows on screen are heavy for the browser to draw. Change detection is how Angular checks whether the screen needs updating, and if a template calls a method like `getTotal()`, that method runs on every check, which adds up fast. Large images and heavy third-party widgets also slow down loading and scrolling.

Profiling means recording what the browser is actually doing, using the Performance panel in DevTools, so I can see which part takes the most time. Adding another library before I know the cause is like buying a faster car when the real problem is a flat tyre.
