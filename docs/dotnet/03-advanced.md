# .NET — Advanced

Architecture, tenants, speed, security. Still use short sentences.

---

## Q1. Clean architecture in a .NET solution?

**Answer:**

Clean architecture is a way of organising code so the important business rules don't depend on technical details like the database or the web framework. Think of a house: the rooms (business rules) shouldn't have to change just because you swap the plumbing (the database).

I split it into four layers.

Domain holds the entities. It has no project references. Entities are the core business objects, like an `Incident` or a `Wallet`, plus the rules that belong to them.

Application holds the use cases and interfaces. It depends on Domain only. A use case is one thing the app does, like "create a booking". The interfaces describe what the use case needs ("I need something that can save a booking") without saying how that's done.

Infrastructure holds things like EF and email. It implements those interfaces. This is where the real database code and the real email sending live.

API handles HTTP and auth, and wires everything together.

The arrows point inward. Controllers do not hide SQL. Domain does not know about EF. "Arrows point inward" means outer layers can depend on inner layers, never the other way round. Why it matters: I can change the database, the email provider or even the web framework without touching the business rules, and I can test those rules without a real database.

**Cross-question:** Where does `DbContext` live? Where do entities live?

**Cross-answer:**

Entities live in Domain. `DbContext` lives in Infrastructure. Application talks to it through an interface, or a port.

A "port" is just an interface that the inner layer owns, like a plug socket. Application says "I need a socket that can save and load bookings", and Infrastructure provides the plug that fits it, using `DbContext` behind the scenes. So Application never knows EF exists.

---

## Q2. Database per tenant vs one database plus `TenantId`?

**Answer:**

A tenant is one customer company using a shared app. The question is whether each customer gets its own database, or everyone shares one database where each row is tagged with a `TenantId`.

A database per tenant gives you strong walls between customers, but operations are harder. It's like giving every customer their own locked house: very safe, but you now have many houses to maintain, back up and upgrade.

A shared database is easier to run, but every query must filter by tenant, so it is easier to leak data. It's like a shared office with labelled drawers: cheaper and simpler, but if one query forgets to check the label, customer A can see customer B's data.

**Cross-question:** If each tenant has a database, do entities need `TenantId`?

**Cross-answer:**

Usually no. The database *is* the tenant. And I never trust a tenant id the browser sends. I resolve the tenant from a claim or from a known host map.

A claim is a fact stored inside the user's signed login token, so the user can't change it. A host map is a trusted list on the server, like "acme.example.com belongs to tenant Acme". Anything the browser sends freely, like a header or a query value, could be edited by an attacker to point at someone else's tenant.

---

## Q3. What is IDOR?

**Answer:**

IDOR stands for Insecure Direct Object Reference. It means someone reaches an object just by knowing or changing its id. Knowing id `123` does not mean you are allowed to see it. So I load the incident, then check the tenant and the permission. If the check fails, I return 404 or 403.

A simple example: my URL is `/incidents/123`. I change it to `/incidents/124` and suddenly I'm reading someone else's incident. That's IDOR. It's like a hotel where any key card opens any room as long as you know the room number. The fix is that the server checks "does this record belong to you, and are you allowed to see it?" on every request, not just "does this record exist?".

**Cross-question:** Are GUIDs enough?

**Cross-answer:**

They are harder to guess. But they are **not** permission.

A GUID is a long random id. It makes guessing harder, but ids leak all the time: in shared links, logs, screenshots or emails. Once someone has the id, a hard-to-guess id doesn't stop them. Only a real permission check on the server does.

---

## Q4. Request path from Kestrel to your action?

**Answer:**

Kestrel reads the HTTP request → middleware → routing → authorization → model binding → action → result → middleware again on the way out.

In plain words:
- Kestrel is the built-in web server. It receives the raw request from the network.
- Middleware is a chain of small steps every request passes through, like security checks at an airport: logging, error handling, HTTPS redirect and so on.
- Routing works out which controller action matches the URL.
- Authorization checks whether this user is allowed to call it.
- Model binding turns the URL, query string and body into the C# parameters of the action.
- The action runs my code and returns a result, which gets written as the response.
- On the way out, the response passes back through the same middleware in reverse order, so things like logging can record the final status.

Knowing this order helps when debugging, for example when auth runs too late or an error handler doesn't catch something because it was registered in the wrong place.

**Cross-question:** Can I read the body twice?

**Cross-answer:**

Not unless I turn on buffering. I prefer to let model binding read it. If I log bodies, I redact secrets first.

