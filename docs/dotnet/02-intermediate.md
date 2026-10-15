# .NET — Intermediate

EF, auth, cache, real API habits.

---

## Q1. What is EF change tracking?

**Answer:**

When I load an entity, EF remembers its original values. On `SaveChanges` it writes only what changed. `AsNoTracking()` skips that for read-only lists, which means less memory and faster queries.

**Cross-question:** `Update()` vs changing a tracked property?

**Cross-answer:**

If the entity is already tracked, I just set the property. `Update()` marks **all** fields as dirty, so it can overwrite columns I did not mean to touch.

---

## Q2. What is an N+1 query?

**Answer:**

Say I load 50 parents, and then for each parent I run a query for its children. That is 51 trips to the database. The fix is `Include`, or better, a `Select` into a DTO. Sometimes I use split queries if a join blows up the number of rows.

**Cross-question:** Can `Include` be worse than N+1?

**Cross-answer:**

Yes. Including two collections in one query can duplicate the parent rows a lot. Then split queries, or two simple queries, can be cheaper. I look at the SQL to decide.

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

Things like my own C# methods, some date tricks, or loops inside `Where`. EF throws an error for those. I rewrite the filter, or I filter in memory after SQL has already cut down the rows.

**Cross-question:** How do you see the SQL?

**Cross-answer:**

With logging in **dev**, or with `ToQueryString()`. I don't log parameter values that are secrets.

---

## Q5. How does JWT auth work on the API?

**Answer:**

The client sends `Authorization: Bearer ...`. The middleware checks the signature, issuer, audience, and expiry. The claims become `User`. Then `[Authorize]` runs.

**Cross-question:** Can I log the token?

**Cross-answer:**

Never. I can log the user id. The secret lives in user-secrets or a vault, not in git.

---

## Q6. Roles vs policies vs resource checks?

**Answer:**

A role is something like “Admin”. A policy is a named rule. A resource check asks “can this user edit **this** incident?”. That last one needs the actual record, not only the role.

**Cross-question:** Why are roles not enough?

**Cross-answer:**

A user can be an admin in tenant A and nobody in tenant B. We also need checks on the specific object.

---

## Q7. Filters vs middleware?

**Answer:**

Middleware runs for every request. Filters run only around MVC actions, and they can see model state and arguments.

**Cross-question:** Global exception middleware or exception filter?

**Cross-answer:**

I like middleware, because it gives one consistent error JSON. Filters miss some paths.

---

## Q8. ProblemDetails?

**Answer:**

It is a standard error JSON with title, status, and detail. It is fine for 400s. The detail should never be a stack trace or a SQL message.

**Cross-question:** Domain rule fail vs a bug?

**Cross-answer:**

A rule failure gets a 400/409 with a stable code. A bug gets a 500: I log it on the server and send generic text to the client.

---

## Q9. Caching?

**Answer:**

Memory cache lives in this process only. Distributed cache (Redis) is shared by all servers. HTTP caching uses headers so the browser or a proxy can cache.

**Cross-question:** Multi-tenant cache key?

**Cross-answer:**

The key must include the tenant id. A cache without the tenant in the key is a data leak.

---

## Q10. Why `IHttpClientFactory`?

**Answer:**

It manages the handlers for me, so we don't run out of sockets and DNS doesn't go stale. Typed clients are clean, like `GitHubClient(HttpClient http)`.

**Cross-question:** Typed vs named client?

**Cross-answer:**

Typed is nicer when it's for one API. Named is `CreateClient("github")`.

---

## Q11. Transactions and two `SaveChanges`?

**Answer:**

One `SaveChanges` is usually one transaction. With two saves, if the second one fails, the data can be left half-written. I prefer one save at the end, or an explicit transaction around both.

**Cross-question:** Background job plus request in one transaction?

**Cross-answer:**

Usually no, because they run in different scopes. If both must happen together, I use an outbox.

---

## Q12. Health checks?

**Answer:**

A `/health` endpoint can ping the database. Orchestrators use it.

**Cross-question:** Liveness vs readiness?

**Cross-answer:**

Liveness asks: is the process stuck? If yes, restart it. Readiness asks: can it take traffic? An app that is still starting can be alive but not ready.

---

## Q13. SignalR in simple words?

**Answer:**

Live messages over WebSockets. The server has a hub, and it calls methods on the client. With more than one server you need a backplane like Redis, or messages get lost between nodes.

**Cross-question:** How do you auth a SignalR connection?

**Cross-answer:**

