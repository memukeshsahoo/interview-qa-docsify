# .NET — Intermediate

EF, auth, cache, real API habits.

---

## Q1. What is EF change tracking?

**Answer:**

When I load an entity, EF remembers the original values. On `SaveChanges` it writes only what changed. `AsNoTracking()` skips that for read-only lists. Less memory, faster.

**Cross-question:** `Update()` vs changing a tracked property?

**Cross-answer:**

If it is already tracked, just set the property. `Update()` marks **all** fields as dirty. That can overwrite columns you did not mean to touch.

---

## Q2. What is an N+1 query?

**Answer:**

I load 50 parents, then for each parent I query children — 51 trips. Fix: `Include`, or better a `Select` into a DTO. Sometimes split queries if a join explodes rows.

**Cross-question:** Can `Include` be worse than N+1?

**Cross-answer:**

Yes. Two collections on one Include can duplicate parent rows a lot. Then split queries or two simple queries can be cheaper. I look at the SQL.

---

## Q3. Who owns migrations?

**Answer:**

On many teams, people generate and apply migrations. I can change entities and Fluent config. I do not run schema updates unless that is the agreed process.

**Cross-question:** `EnsureCreated` in production?

**Cross-answer:**

No. It is for samples and throwaway tests. Real apps use migrations.

---

## Q4. LINQ EF cannot translate?

**Answer:**

My own C# methods, some date tricks, loops inside `Where`. EF throws. I rewrite the filter, or I filter in memory after SQL already reduced the rows.

**Cross-question:** How do you see the SQL?

**Cross-answer:**

Logging in **dev**, or `ToQueryString()`. I do not log parameter values that are secrets.

---

## Q5. How does JWT auth work on the API?

**Answer:**

The client sends `Authorization: Bearer ...`. Middleware checks signature, issuer, audience, expiry. Claims become `User`. Then `[Authorize]` runs.

**Cross-question:** Can I log the token?

**Cross-answer:**

Never. I can log user id. The secret lives in user-secrets or a vault, not in git.

---

## Q6. Roles vs policies vs resource checks?

**Answer:**

Roles: “Admin”. Policies: named rules. Resource: “can this user edit **this** incident?” That last one needs the record, not only the role.

**Cross-question:** Why are roles not enough?

**Cross-answer:**

A user can be admin in tenant A and nobody in tenant B. We also need object-level checks.

---

## Q7. Filters vs middleware?

**Answer:**

Middleware: every request. Filters: MVC actions, they see model state and arguments.

**Cross-question:** Global exception middleware or exception filter?

**Cross-answer:**

I like middleware for one consistent error JSON. Filters miss some paths.

---

## Q8. ProblemDetails?

**Answer:**

A standard error JSON: title, status, detail. Fine for 400s. Detail should not be a stack trace or a SQL message.

**Cross-question:** Domain rule fail vs a bug?

**Cross-answer:**

Rule: 400/409 with a stable code. Bug: 500, log on the server, generic text to the client.

---

## Q9. Caching?

**Answer:**

Memory cache: this process only. Distributed (Redis): all servers. HTTP cache: browser/proxy headers.

**Cross-question:** Multi-tenant cache key?

**Cross-answer:**

Must include tenant id. A cache without tenant is a data leak.

---

## Q10. Why `IHttpClientFactory`?

**Answer:**

It manages handlers so we do not run out of sockets, and DNS does not go stale. Typed clients are clean: `GitHubClient(HttpClient http)`.

**Cross-question:** Typed vs named client?

**Cross-answer:**

Typed is nicer for one API. Named is `CreateClient("github")`.

---

## Q11. Transactions and two `SaveChanges`?

**Answer:**

One `SaveChanges` is usually one transaction. Two saves can leave a half-written state if the second fails. I prefer one save at the end, or an explicit transaction around both.

**Cross-question:** Background job plus request in one transaction?

**Cross-answer:**

Usually no. Different scopes. Use an outbox if both must happen together.

---

## Q12. Health checks?

**Answer:**

`/health` can ping the database. Orchestrators use them.

**Cross-question:** Liveness vs readiness?

**Cross-answer:**

Liveness: is the process stuck? Restart it. Readiness: can it take traffic? A starting app can be alive but not ready.

---

## Q13. SignalR in simple words?

**Answer:**

Live messages over WebSockets. Server hub, client methods. More than one server needs a backplane like Redis or messages get lost between nodes.

