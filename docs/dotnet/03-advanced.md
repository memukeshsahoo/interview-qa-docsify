# .NET — Advanced

Architecture, tenants, speed, security. Still use short sentences.

---

## Q1. Clean architecture in a .NET solution?

**Answer:**

I split it into four layers.

Domain holds the entities. It has no project references.

Application holds the use cases and interfaces. It depends on Domain only.

Infrastructure holds things like EF and email. It implements those interfaces.

API handles HTTP and auth, and wires everything together.

The arrows point inward. Controllers do not hide SQL. Domain does not know about EF.

**Cross-question:** Where does `DbContext` live? Where do entities live?

**Cross-answer:**

Entities live in Domain. `DbContext` lives in Infrastructure. Application talks to it through an interface, or a port.

---

## Q2. Database per tenant vs one database plus `TenantId`?

**Answer:**

A database per tenant gives you strong walls between customers, but operations are harder.

A shared database is easier to run, but every query must filter by tenant, so it is easier to leak data.

**Cross-question:** If each tenant has a database, do entities need `TenantId`?

**Cross-answer:**

Usually no. The database *is* the tenant. And I never trust a tenant id the browser sends. I resolve the tenant from a claim or from a known host map.

---

## Q3. What is IDOR?

**Answer:**

It means someone reaches an object just by knowing or changing its id. Knowing id `123` does not mean you are allowed to see it. So I load the incident, then check the tenant and the permission. If the check fails, I return 404 or 403.

**Cross-question:** Are GUIDs enough?

**Cross-answer:**

They are harder to guess. But they are **not** permission.

---

## Q4. Request path from Kestrel to your action?

**Answer:**

Kestrel reads the HTTP request → middleware → routing → authorization → model binding → action → result → middleware again on the way out.

**Cross-question:** Can I read the body twice?

**Cross-answer:**

Not unless I turn on buffering. I prefer to let model binding read it. If I log bodies, I redact secrets first.

---

## Q5. Thread pool starvation?

**Answer:**

It happens when too many threads are blocked, for example by `.Result`, `Wait()`, `Thread.Sleep`, or sync IO. Then no free thread is left to finish the tasks those blocked threads are waiting on. The site feels dead.

**Cross-question:** How do you spot it?

**Cross-answer:**

I look at counters, traces, and sudden jumps in latency. Then I search the code for sync-over-async and for huge EF loads inside the request.

---

## Q6. `ExecuteUpdate` vs loading entities?

**Answer:**

`ExecuteUpdate` sends the SQL directly without tracking each row, so it is fast for bulk changes. The catch is you skip per-entity events. If I need domain events, I load the entities and save them the normal way.

**Cross-question:** Raw SQL?

**Cross-answer:**

I use LINQ and the EF APIs. I do not paste SQL strings into app code when the team forbids it.

---

## Q7. MediatR / CQRS — when?

**Answer:**

CQRS means splitting reads from writes. MediatR gives you in-process messages and pipelines. Both are nice in a large app. For five CRUD endpoints, they are too much.

**Cross-question:** Is CQRS event sourcing?

**Cross-answer:**

No. Event sourcing stores the events themselves as the source of truth. Most teams never need that.

---

## Q8. Outbox pattern?

**Answer:**

I save the business row and a “please publish this” row in the **same** database transaction. A worker publishes the message later. That way you never get “saved, but the message was lost”.

**Cross-question:** Publish to the queue then save EF?

**Cross-answer:**

If the save fails, you have already told the world a lie. A dual write without an outbox is a classic cause of incidents.

---

## Q9. Idempotency keys?

**Answer:**

The client sends a key with a POST. I store that key per tenant. If the client retries with the same key, we do not create a second payment.

**Cross-question:** Are all HTTP methods safe to retry?

**Cross-answer:**

GET, PUT and DELETE are meant to be. POST is not, unless I make it so. A timeout plus a retry on POST is how you get a double-charge bug.

---

## Q10. Data Protection keys on more than one server?

**Answer:**

Cookies and some tokens are encrypted with a key ring. All servers must share those keys. If they don't, users bounce between servers and get logged out.

**Cross-question:** Public folder for the key ring?

**Cross-answer:**

No. The keys are secrets.

---

## Q11. How do you pick the tenant database?

**Answer:**

A factory reads the current tenant, from a claim or a validated host, and opens `tenant_{slug}` using a catalog. If the tenant is missing, we do not run the query.

**Cross-question:** Host header as tenant — safe?

**Cross-answer:**

Only if I map host → tenant from a trusted list. I never put the raw host into a connection string. And the client never sends a connection string.

---

## Q12. Logs, metrics, traces?

**Answer:**

Logs are events. Metrics are numbers, like request time. Traces show the path a request takes through API → EF → HTTP. I tie all three together with a trace id. I do not log every SQL statement in production.

**Cross-question:** Why not log all SQL?

**Cross-answer:**

Too much volume, personal data in the parameters, and extra cost. Logging slow queries is enough.

---