The request body is a stream, like water flowing through a pipe. Once it's been read, it's gone. Buffering keeps a copy so it can be "rewound" and read again, but that costs memory. Redacting means hiding passwords, tokens and similar values before anything gets written to a log.

---

## Q5. Thread pool starvation?

**Answer:**

The thread pool is a fixed group of worker threads that handle requests, like a team of waiters in a restaurant. Starvation happens when too many threads are blocked, for example by `.Result`, `Wait()`, `Thread.Sleep`, or sync IO. Then no free thread is left to finish the tasks those blocked threads are waiting on. The site feels dead.

Picture every waiter standing at the kitchen door waiting for food, while the cook needs a waiter to carry a plate before he can start the next dish. Nobody moves. The pool does add new threads, but slowly, so under load the app just hangs. The fix is to use `await` all the way through, so a thread is handed back to the pool while it waits, instead of standing there doing nothing.

**Cross-question:** How do you spot it?

**Cross-answer:**

I look at counters, traces, and sudden jumps in latency. Then I search the code for sync-over-async and for huge EF loads inside the request.

Counters are live numbers from the runtime, such as how many threads the pool has and how much work is queued. A typical sign is the thread count slowly climbing while response times shoot up, even though CPU is low. "Sync-over-async" means calling async code and then blocking on it with `.Result` or `Wait()`, which is the most common cause.

---

## Q6. `ExecuteUpdate` vs loading entities?

**Answer:**

`ExecuteUpdate` sends the SQL directly without tracking each row, so it is fast for bulk changes. The catch is you skip per-entity events. If I need domain events, I load the entities and save them the normal way.

The normal EF way is: load rows into memory, change them in C#, then call `SaveChanges`. That's fine for a few rows, but slow for thousands. `ExecuteUpdate` is like telling the database "mark all these as archived" in one instruction, instead of fetching every file, stamping it and putting it back. The downside is that EF never sees the objects, so any logic that runs when an entity is saved, like domain events or change tracking, does not happen.

**Cross-question:** Raw SQL?

**Cross-answer:**

I use LINQ and the EF APIs. I do not paste SQL strings into app code when the team forbids it.

LINQ is C# query syntax that EF turns into SQL for me. It's checked by the compiler, it uses parameters so values are passed safely, and it's easier to refactor. Hand-built SQL strings are easy to get wrong and, if user input is glued into them, open the door to SQL injection.

---

## Q7. MediatR / CQRS — when?

**Answer:**

CQRS means splitting reads from writes. MediatR gives you in-process messages and pipelines. Both are nice in a large app. For five CRUD endpoints, they are too much.

CQRS stands for Command Query Responsibility Segregation. Commands change data ("create booking"), queries only read data ("list bookings"). Keeping them separate lets each side stay simple and be tuned on its own. MediatR is a library where a controller sends a message like `CreateBookingCommand`, and one handler class picks it up. "Pipelines" means you can add shared steps around every handler, like validation or logging, in one place.

In a big app with many features, this keeps things tidy. In a tiny app, it just adds extra files and jumps between them, like setting up a full post room to pass notes between two desks.

**Cross-question:** Is CQRS event sourcing?

**Cross-answer:**

No. Event sourcing stores the events themselves as the source of truth. Most teams never need that.

With event sourcing, instead of storing "balance = 500", you store every change: "deposited 1000", "spent 300", "spent 200", and work out the balance by replaying them, like a bank statement. CQRS can be used without it. It just means reads and writes are handled separately.

---

## Q8. Outbox pattern?

**Answer:**

I save the business row and a “please publish this” row in the **same** database transaction. A worker publishes the message later. That way you never get “saved, but the message was lost”.

The problem it solves: often I need to save something and also tell other systems about it, for example by putting a message on a queue. Those are two different systems, so one can succeed and the other fail. A transaction means "all or nothing", so either both rows are saved or neither is. The "please publish" row works like an outbox tray on a desk: the letter is safely in the tray, and the post person (the background worker) picks it up and sends it, retrying if needed.

**Cross-question:** Publish to the queue then save EF?

**Cross-answer:**

If the save fails, you have already told the world a lie. A dual write without an outbox is a classic cause of incidents.

For example, the queue gets "order created", the email service sends a confirmation, but then the database save fails. Now the customer has an email for an order that doesn't exist. A "dual write" is writing to two systems separately with no shared transaction, and that gap is exactly where these problems happen.

---

## Q9. Idempotency keys?