**Cross-question:** How do you auth a SignalR connection?

**Cross-answer:**

Same user as HTTP. Authorize hub methods. Do not broadcast another tenant’s data. Treat the token as secret even if it ends up in a query string for the browser.

---

## Q14. API versioning?

**Answer:**

`/v1/...` or a header. Adding a field is usually ok. Renaming or changing meaning needs a new version.

**Cross-question:** `int` to `string` — breaking?

**Cross-answer:**

Yes. Clients will break.

---

## Q15. Soft delete vs hard delete?

**Answer:**

Soft: `IsDeleted` flag, hide with a global filter. History stays. Unique emails get messy. Hard: the row is gone. I pick per table.

**Cross-question:** `IgnoreQueryFilters()`?

**Cross-answer:**

Only for admin/jobs that must see deleted rows. Never on a public get-by-id without extra checks.

---

## Q16. What is the options pattern validation?

**Answer:**

Bind config to a class and fail at startup if required values are missing. Better than a null reference at noon on Monday.

**Cross-question:** Can options hold a secret?

**Cross-answer:**

The object can, but I still do not log it. Load from a secret store.

---

## Q17. `AsNoTrackingWithIdentityResolution` — in simple words?

**Answer:**

No tracking, but if the same person appears twice in the graph, I get one object, not two copies. Useful for read graphs.

**Cross-question:** Default `AsNoTracking`?

**Cross-answer:**

Same key can become two instances. That can surprise you when you compare references.

---

## Q18. Pagination — skip/take vs keyset?

**Answer:**

`Skip/Take` is easy and gets slow on big offsets. Keyset (“where id > lastId take 20”) stays fast. For admin pages Skip is often fine.

**Cross-question:** Should the API return the total count always?

**Cross-answer:**

Count can be expensive. I return it when the UI needs it, not by habit.

---

## Q19. File upload in ASP.NET Core — what do you check?

**Answer:**

Size, type, extension. I do not trust the client content type. I store under a generated name, not the user file name. No `../` in paths.

**Cross-question:** Save inside `wwwroot`?

**Cross-answer:**

Only if it must be public. Private files go outside the web root and through a download action that checks permission.

---

## Q20. What is CORS preflight?

**Answer:**

For some cross-origin calls the browser first sends `OPTIONS`. The API must answer with the right allow headers. If that fails, the real POST never runs from JS.

**Cross-question:** Why does Swagger work but Angular does not?

**Cross-answer:**

Swagger is often same origin. Angular on another port is another origin.

---

## Q21. Cookie auth vs JWT?

**Answer:**

Cookies: browser sends them itself, CSRF is a concern. JWT in a header: Angular attaches it, CSRF is weaker, XSS can steal a token in storage. Both need HTTPS.

**Cross-question:** JWT in localStorage?

**Cross-answer:**

Common and risky if you have XSS. HttpOnly cookie is another style. I never put the token in the URL if I can help it.

---

## Q22. What does `SaveChanges` return?

**Answer:**

The number of state entries written. I do not use that as my only success check for business rules.

**Cross-question:** 0 rows updated — success?

**Cross-answer:**

Maybe the id was wrong. I check that the entity existed and the user was allowed.

---

## Q23. Global query filters?

**Answer:**

A default `Where` on every query, like `!IsDeleted`. Easy to forget when you really need the hidden rows.

**Cross-question:** Filter by tenant in a shared database?

**Cross-answer:**

Yes, a filter on `TenantId` helps, but I still do not trust the client’s tenant id. I set tenant from the server context.

---

## Q24. How do you handle concurrency (two users edit the same row)?

**Answer:**

A row version (`xmin` / timestamp). Second save gets a concurrency exception. I show “someone else changed this, reload”.

**Cross-question:** Last write wins?

**Cross-answer:**

Fine for some fields. Bad for money and status. I pick per feature.

---

## Q25. What is compiled query in EF?

**Answer:**

EF caches the translation of a query you use a lot. Helps hot paths. I do not compile every random LINQ line.

**Cross-question:** First call still slow?

**Cross-answer:**

Yes. Compile pays once. After that it is cheaper.

---

## Q26. Rate limiting?

**Answer:**

Stop one client flooding the API. ASP.NET has middleware for this. Pair it with auth so a public login endpoint cannot be guessed all day.

**Cross-question:** 429 meaning?

**Cross-answer:**

Too many requests. Retry later. Clients should back off.