## Q13. NativeAOT vs JIT?

**Answer:**

JIT compiles the code at run time. AOT compiles it ahead of time. That gives faster startup, but it is stricter, with less reflection allowed. Most business APIs still use JIT. The thing to know is that old reflection tricks can break under AOT.

**Cross-question:** What breaks first on AOT?

**Cross-answer:**

Dynamic `Assembly.Load`, some JSON code that doesn't use source generation, and emit.

---

## Q14. Permission design?

**Answer:**

I use stable codes like `incidents.read`. Roles are just bundles of those codes. Object-level checks still happen on the server. The UI only hides buttons.

**Cross-question:** Permissions in the JWT or look up each time?

**Cross-answer:**

The JWT is faster but can go stale. The database is always fresh but heavier. I go hybrid: role in the token, and dangerous actions get checked again.

---

## Q15. Slow endpoint playbook?

**Answer:**

First I reproduce it. Then I measure: is it network, CPU, SQL, or JSON? I count the SQL queries. I fix N+1 or over-selecting. Then I measure again.

**Cross-question:** Cache first?

**Cross-answer:**

Not if the query is wrong or the cache key is missing the tenant. I fix the query first.

---

## Q16. gRPC vs REST vs GraphQL?

**Answer:**

REST is easy for browsers. gRPC is fast for service-to-service calls and uses protobuf. GraphQL gives flexible queries, but has more moving parts and needs auth per field.

**Cross-question:** Replace REST with gRPC for Angular?

**Cross-answer:**

Usually no. Browser tools, file handling, and HTTP status codes still fit REST/JSON better.

---

## Q17. What is a captive dependency?

**Answer:**

It's when a singleton holds on to a scoped service. That scoped service then becomes a singleton by accident. A `DbContext` shared across requests is the famous case. The DI container can warn you about it.

**Cross-question:** How do you use a scoped service inside a singleton?

**Cross-answer:**

I inject `IServiceScopeFactory`, create a scope for each operation, and dispose it when done.

---

## Q18. YARP / reverse proxy — why in front of Kestrel?

**Answer:**

For TLS, path routing, running more than one app, and rate limits. Kestrel still runs the app. I don't expose raw Kestrel to the internet without a plan.

**Cross-question:** Forwarded headers?

**Cross-answer:**

They let the app know the request was HTTPS and what the real client IP is. I only trust proxies I own, not any `X-Forwarded-*` header coming from the outside world.

---

## Q19. How do you keep secrets out of logs and errors?

**Answer:**

No tokens in URLs. No connection strings in 500 responses. The exception middleware sends back a generic message. I log ids, not payloads that contain passwords.

**Cross-question:** Dev vs prod exception pages?

**Cross-answer:**

The developer exception page is for local only. Production uses ProblemDetails without stack traces.

---

## Q20. EF compiled models / precompiled queries — when?

**Answer:**

Large models take time to load at startup. Compiled models help with that. I bring it up when the app has many entities and a slow boot. It is not the first fix for a slow page.

**Cross-question:** Does this replace indexing?

**Cross-answer:**

No. Indexes are a database thing. It's a different problem.

---

## Q21. How would you design a multi-tenant “current user” service?

**Answer:**

I'd make a scoped `IUserContext` that gets filled from the claims after auth. It holds UserId, Tenant, and permissions. If it is empty on a protected request, it fails closed. No static `CurrentUser` sitting on a singleton.

**Cross-question:** Async and user context?

**Cross-answer:**

`AsyncLocal` can work, but it is easy to leak data across requests if you get it wrong. For web apps, scoped services plus HttpContext are clearer.

---

## Q22. What is backpressure?

**Answer:**

If work arrives faster than we can handle it, we have to slow down the producer, or drop work with a plan. Tools for that are bounded channels, 429 responses, and queue limits. An unlimited in-memory queue will eventually take the process down.

**Cross-question:** Hangfire queue growing forever?

**Cross-answer:**

I set alerts on queue length and failed jobs. I don't just keep adding servers with no limit.

---

## Q23. How do you version a database in a tenant-per-db world?

**Answer:**

Every tenant database gets the same migrations. A catalog tracks which version each one is on. Rollout is planned: one tenant first, then batches. I don't invent a special schema per customer.

**Cross-question:** One tenant needs a custom column?

**Cross-answer:**

That is a product smell. I'd prefer a safe extension, like JSON metadata, over a one-off snowflake schema, if the project already uses that pattern.

---

## Q24. What is the difference between authentication schemes?

**Answer:**

A scheme is how the app identifies the caller: cookies, JWT bearer, maybe API keys. One app can have more than one. A browser site might use cookies, and a SPA might use bearer tokens. I don't mix them on the same endpoint by accident without thinking it through.

**Cross-question:** `[Authorize(AuthenticationSchemes = "...")]`?

**Cross-answer:**

It picks which scheme that endpoint accepts. If you registered two, the default scheme is not always the one you think.

---

## Q25. How do you test multi-tenant isolation?