**Answer:**

Idempotent means doing the same thing twice has the same effect as doing it once. The client sends a key with a POST. I store that key per tenant. If the client retries with the same key, we do not create a second payment.

Why it matters: networks fail. The user clicks "Pay", the request reaches the server and the payment is made, but the response gets lost. The app retries. Without a key, the user gets charged twice. With a key, the server sees "I've already handled this one" and returns the earlier result. It's like a ticket number at a counter: show the same ticket twice and you still only get served once.

**Cross-question:** Are all HTTP methods safe to retry?

**Cross-answer:**

GET, PUT and DELETE are meant to be. POST is not, unless I make it so. A timeout plus a retry on POST is how you get a double-charge bug.

GET only reads. PUT sets something to a given value, so setting it twice gives the same result. DELETE removes something, and removing it again changes nothing more. POST usually means "create a new thing", so each call can create another one. That's why POST needs an idempotency key.

---

## Q10. Data Protection keys on more than one server?

**Answer:**

ASP.NET Core Data Protection is the built-in system that encrypts things like login cookies so users can't read or change them. Cookies and some tokens are encrypted with a key ring. All servers must share those keys. If they don't, users bounce between servers and get logged out.

Picture two shops where each one locks its lockers with a different key. You put your bag in a locker at shop A, then get sent to shop B, and shop B's key can't open it. Behind a load balancer, the same thing happens: server A creates the cookie, the next request lands on server B, B can't decrypt it, and the user looks logged out. The fix is to store the key ring somewhere shared and protected, like a database, Redis or a cloud key store.

**Cross-question:** Public folder for the key ring?

**Cross-answer:**

No. The keys are secrets.

Anyone with those keys could decrypt cookies or create fake ones, and log in as any user. They belong in protected storage with limited access, and ideally encrypted at rest.

---

## Q11. How do you pick the tenant database?

**Answer:**

A factory reads the current tenant, from a claim or a validated host, and opens `tenant_{slug}` using a catalog. If the tenant is missing, we do not run the query.

In plain words: a factory is a class whose job is to build the right `DbContext` for each request. A slug is a short name for the tenant, like `acme`. The catalog is a central table that lists every tenant and where its database is. So for each request the app works out "this is Acme", looks up Acme in the catalog, and connects to Acme's database. If it can't work out the tenant, it stops. It "fails closed", which is much safer than falling back to some default database and showing the wrong customer's data.

**Cross-question:** Host header as tenant — safe?

**Cross-answer:**

Only if I map host → tenant from a trusted list. I never put the raw host into a connection string. And the client never sends a connection string.

The host header is the domain name the browser asked for, and an attacker can set it to anything. If it's matched against a known list, unknown values are simply rejected. If it's pasted straight into a connection string, the attacker can point the app at another database or break the string.

---

## Q12. Logs, metrics, traces?

**Answer:**

Logs are events. Metrics are numbers, like request time. Traces show the path a request takes through API → EF → HTTP. I tie all three together with a trace id. I do not log every SQL statement in production.

An everyday comparison is a parcel delivery:
- Logs are the diary entries: "parcel picked up", "delivery failed, nobody home".
- Metrics are the totals on the dashboard: parcels per hour, average delivery time.
- Traces are the full tracking history of one parcel, showing each stop and how long it spent there.

The trace id is like the tracking number. When something goes wrong, I search by that one id and see the logs, timings and calls for that exact request, instead of guessing.

**Cross-question:** Why not log all SQL?

**Cross-answer:**

Too much volume, personal data in the parameters, and extra cost. Logging slow queries is enough.

A busy app can run thousands of queries a minute, so the logs get huge and expensive to store and search. The parameter values can contain phone numbers, emails or amounts, which shouldn't sit in logs. Slow queries are the ones I actually need to look at.

---

## Q13. NativeAOT vs JIT?

**Answer:**

JIT compiles the code at run time. AOT compiles it ahead of time. That gives faster startup, but it is stricter, with less reflection allowed. Most business APIs still use JIT. The thing to know is that old reflection tricks can break under AOT.

JIT means "just in time": the app ships in an in-between form and the runtime turns it into machine code while it runs, a bit like a live translator. NativeAOT does all the translating before you ship, like handing over a printed translation, so the app starts almost instantly and uses less memory. That's useful for small services or containers that start and stop often. The cost is that the compiler must know all the code up front, so anything that discovers or creates code at run time, which is what reflection does, may not work.

**Cross-question:** What breaks first on AOT?