---

## Q27. How does soft delete work in NriCare?

**Answer:**

Entities implement `IBaseEntity` with `IsDeleted`. In `SaveChanges`, a hard delete is turned into a modify: `IsDeleted = true`. A global query filter hides those rows. Audit fields `CreatedBy` and `UpdatedBy` come from `IUserSession`.

**Cross-question:** How do you still load a deleted wallet?

**Cross-answer:**

`IgnoreQueryFilters()` plus `!x.IsDeleted` when I really mean “include filtered types but still skip deleted”. Booking payout uses that for wallets across associations.

---

## Q28. How do you stop one association seeing another association’s data?

**Answer:**

Not a separate database. Entities like Booking, NRUser, ServiceProvider implement `IMainAssociationEntity`. The global filter is `MainAssociationId == current association`. SuperAdmin sets `DisableMainAssociationFilter`. On insert, SaveChanges stamps the association id from the session.

**Cross-question:** Can the Angular app send `MainAssociationId` and switch tenant?

**Cross-answer:**

It should not win. The server session is the source. SuperAdmin is the only role that turns the filter off.

---

## Q29. JWT claims in NriCare — what is inside?

**Answer:**

Custom claims, not ASP.NET Identity roles: `user_id`, `session_id`, `user_access_type`, plus ids for service provider, association, NR user, and verifier. Access token life is about 20 minutes. `ClockSkew` is zero, so expiry is strict.

**Cross-question:** `[Authorize(Roles = "Admin")]`?

**Cross-answer:**

We do not use that. `[Authorize]` on the controller, then code checks `UserAccessType`. I would add policies if the role matrix grew.

---

## Q30. How do you paginate lists?

**Answer:**

An extension `ApplyPaginationAsync` on `IQueryable`. Search, date, skip/take stay on the query so EF turns them into SQL. Wallet transactions and bookings use this. I do not `ToList` the whole table then page in memory.

**Cross-question:** Why `CreatedDate.Date` in some queries?

**Cross-answer:**

It is easy to read and can block an index because of the `.Date` conversion. For hot paths I would compare to a UTC start/end instead.

---

## Q31. How do you map entities to DTOs?

**Answer:**

Most list APIs use LINQ `Select` into a DTO so EF projects in SQL. Mapster is registered, but we barely use it. I prefer `Select` for lists so we do not load full graphs.

**Cross-question:** Returning the Booking entity?

**Cross-answer:**

No. It has fee breakdown, holds, chat, histories. That is a huge JSON and can loop. DTO only.

---

## Q32. File uploads — where do files go?

**Answer:**

DigitalOcean Spaces through the AWS S3 SDK. `FileUploadHelper` stores public or private objects. Images can get a thumbnail. Private files use a short presigned URL. The `File` row in Postgres is the metadata.

**Cross-question:** If S3 succeeds and the DB insert fails?

**Cross-answer:**

You can get an orphan object in Spaces. I log that. A cleanup job or delete-on-failure would be the improvement.

---

## Q33. How does OTP work in your signup?

**Answer:**

Anonymous endpoints: request, verify, resend. We store a 6-digit code in `OtpVerification` with a 5-minute expiry. Login itself is phone + password, not OTP. OTP is for signup verification.

**Cross-question:** How is the SMS sent?

**Cross-answer:**

A real SMS provider is not wired yet. The verify API currently returns the OTP in the message for testing. I would not ship that. Production needs SMS/email and no OTP in the response.

---

## Q34. Why is almost every repository scoped?

**Answer:**

They need `DbContext` and `IUserSession`, which are per request. If I put `BookingRepository` in a singleton, it would capture one context and one user for the whole app.

**Cross-question:** Background job then?

**Cross-answer:**

TickerQ jobs create a **new scope**, resolve `ApplicationDbContext` from that scope, then dispose it. Same rule as Hangfire.


---
## Additional Questions
### Q26. Tracking vs AsNoTracking?
**Answer:** Tracking is useful when I will update the entity. For read-only lists, AsNoTracking reduces tracking overhead and memory.

### Q27. How do you avoid an N+1 query?
**Answer:** I first check the SQL. Usually I project directly to a DTO with Select, or use Include carefully. I avoid querying children inside a loop.

### Q28. When would you use keyset pagination?
**Answer:** For very large tables where Skip becomes expensive. Instead of skipping thousands of rows, I ask for records after the last seen id or timestamp.