**Answer:**

I set up two users in two tenants. User A’s token must not be able to read B’s id. That is an automated test. I also try a guessed id. This is not optional.

**Cross-question:** Is a manual click enough?

**Cross-answer:**

No. Someone will forget to check on the next feature. Tests stay.

---

## Q26. Walk me through booking money in NriCare.

**Answer:**

When an NR user books, we start a database transaction. We lock the wallet with Postgres `FOR UPDATE`. The available money is `Balance` minus active holds. If that is enough, we insert a `HoldBalance` and a Hold ledger row. We do **not** subtract from `Balance` yet. The money is just reserved. When the job is completed, `ReleasePayment` debits the user and splits the money to the service provider, platform, association, and verifier.

**Cross-question:** Why not debit immediately?

**Cross-answer:**

Because the booking can still be cancelled while it is under discussion. A hold keeps the amount aside without paying the provider yet.

---

## Q27. Two booking requests at the same time for one wallet. What happens?

**Answer:**

When creating a booking we `SELECT ... FOR UPDATE` that wallet row, so the second request has to wait. After the first hold is saved, the second request sees a smaller available balance and fails with insufficient funds. That is pessimistic locking.

**Cross-question:** Does quotation accept use the same lock?

**Cross-answer:**

Not today. Accepting a quotation checks the available balance without `FOR UPDATE`, so two accepts could race. I would reuse the same lock, or add a unique idempotency index.

---

## Q28. What would you change in production logging?

**Answer:**

Serilog request logging is good as it is: method, path, status, time. I would turn **off** `EnableSensitiveDataLogging` and `EnableDetailedErrors` outside Development. Those print SQL parameter values, which can include phone numbers and amounts. Also, the session middleware logs every user id at Information level. That is noisy in production; Warning is enough.

**Cross-question:** Log the JWT on 401?

**Cross-answer:**

Never. I log the user id and the path. Tokens and passwords stay out.

---

## Q29. How do you split payment on booking complete?

**Answer:**

`ReleasePayment` runs in a transaction. The user wallet `Balance` goes down by the hold total. The service provider gets the total minus the platform, association, and verifier fees. The SuperAdmin wallet gets the platform fee. The association and verifier wallets get their fees, if they are present. The holds become Released. Each split has its own ledger row and key.

**Cross-question:** What if SaveChanges fails in the middle?

**Cross-answer:**

The explicit transaction rolls back, so there is no half payout. After the commit we queue the PDF report jobs. Those can fail without undoing any money, which is acceptable as “eventually generate the receipt”.

---

## Q30. How do subscriptions debit the wallet safely?

**Answer:**

In `ProcessSubscriptionInternal` we use `ExecuteUpdateAsync` to update the payer wallet only if `Balance >= total`. If zero rows were updated, we fail. Then we write the ledger rows, the subscription, and the payment log in the same transaction.

**Cross-question:** Why `ExecuteUpdate` instead of load-and-save?

**Cross-answer:**

Load-and-save can overwrite a top-up that happened at the same time. The SQL `WHERE Balance >= amount` makes it one atomic decrement. There is no rowversion on Wallet, so this pattern matters.

---

## Q31. How would you improve Razorpay verify?

**Answer:**

Today `verify` captures the payment, then credits the wallet in a second step. It also picks the latest payment log that has no Razorpay payment id, and it doesn't filter by user. If the capture works but the wallet credit fails, a retry can credit twice. I would match the log by order id, put a unique constraint on the payment id, do the credit inside a transaction, and add a webhook as the source of truth.

**Cross-question:** Why manual capture?

**Cross-answer:**

Orders are created with `payment_capture = 0`. We capture only after the client calls verify. That's fine as long as verify is honest. A webhook is safer if the app dies right after capture.

---

## Q32. TickerQ vs Hangfire — what do you actually use?

**Answer:**

TickerQ. Job methods are marked with `[TickerFunction]`. We enqueue them with `ITimeTickerManager`. The store is EF tables in the same Postgres database. Max concurrency is 4. The dashboard is at `/tickerq`. Each job creates its own DI scope, so it doesn't reuse a request's DbContext.

**Cross-question:** What if the API restarts?

**Cross-answer:**

Missed tickers can be cancelled on restart (`CancelMissedTickersOnApplicationRestart`). Right now, document expiry schedules **another** daily job on every startup, so I would make that schedule idempotent.


---
## Additional Questions
### Q33. What is idempotency in an API?
**Answer:** If the same request is repeated, it should not cause the same business effect twice. For payments or wallet operations, I use a unique business key or idempotency key and enforce it in the database.

### Q34. Why is an outbox useful?
**Answer:** If I save a database change and publish an event separately, one can succeed while the other fails. The outbox stores the event in the same database transaction, and a worker publishes it later.

### Q35. How do you handle concurrency in a wallet?
**Answer:** I wrap the balance and ledger operation in a database transaction, with a lock or concurrency control around it. I also make the business transaction id unique, so retries can't double-charge.