**Cross-answer:**

Dynamic `Assembly.Load`, some JSON code that doesn't use source generation, and emit.

All three create or find code while the app is running. `Assembly.Load` loads a DLL on the fly. Reflection-based JSON looks at your classes at run time to work out how to read and write them, whereas source generation writes that code at build time, so it is AOT-friendly. Emit means generating new code at run time, which an ahead-of-time app simply can't do.

---

## Q14. Permission design?

**Answer:**

I use stable codes like `incidents.read`. Roles are just bundles of those codes. Object-level checks still happen on the server. The UI only hides buttons.

A permission code is one specific thing you're allowed to do. A role, like "Manager", is just a named set of codes. Code checks the permission, not the role name, so if the business later decides managers can also export reports, I add a code to the role without changing code everywhere.

"Object-level" means checking this specific record, not just the action in general: "can you read incidents?" and also "can you read *this* incident?". Hiding a button in the UI is only for convenience. Anyone can call the API directly, so the real check must be on the server.

**Cross-question:** Permissions in the JWT or look up each time?

**Cross-answer:**

The JWT is faster but can go stale. The database is always fresh but heavier. I go hybrid: role in the token, and dangerous actions get checked again.

A JWT is a signed token the user carries with each request, like a printed visitor badge. Reading it is quick, but if an admin removes a permission, the old badge still says the user has it until it expires. Checking the database each time is always up to date, but adds a lookup to every request. So I keep everyday checks fast and re-check against the database for risky actions like deleting data or moving money.

---

## Q15. Slow endpoint playbook?

**Answer:**

First I reproduce it. Then I measure: is it network, CPU, SQL, or JSON? I count the SQL queries. I fix N+1 or over-selecting. Then I measure again.

The point is to find the actual cause before changing anything, like a doctor running tests before prescribing. N+1 is when code runs one query to get a list, then one more query for every item in it. Loading 100 orders turns into 101 queries. Over-selecting means pulling columns or rows I don't need, like loading full records just to show a name. Measuring again at the end proves the fix actually helped.

**Cross-question:** Cache first?

**Cross-answer:**

Not if the query is wrong or the cache key is missing the tenant. I fix the query first.

A cache hides a slow query instead of fixing it, and adds new problems such as stale data. Worse, if the cache key doesn't include the tenant, tenant A's cached result can be handed to tenant B. That turns a speed problem into a data leak.

---

## Q16. gRPC vs REST vs GraphQL?

**Answer:**

These are three ways for programs to talk to each other over a network.

REST is easy for browsers. gRPC is fast for service-to-service calls and uses protobuf. GraphQL gives flexible queries, but has more moving parts and needs auth per field.

- REST uses normal URLs, HTTP verbs and usually JSON, so any browser or tool can call it.
- gRPC sends compact binary messages defined by protobuf, a strict contract file. It's faster and smaller, which suits backend services calling each other a lot.
- GraphQL lets the client ask for exactly the fields it wants in one request. That's flexible, but because a client can ask for any field, you have to think about who may see each field, and about very expensive queries.

**Cross-question:** Replace REST with gRPC for Angular?

**Cross-answer:**

Usually no. Browser tools, file handling, and HTTP status codes still fit REST/JSON better.

Browsers can't call normal gRPC directly. You need gRPC-Web and usually a proxy in between. With REST I can see requests clearly in browser dev tools, upload and download files simply, and use standard status codes like 404 and 401 that Angular already handles well.

---

## Q17. What is a captive dependency?

**Answer:**

In .NET dependency injection, services have lifetimes. A singleton lives for the whole app. A scoped service lives for one request.

A captive dependency is when a singleton holds on to a scoped service. That scoped service then becomes a singleton by accident. A `DbContext` shared across requests is the famous case. The DI container can warn you about it.

It's like a shared office printer that keeps one visitor's ID badge stuck inside it: every later visitor ends up using that first person's badge. With `DbContext`, many requests share one instance. It isn't thread-safe, it keeps growing as it tracks more and more objects, and it can mix up data between users. Scope validation in the container, which is on by default in Development, throws an error when it spots this.

**Cross-question:** How do you use a scoped service inside a singleton?

**Cross-answer:**

I inject `IServiceScopeFactory`, create a scope for each operation, and dispose it when done.

So instead of holding one `DbContext` forever, the singleton opens a fresh "mini request" for each piece of work, gets a new `DbContext` from it, uses it, and throws it away. This is the usual pattern in background services.

---