It is the same user as for HTTP. I authorize hub methods and never broadcast another tenant’s data. I treat the token as a secret, even if it ends up in a query string for the browser.

---

## Q14. API versioning?

**Answer:**

Either `/v1/...` in the URL or a header. Adding a field is usually ok. Renaming a field or changing its meaning needs a new version.

**Cross-question:** `int` to `string` — breaking?

**Cross-answer:**

Yes. Clients will break.

---

## Q15. Soft delete vs hard delete?

**Answer:**

Soft delete means an `IsDeleted` flag, and a global filter hides those rows. History stays, but things like unique emails get messy. Hard delete means the row is really gone. I pick per table.

**Cross-question:** `IgnoreQueryFilters()`?

**Cross-answer:**

Only for admin screens or jobs that must see deleted rows. Never on a public get-by-id without extra checks.

---

## Q16. What is the options pattern validation?

**Answer:**

I bind config to a class and fail at startup if required values are missing. That is much better than a null reference at noon on Monday.

**Cross-question:** Can options hold a secret?

**Cross-answer:**

The object can, but I still don't log it. I load it from a secret store.

---

## Q17. `AsNoTrackingWithIdentityResolution` — in simple words?

**Answer:**

No tracking, but if the same person appears twice in the result graph, I get one object, not two copies. It is useful when reading object graphs.

**Cross-question:** Default `AsNoTracking`?

**Cross-answer:**

With that, the same key can turn into two separate instances. That can surprise you when you compare references.

---

## Q18. Pagination — skip/take vs keyset?

**Answer:**

`Skip/Take` is easy, but it gets slow on big offsets. Keyset (“where id > lastId take 20”) stays fast. For admin pages, Skip is often fine.

**Cross-question:** Should the API return the total count always?

**Cross-answer:**

A count can be expensive. I return it when the UI needs it, not out of habit.

---

## Q19. File upload in ASP.NET Core — what do you check?

**Answer:**

Size, type, and extension. I don't trust the content type the client sends. I store the file under a generated name, not the user's file name. And no `../` in paths.

**Cross-question:** Save inside `wwwroot`?

**Cross-answer:**

Only if it must be public. Private files go outside the web root, and users get them through a download action that checks permission.

---

## Q20. What is CORS preflight?

**Answer:**

For some cross-origin calls, the browser first sends an `OPTIONS` request. The API must answer with the right allow headers. If that fails, the real POST from JS never runs.

**Cross-question:** Why does Swagger work but Angular does not?

**Cross-answer:**

Swagger is often on the same origin. Angular on another port counts as another origin.

---

## Q21. Cookie auth vs JWT?

**Answer:**

With cookies, the browser sends them on its own, so CSRF is a concern. With a JWT in a header, Angular attaches it, so CSRF is less of a risk, but XSS can steal a token kept in storage. Both need HTTPS.

**Cross-question:** JWT in localStorage?

**Cross-answer:**

It is common, and risky if you have XSS. An HttpOnly cookie is the other approach. I never put the token in the URL if I can help it.

---

## Q22. What does `SaveChanges` return?

**Answer:**

The number of state entries written to the database. I don't use that as my only success check for business rules.

**Cross-question:** 0 rows updated — success?

**Cross-answer:**

Maybe the id was wrong. I check that the entity existed and that the user was allowed to change it.

---

## Q23. Global query filters?

**Answer:**

It is a default `Where` added to every query, like `!IsDeleted`. The catch is that it is easy to forget about when you really do need the hidden rows.

**Cross-question:** Filter by tenant in a shared database?

**Cross-answer:**

Yes, a filter on `TenantId` helps, but I still don't trust the tenant id the client sends. I set the tenant from the server context.

---

## Q24. How do you handle concurrency (two users edit the same row)?

**Answer:**

I use a row version (`xmin` / timestamp). The second save gets a concurrency exception, and I show “someone else changed this, please reload”.

**Cross-question:** Last write wins?

**Cross-answer:**

That's fine for some fields, but bad for money and status. I pick per feature.

---

## Q25. What is compiled query in EF?

**Answer:**

EF caches the translation of a query you run a lot. It helps on hot paths. I don't compile every random LINQ line.

**Cross-question:** First call still slow?

**Cross-answer:**

Yes. You pay the compile cost once. After that it is cheaper.

---

## Q26. Rate limiting?

**Answer:**

It stops one client from flooding the API. ASP.NET has middleware for this. I pair it with auth, so a public login endpoint cannot be guessed at all day.

**Cross-question:** 429 meaning?

**Cross-answer:**

Too many requests, retry later. Clients should back off.

