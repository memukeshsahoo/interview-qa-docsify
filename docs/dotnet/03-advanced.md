# .NET — Advanced

Architecture, tenants, speed, security. Still use short sentences.

---

## Q1. Clean architecture in a .NET solution?

**Answer:**

Domain: entities, no project references.

Application: use cases, interfaces. Depends on Domain only.

Infrastructure: EF, email. Implements those interfaces.

API: HTTP, auth, wires everything.

Arrows point inward. Controllers do not hide SQL. Domain does not know EF.

**Cross-question:** Where does `DbContext` live? Where do entities live?

**Cross-answer:**

Entities in Domain. `DbContext` in Infrastructure. Application talks through an interface or a port.

---

## Q2. Database per tenant vs one database plus `TenantId`?

**Answer:**

Per tenant: strong walls, harder ops.

Shared: easier ops, every query must filter, easier to leak.

**Cross-question:** If each tenant has a database, do entities need `TenantId`?

**Cross-answer:**

Usually no. The database *is* the tenant. I never trust a tenant id the browser sends. I resolve tenant from a claim or a known host map.

---

## Q3. What is IDOR?

**Answer:**

Knowing id `123` does not mean you may see it. I load the incident, then check tenant and permission. Else 404 or 403.

**Cross-question:** Are GUIDs enough?

**Cross-answer:**

They are harder to guess. They are **not** permission.

---

## Q4. Request path from Kestrel to your action?

**Answer:**

Kestrel reads HTTP → middleware → routing → authz → bind model → action → result → middleware on the way out.

**Cross-question:** Can I read the body twice?

**Cross-answer:**

Not unless I enable buffering. I prefer model binding. If I log bodies, I redact secrets.

---

## Q5. Thread pool starvation?

**Answer:**

Too many blocked threads: `.Result`, `Wait()`, `Thread.Sleep`, sync IO. Then nothing is left to finish the tasks those threads wait on. The site feels dead.

**Cross-question:** How do you spot it?

**Cross-answer:**

Counters, traces, sudden latency. I search for sync-over-async and huge EF loads on the request.

---

## Q6. `ExecuteUpdate` vs loading entities?

**Answer:**

`ExecuteUpdate` writes SQL without tracking each row. Fast for bulk. You skip per-entity events. If you need domain events, load and save the normal way.

**Cross-question:** Raw SQL?

**Cross-answer:**

I use LINQ/EF APIs. I do not paste SQL strings in app code when the team forbids it.

---

## Q7. MediatR / CQRS — when?

**Answer:**

CQRS: split read and write. MediatR: in-process messages and pipelines. Nice in a large app. Too much for five CRUD endpoints.

**Cross-question:** Is CQRS event sourcing?

**Cross-answer:**

No. Event sourcing stores events as the source of truth. Most teams never need that.

---

## Q8. Outbox pattern?

**Answer:**

Save the business row and a “please publish this” row in the **same** database transaction. A worker publishes later. You do not get “saved but message lost”.

**Cross-question:** Publish to the queue then save EF?

**Cross-answer:**

If save fails, you already told the world a lie. Dual write without an outbox is a classic incident.

---

## Q9. Idempotency keys?

**Answer:**

A client sends a key with a POST. I store it per tenant. Retrying the same key does not create a second payment.

**Cross-question:** Are all HTTP methods safe to retry?

**Cross-answer:**

GET/PUT/DELETE are meant to be. POST is not, unless I make it so. Timeout + retry on POST is a double-charge bug.

---

## Q10. Data Protection keys on more than one server?

**Answer:**

Cookies and some tokens are encrypted with a key ring. All servers must share those keys or users bounce and get logged out.

**Cross-question:** Public folder for the key ring?

**Cross-answer:**

No. Keys are secrets.

---

## Q11. How do you pick the tenant database?

**Answer:**

A factory reads the current tenant (claim or validated host) and opens `tenant_{slug}` from a catalog. If tenant is missing, we do not query.

**Cross-question:** Host header as tenant — safe?

**Cross-answer:**

Only if I map host → tenant in a trusted list. I do not stick the raw host into a connection string. The client never sends a connection string.

---

## Q12. Logs, metrics, traces?

**Answer:**

Logs: events. Metrics: numbers like request time. Traces: the path through API → EF → HTTP. I join them with a trace id. I do not log every SQL in production.

**Cross-question:** Why not log all SQL?

**Cross-answer:**

Volume, personal data in parameters, extra cost. Slow-query logging is enough.

---

## Q13. NativeAOT vs JIT?

**Answer:**