## Q18. YARP / reverse proxy — why in front of Kestrel?

**Answer:**

A reverse proxy is a server that sits in front of your app and receives requests first, like a receptionist who greets visitors and sends them to the right office. YARP is Microsoft's reverse proxy library for .NET.

For TLS, path routing, running more than one app, and rate limits. Kestrel still runs the app. I don't expose raw Kestrel to the internet without a plan.

TLS is the HTTPS encryption, and the proxy can handle certificates in one place. Path routing means sending `/api` to one app and `/admin` to another. Rate limits stop one client from flooding the app with requests. Kestrel is good and fast, but putting a proxy in front gives one controlled front door for all of this.

**Cross-question:** Forwarded headers?

**Cross-answer:**

They let the app know the request was HTTPS and what the real client IP is. I only trust proxies I own, not any `X-Forwarded-*` header coming from the outside world.

Behind a proxy, the app only sees the proxy's own connection, so the proxy adds headers like `X-Forwarded-For` (original client IP) and `X-Forwarded-Proto` (http or https). But anyone can type those headers into a request. If the app trusted them from anywhere, an attacker could fake their IP to get around rate limits or IP blocks.

---

## Q19. How do you keep secrets out of logs and errors?

**Answer:**

No tokens in URLs. No connection strings in 500 responses. The exception middleware sends back a generic message. I log ids, not payloads that contain passwords.

Why each one matters:
- URLs get saved in browser history, proxy logs and server logs, so a token in a URL gets copied everywhere.
- A 500 error that shows a connection string hands an attacker the database address and maybe a password.
- The exception middleware catches errors in one place and returns something like "Something went wrong", while the full details go only to our own logs.
- Logging an id like "user 42 failed to log in" is enough to investigate, without storing what they typed.

**Cross-question:** Dev vs prod exception pages?

**Cross-answer:**

The developer exception page is for local only. Production uses ProblemDetails without stack traces.

The developer page shows the full error, stack trace and code lines, which is great while coding but a gift to attackers. ProblemDetails is a standard JSON error format with a status, a title and maybe a trace id, so users and the frontend get a clean error and nothing internal leaks.

---

## Q20. EF compiled models / precompiled queries — when?

**Answer:**

Large models take time to load at startup. Compiled models help with that. I bring it up when the app has many entities and a slow boot. It is not the first fix for a slow page.

When EF starts, it builds a picture of all your entities, their columns and relationships. This is the "model". With hundreds of entities, that can take noticeable time on the first request. A compiled model builds that picture at build time and ships it with the app, so startup is quicker. It's like printing a map in advance instead of drawing it every morning. But it only helps startup. If a page is slow every time, the problem is usually the query itself.

**Cross-question:** Does this replace indexing?

**Cross-answer:**

No. Indexes are a database thing. It's a different problem.

Compiled models make EF itself start faster on the app side. An index helps the database find rows quickly, like the index at the back of a book, so it doesn't scan the whole table. A slow query on a big table still needs the right index, whatever EF does.

---

## Q21. How would you design a multi-tenant “current user” service?

**Answer:**

I'd make a scoped `IUserContext` that gets filled from the claims after auth. It holds UserId, Tenant, and permissions. If it is empty on a protected request, it fails closed. No static `CurrentUser` sitting on a singleton.

"Scoped" means one fresh instance per request, so each request has its own user, like each customer at a bank counter getting their own form. Claims are the facts inside the verified login token. "Fails closed" means if we don't know who the user is, we refuse, rather than carry on with an empty or default user.

A static or singleton `CurrentUser` is shared by every request at the same time. With two users hitting the API together, one could end up acting as the other. That's a serious data leak.

**Cross-question:** Async and user context?

**Cross-answer:**

`AsyncLocal` can work, but it is easy to leak data across requests if you get it wrong. For web apps, scoped services plus HttpContext are clearer.

`AsyncLocal` stores a value that follows the async flow of code, even across `await`. It's handy but invisible: values can be copied into background work started from a request, or not set where you expect, and those bugs are hard to find. Scoped services and `HttpContext` are tied to the request in an obvious way, so they're easier to reason about.

---

## Q22. What is backpressure?

**Answer:**

If work arrives faster than we can handle it, we have to slow down the producer, or drop work with a plan. Tools for that are bounded channels, 429 responses, and queue limits. An unlimited in-memory queue will eventually take the process down.