---

## Q27. How does soft delete work in NriCare?

**Answer:**

Entities implement `IBaseEntity`, which has `IsDeleted`. In `SaveChanges`, a hard delete is turned into an update that sets `IsDeleted = true`. A global query filter hides those rows. The audit fields `CreatedBy` and `UpdatedBy` come from `IUserSession`.

**Cross-question:** How do you still load a deleted wallet?

**Cross-answer:**

I use `IgnoreQueryFilters()` plus `!x.IsDeleted` when I really mean “include filtered types but still skip deleted”. Booking payout does this for wallets across associations.

---

## Q28. How do you stop one association seeing another association’s data?

**Answer:**

It is not a separate database. Entities like Booking, NRUser, and ServiceProvider implement `IMainAssociationEntity`. The global filter is `MainAssociationId == current association`. SuperAdmin sets `DisableMainAssociationFilter`. On insert, SaveChanges stamps the association id from the session.

**Cross-question:** Can the Angular app send `MainAssociationId` and switch tenant?

**Cross-answer:**

It should not win. The server session is the source of truth. SuperAdmin is the only role that turns the filter off.

---

## Q29. JWT claims in NriCare — what is inside?

**Answer:**

Custom claims, not ASP.NET Identity roles: `user_id`, `session_id`, `user_access_type`, plus ids for service provider, association, NR user, and verifier. The access token lives for about 20 minutes. `ClockSkew` is zero, so expiry is strict.

**Cross-question:** `[Authorize(Roles = "Admin")]`?

**Cross-answer:**

We don't use that. We put `[Authorize]` on the controller, and then the code checks `UserAccessType`. If the role matrix grew, I would add policies.

---

## Q30. How do you paginate lists?

**Answer:**

We have an extension, `ApplyPaginationAsync`, on `IQueryable`. Search, date, and skip/take stay on the query, so EF turns them into SQL. Wallet transactions and bookings use this. I don't `ToList` the whole table and then page in memory.

**Cross-question:** Why `CreatedDate.Date` in some queries?

**Cross-answer:**

It is easy to read, but it can stop the database from using an index because of the `.Date` conversion. For hot paths I would compare against a UTC start/end instead.

---

## Q31. How do you map entities to DTOs?

**Answer:**

Most list APIs use a LINQ `Select` into a DTO, so EF does the projection in SQL. Mapster is registered, but we barely use it. I prefer `Select` for lists so we don't load full graphs.

**Cross-question:** Returning the Booking entity?

**Cross-answer:**

No. It has the fee breakdown, holds, chat, and histories. That is a huge JSON and it can loop. DTO only.

---

## Q32. File uploads — where do files go?

**Answer:**

To DigitalOcean Spaces, through the AWS S3 SDK. `FileUploadHelper` stores public or private objects. Images can get a thumbnail. Private files are served with a short-lived presigned URL. The `File` row in Postgres holds the metadata.

**Cross-question:** If S3 succeeds and the DB insert fails?

**Cross-answer:**

You can end up with an orphan object in Spaces. I log that. A cleanup job or delete-on-failure would be the improvement.

---

## Q33. How does OTP work in your signup?

**Answer:**

There are anonymous endpoints for request, verify, and resend. We store a 6-digit code in `OtpVerification` with a 5-minute expiry. Login itself is phone + password, not OTP. OTP is only for signup verification.

**Cross-question:** How is the SMS sent?

**Cross-answer:**

A real SMS provider is not wired up yet. Right now the verify API returns the OTP in the message for testing. I would not ship that. Production needs SMS/email and no OTP in the response.

---

## Q34. Why is almost every repository scoped?

**Answer:**

They need `DbContext` and `IUserSession`, which are per request. If I made `BookingRepository` a singleton, it would hold on to one context and one user for the whole app.

**Cross-question:** Background job then?

**Cross-answer:**

TickerQ jobs create a **new scope**, get `ApplicationDbContext` from that scope, and then dispose it. Same rule as with Hangfire.


---
## Additional Questions
### Q26. Tracking vs AsNoTracking?
**Answer:** Tracking is useful when I'm going to update the entity. For read-only lists, AsNoTracking cuts the tracking overhead and memory.

### Q27. How do you avoid an N+1 query?
**Answer:** First I check the SQL. Usually I project straight into a DTO with Select, or I use Include carefully. I avoid querying children inside a loop.

### Q28. When would you use keyset pagination?
**Answer:** For very large tables where Skip gets expensive. Instead of skipping thousands of rows, I ask for the records after the last id or timestamp I've seen.