JIT compiles at run time. AOT compiles ahead: faster start, stricter (less reflection). Most business APIs still JIT. Know that old reflection tricks can break AOT.

**Cross-question:** What breaks first on AOT?

**Cross-answer:**

Dynamic `Assembly.Load`, some JSON without source gen, emit.

---

## Q14. Permission design?

**Answer:**

Stable codes like `incidents.read`. Roles are bundles. Object checks still happen on the server. The UI only hides buttons.

**Cross-question:** Permissions in the JWT or look up each time?

**Cross-answer:**

JWT is faster and can be stale. Database is fresh and heavier. Hybrid: role in token, dangerous actions checked again.

---

## Q15. Slow endpoint playbook?

**Answer:**

Reproduce. Measure: network, CPU, SQL, JSON? Count SQL. Fix N+1 or over-select. Measure again.

**Cross-question:** Cache first?

**Cross-answer:**

Not if the query is wrong or the key misses tenant. Fix the query first.

---

## Q16. gRPC vs REST vs GraphQL?

**Answer:**

REST: easy for browsers. gRPC: fast service-to-service, protobuf. GraphQL: flexible queries, more moving parts, auth per field.

**Cross-question:** Replace REST with gRPC for Angular?

**Cross-answer:**

Usually no. Browser tools, files, and HTTP status still fit REST/JSON.

---

## Q17. What is a captive dependency?

**Answer:**

A singleton holds a scoped service. That scoped service becomes a singleton by accident. `DbContext` shared across requests is a famous case. The DI container can warn you.

**Cross-question:** How do you use a scoped service inside a singleton?

**Cross-answer:**

Inject `IServiceScopeFactory`, create a scope per operation, dispose it.

---

## Q18. YARP / reverse proxy — why in front of Kestrel?

**Answer:**

TLS, path routing, more than one app, rate limits. Kestrel still runs the app. I do not expose Kestrel raw on the internet without a plan.

**Cross-question:** Forwarded headers?

**Cross-answer:**

So the app knows HTTPS and the real client IP. I only trust proxies I own, not any `X-Forwarded-*` from the world.

---

## Q19. How do you keep secrets out of logs and errors?

**Answer:**

No tokens in URLs. No connection strings in 500 responses. Exception middleware sends a generic message. I log ids, not payloads with passwords.

**Cross-question:** Dev vs prod exception pages?

**Cross-answer:**

Developer exception page is local only. Prod uses ProblemDetails without stack traces.

---

## Q20. EF compiled models / precompiled queries — when?

**Answer:**

Large models take time to start. Compiled models help startup. I mention it if the app has many entities and slow boot. Not the first fix for a slow page.

**Cross-question:** Does this replace indexing?

**Cross-answer:**

No. Indexes are a database thing. Different problem.

---

## Q21. How would you design a multi-tenant “current user” service?

**Answer:**

A scoped `IUserContext` filled from claims after auth. It has UserId, Tenant, permissions. Fail closed if empty on a protected request. No static `CurrentUser` on a singleton.

**Cross-question:** Async and user context?

**Cross-answer:**

`AsyncLocal` can work but is easy to leak across requests if you get it wrong. Scoped + HttpContext is clearer for web.

---

## Q22. What is backpressure?

**Answer:**

If work arrives faster than we can do it, we must slow the producer or drop with a plan. Bounded channels, 429, queue limits. Infinite in-memory queues will take the process down.

**Cross-question:** Hangfire queue growing forever?

**Cross-answer:**

I alert on queue length and failed jobs. I do not only add more servers with no limit.

---

## Q23. How do you version a database in a tenant-per-db world?

**Answer:**

Same migrations on every tenant database. A catalog tracks version. Rollout is a plan: one tenant, then batches. I do not invent a special schema per customer.

**Cross-question:** One tenant needs a custom column?

**Cross-answer:**

That is a product smell. Prefer a safe extension (JSON metadata) over a snowflake schema, if the project already has that pattern.

---

## Q24. What is the difference between authentication schemes?

**Answer:**

Cookies, JWT bearer, maybe API keys. One app can have more than one. A browser site might use cookies. A SPA might use bearer. I do not mix them by accident on the same endpoint without thinking.

**Cross-question:** `[Authorize(AuthenticationSchemes = "...")]`?

**Cross-answer:**

Picks which scheme that endpoint accepts. Default scheme is not always what you think if you registered two.

---

## Q25. How do you test multi-tenant isolation?

**Answer:**

Two users, two tenants. User A’s token must not read B’s id. Automated test. Also try a guessed id. This is not optional.

**Cross-question:** Is a manual click enough?

**Cross-answer:**

No. Someone will forget on the next feature. Tests stay.