Think of a sink with the tap on full and a small drain. If you don't turn the tap down, it overflows. Backpressure is how the system says "slow down". A bounded channel is a queue with a maximum size, so the producer has to wait when it's full. A 429 response means "Too Many Requests, try again later". Without limits, the waiting work keeps piling up in memory until the app runs out of memory and crashes, and then all of that work is lost anyway.

**Cross-question:** Hangfire queue growing forever?

**Cross-answer:**

I set alerts on queue length and failed jobs. I don't just keep adding servers with no limit.

A queue that keeps growing means jobs are coming in faster than they finish, or they keep failing and retrying. Alerts tell me early. Then I look for the cause, like a slow job, a broken downstream service or a retry loop. Adding servers might hide it for a while, but it costs money and doesn't fix the cause.

---

## Q23. How do you version a database in a tenant-per-db world?

**Answer:**

Every tenant database gets the same migrations. A catalog tracks which version each one is on. Rollout is planned: one tenant first, then batches. I don't invent a special schema per customer.

A migration is a scripted change to the database structure, like "add a column". With many tenant databases, every one of them needs the same changes in the same order, so they stay alike and the same code works for all of them. The catalog is like a checklist showing which tenant is up to date. Trying one tenant first is like testing a new recipe on one table before serving the whole restaurant: if something goes wrong, only one customer is affected.

**Cross-question:** One tenant needs a custom column?

**Cross-answer:**

That is a product smell. I'd prefer a safe extension, like JSON metadata, over a one-off snowflake schema, if the project already uses that pattern.

A "smell" is a hint that something may be wrong with the design. A "snowflake" schema here means one tenant's database is unique, so every future migration and every piece of code has to remember that special case. A JSON metadata column lets each tenant store extra fields without changing the table structure, so all databases stay the same shape.

---

## Q24. What is the difference between authentication schemes?

**Answer:**

A scheme is how the app identifies the caller: cookies, JWT bearer, maybe API keys. One app can have more than one. A browser site might use cookies, and a SPA might use bearer tokens. I don't mix them on the same endpoint by accident without thinking it through.

Each scheme is a different kind of ID check, like a building that accepts staff badges at one door and visitor passes at another. Cookies are sent by the browser automatically. A bearer token is a JWT that the frontend code attaches in the `Authorization` header. An API key is a fixed secret used by another system. Mixing them carelessly can open security gaps. For example, cookie-based endpoints need protection against cross-site request forgery, and bearer-token endpoints work differently.

**Cross-question:** `[Authorize(AuthenticationSchemes = "...")]`?

**Cross-answer:**

It picks which scheme that endpoint accepts. If you registered two, the default scheme is not always the one you think.

So if an endpoint should only accept JWT bearer tokens, I name that scheme on it. Otherwise the app uses whatever default was set at startup, which might be cookies, and requests can fail with 401 or be accepted in a way I didn't plan.

---

## Q25. How do you test multi-tenant isolation?

**Answer:**

Tenant isolation means one customer can never see or change another customer's data.

I set up two users in two tenants. User A’s token must not be able to read B’s id. That is an automated test. I also try a guessed id. This is not optional.

The test logs in as a user from tenant A, then calls the API asking for a record that belongs to tenant B, and expects a 404 or 403. It's like checking that your flat key doesn't open your neighbour's door. Because it's automated, it runs on every build, so a new endpoint that forgets the tenant check gets caught before it reaches production.

**Cross-question:** Is a manual click enough?

**Cross-answer:**

No. Someone will forget to check on the next feature. Tests stay.

A manual check only proves it worked on the day someone clicked through it. Code keeps changing, and an automated test checks again every time, without anyone needing to remember.

---

## Q26. Walk me through booking money in NriCare.

**Answer:**

When an NR user books, we start a database transaction. We lock the wallet with Postgres `FOR UPDATE`. The available money is `Balance` minus active holds. If that is enough, we insert a `HoldBalance` and a Hold ledger row. We do **not** subtract from `Balance` yet. The money is just reserved. When the job is completed, `ReleasePayment` debits the user and splits the money to the service provider, platform, association, and verifier.

In plain words: the transaction makes all these steps succeed or fail together. `FOR UPDATE` locks the wallet row so no other request can change it while we work. A hold works like a hotel holding an amount on your card at check-in. The money is set aside, so you can't spend it elsewhere, but it hasn't actually been taken yet. The ledger row is a permanent record of what happened, like a line in a bank statement. Only when the job is done does the real payment happen and get shared out.

**Cross-question:** Why not debit immediately?

**Cross-answer:**

Because the booking can still be cancelled while it is under discussion. A hold keeps the amount aside without paying the provider yet.

If we debited right away, a cancellation would mean working out refunds. With a hold, cancelling just releases the reserved amount, and the user's balance was never touched.

---

## Q27. Two booking requests at the same time for one wallet. What happens?

**Answer:**

When creating a booking we `SELECT ... FOR UPDATE` that wallet row, so the second request has to wait. After the first hold is saved, the second request sees a smaller available balance and fails with insufficient funds. That is pessimistic locking.

Think of a shared notebook with one pen. The first person takes the pen and writes. The second has to wait until the pen is put down, and then reads the updated page. Without the lock, both requests could read the same balance at the same moment, both think there's enough money, and both create holds, spending more than the user has. "Pessimistic" means we assume a clash might happen, so we lock first instead of checking for clashes afterwards.

**Cross-question:** Does quotation accept use the same lock?

**Cross-answer:**

Not today. Accepting a quotation checks the available balance without `FOR UPDATE`, so two accepts could race. I would reuse the same lock, or add a unique idempotency index.

A "race" means two requests running at the same time, where the result depends on which one gets there first. Here both could pass the balance check before either saves. Reusing the lock makes them take turns. A unique idempotency index makes the database itself reject a second identical accept, even if the code lets it through.

---

## Q28. What would you change in production logging?

**Answer:**

Serilog request logging is good as it is: method, path, status, time. I would turn **off** `EnableSensitiveDataLogging` and `EnableDetailedErrors` outside Development. Those print SQL parameter values, which can include phone numbers and amounts. Also, the session middleware logs every user id at Information level. That is noisy in production; Warning is enough.

Serilog is the logging library. Those two EF settings are useful while debugging on my machine, but in production they would write customers' personal data into logs, which many more people can read and which are kept for a long time. Log levels are like volume settings: Information records normal everyday events, Warning only records things that may need attention. Logging every user id on every request fills the logs with noise and makes real problems harder to find.

**Cross-question:** Log the JWT on 401?

**Cross-answer:**

Never. I log the user id and the path. Tokens and passwords stay out.

A JWT is a working key. Anyone who copies it from a log could use it to act as that user until it expires. The user id and path are enough to see who was refused and where.

---

## Q29. How do you split payment on booking complete?

**Answer:**

`ReleasePayment` runs in a transaction. The user wallet `Balance` goes down by the hold total. The service provider gets the total minus the platform, association, and verifier fees. The SuperAdmin wallet gets the platform fee. The association and verifier wallets get their fees, if they are present. The holds become Released. Each split has its own ledger row and key.

It's like splitting a restaurant bill: the customer pays the total, and it gets shared between the cook, the restaurant and the delivery partner. Doing it all in one transaction means either everyone gets their share or nobody does. Giving each split its own ledger row and key means every movement of money can be traced, and the same split can't be recorded twice.

**Cross-question:** What if SaveChanges fails in the middle?

**Cross-answer:**

The explicit transaction rolls back, so there is no half payout. After the commit we queue the PDF report jobs. Those can fail without undoing any money, which is acceptable as “eventually generate the receipt”.

"Rolls back" means the database undoes every change made in that transaction, as if nothing happened. So the user is never debited while the provider goes unpaid. The PDF receipts are queued only after the money part is safely committed. If a PDF job fails, it can be retried later, and the payment itself is still correct.

---

## Q30. How do subscriptions debit the wallet safely?

**Answer:**

In `ProcessSubscriptionInternal` we use `ExecuteUpdateAsync` to update the payer wallet only if `Balance >= total`. If zero rows were updated, we fail. Then we write the ledger rows, the subscription, and the payment log in the same transaction.

The check and the deduction happen in one single database statement: "take the money only if there's enough". The database runs that as one step, so nothing can sneak in between the check and the update. If the balance was too low, the statement changes zero rows, and we treat that as "insufficient funds". The rest of the records are saved in the same transaction, so we never end up with money taken but no subscription, or the other way round.

**Cross-question:** Why `ExecuteUpdate` instead of load-and-save?

**Cross-answer:**

Load-and-save can overwrite a top-up that happened at the same time. The SQL `WHERE Balance >= amount` makes it one atomic decrement. There is no rowversion on Wallet, so this pattern matters.

Example: the balance is 1000. We load it and plan to save 1000 − 300 = 700. Meanwhile a top-up of 500 makes it 1500. Then we save 700, and the top-up is lost. "Atomic" means it happens as one step that can't be interrupted. A rowversion is a column that changes on every update, so EF can notice "someone else changed this row" and stop. Since the Wallet has none, the single-statement update is what keeps the balance correct.

---

## Q31. How would you improve Razorpay verify?

**Answer:**

Today `verify` captures the payment, then credits the wallet in a second step. It also picks the latest payment log that has no Razorpay payment id, and it doesn't filter by user. If the capture works but the wallet credit fails, a retry can credit twice. I would match the log by order id, put a unique constraint on the payment id, do the credit inside a transaction, and add a webhook as the source of truth.

"Capture" means actually taking money that the customer has authorised. Why each fix helps:
- Matching by order id makes sure we update the exact payment record for this order, not just "the latest one", which could belong to someone else.
- A unique constraint on the payment id means the database refuses to record the same payment twice, even if the code runs twice.
- Doing the credit inside a transaction makes the wallet credit and the payment log update succeed or fail together.
- A webhook is a call Razorpay makes to our server when a payment changes state. It doesn't depend on the user's app staying open, so it's a more reliable signal than the client calling verify.

**Cross-question:** Why manual capture?

**Cross-answer:**

Orders are created with `payment_capture = 0`. We capture only after the client calls verify. That's fine as long as verify is honest. A webhook is safer if the app dies right after capture.

With `payment_capture = 0`, Razorpay only authorises the payment. The money is approved but not taken until we capture it. That gives our server a chance to check the payment before taking it. The risk is in the gap: if the app or server dies right after capture but before the wallet is credited, the user has paid but sees nothing. A webhook would still tell our server about the payment, so it could finish the job.

---

## Q32. TickerQ vs Hangfire — what do you actually use?

**Answer:**

TickerQ. Job methods are marked with `[TickerFunction]`. We enqueue them with `ITimeTickerManager`. The store is EF tables in the same Postgres database. Max concurrency is 4. The dashboard is at `/tickerq`. Each job creates its own DI scope, so it doesn't reuse a request's DbContext.

TickerQ, like Hangfire, is a library for background jobs, meaning work that runs outside the user's request, such as generating a PDF or sending a reminder later. In plain words: the attribute marks which methods can run as jobs, and the manager is how we schedule them. Jobs are saved in database tables, so they survive a restart. At most 4 jobs run at once, so they don't overload the server. The dashboard is a web page to see what ran and what failed. Each job getting its own scope means it gets its own fresh `DbContext`, which avoids the captive dependency problem from Q17.

**Cross-question:** What if the API restarts?

**Cross-answer:**

Missed tickers can be cancelled on restart (`CancelMissedTickersOnApplicationRestart`). Right now, document expiry schedules **another** daily job on every startup, so I would make that schedule idempotent.

A "missed ticker" is a job that was due while the app was down, and this setting decides to cancel those rather than run them all at once on startup. The document expiry problem is that each restart adds one more copy of the same daily job, so after a few deploys the check could run several times a day. Making it idempotent means "create this daily job only if it doesn't already exist", so restarting any number of times still leaves exactly one.


---
## Additional Questions
### Q33. What is idempotency in an API?
**Answer:** If the same request is repeated, it should not cause the same business effect twice. For payments or wallet operations, I use a unique business key or idempotency key and enforce it in the database.

For example, if a "pay ₹500" request is sent twice because of a network retry, the user should still be charged only once. Enforcing it in the database, with a unique index, matters because the database is the final gatekeeper. Even if two copies of the request arrive at the same moment on different servers, only one can save, and the other is rejected.

### Q34. Why is an outbox useful?
**Answer:** If I save a database change and publish an event separately, one can succeed while the other fails. The outbox stores the event in the same database transaction, and a worker publishes it later.

So the data change and the "tell others about it" note are saved together or not at all. If the message system is down for a while, nothing is lost. The note waits in the outbox table, and the worker keeps trying until it's sent. It's like dropping a letter in a post box instead of trying to hand it to the recipient yourself.

### Q35. How do you handle concurrency in a wallet?
**Answer:** I wrap the balance and ledger operation in a database transaction, with a lock or concurrency control around it. I also make the business transaction id unique, so retries can't double-charge.

Concurrency means several requests touching the same wallet at the same time. The transaction makes sure the balance change and its ledger record are saved together. The lock, or a conditional update like "only if the balance is enough", stops two requests from both spending the same money. The unique transaction id stops a repeated request from being processed twice. Together they keep the balance correct even when things happen at once or get retried.
