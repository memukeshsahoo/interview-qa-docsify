# .NET — Basic

Web APIs, DI, HTTP. Say it like you built a controller last week.

---

## Q1. What is ASP.NET Core?

**Answer:**

It is the web framework on .NET. I use it for HTTP APIs, MVC, SignalR, and so on. The host wires logging, config, DI, and a pipeline of middleware.

**Cross-question:** Controllers vs minimal APIs?

**Cross-answer:**

Controllers group actions with attributes. Minimal APIs map routes in `Program.cs` with less code. Both work. Bigger apps often keep controllers so files stay tidy.

---

## Q2. What is middleware?

**Answer:**

A chain. Each piece can run code, call the next piece, or stop. Order matters: errors first, then HTTPS, then auth, then authorization, then your endpoints.

**Cross-question:** `Use` vs `Run` vs `Map`?

**Cross-answer:**

`Use` can call next. `Run` is the end. `Map` branches by path.

---

## Q3. Dependency Injection in ASP.NET Core?

**Answer:**

My class does not `new` its helpers. The app creates them and passes them in, usually in the constructor.

**Cross-question:** Transient, scoped, singleton?

**Cross-answer:**

Transient: new every time. Scoped: one per HTTP request. Singleton: one for the whole app.

Never put a scoped thing (like `DbContext`) into a singleton. That scoped object would live forever and get shared across users.

---

## Q4. What is `DbContext`? Why scoped?

**Answer:**

It is EF’s unit of work. It tracks entities. It is not safe across threads. One per request is the default.

**Cross-question:** Hangfire / background job needs EF?

**Cross-answer:**

Open a **new scope**, resolve `DbContext` from that scope, then dispose the scope. Do not reuse the request context on another thread.

---

## Q5. GET vs POST vs PUT vs PATCH vs DELETE?

**Answer:**

GET reads. It should not change data. POST creates or does an action. PUT replaces the whole thing. PATCH updates part. DELETE removes.

**Cross-question:** Why must GET not change data?

**Cross-answer:**

Browsers, caches, and retries treat GET as safe. Side effects belong on POST/PUT/PATCH. Also never put secrets in the query string.

---

## Q6. Model binding and validation?

**Answer:**

ASP.NET maps URL, query, and body into parameters. `[Required]` or FluentValidation check the data. With `[ApiController]`, bad input becomes 400.

**Cross-question:** Angular validation vs server validation?

**Cross-answer:**

The UI is for the user. The **server** is for safety. Always check again on the API.

---

## Q7. What is a DTO? Why not return the EF entity?

**Answer:**

A DTO is the shape I send to the client. Entities have extra fields, navigation properties, and can loop in JSON. I map to a DTO.

**Cross-question:** AutoMapper or hand mapping?

**Cross-answer:**

Hand `Select` is clear and EF can turn it into SQL. AutoMapper is faster to type. Either is fine if I do not leak entities.

---

## Q8. Where does config live?

**Answer:**

`appsettings.json`, then environment file, then environment variables, then user secrets in dev. No production passwords in git.

**Cross-question:** Bind a section to a class?

**Cross-answer:**

`Configure<SmtpOptions>(...)` and inject `IOptions<SmtpOptions>`. I do not sprinkle `configuration["Smtp:Password"]` all over.

---

## Q9. Authentication vs authorization?

**Answer:**

Authentication: who are you?

Authorization: what are you allowed to do?

`UseAuthentication` must run before `UseAuthorization`.

**Cross-question:** Angular route guard vs `[Authorize]`?

**Cross-answer:**

A guard only hides a screen. Anyone can still call the API. The API must check.

---

## Q10. What is Kestrel?

**Answer:**

It is the web server inside ASP.NET Core. In production we often put Nginx or IIS in front for HTTPS and process management.

**Cross-question:** Can Kestrel do HTTPS itself?

**Cross-answer:**

Yes. Many teams still terminate HTTPS at the proxy. Then the app must trust forwarded headers so it knows the real scheme and client IP.

---

## Q11. Logging vs `Console.WriteLine`?

**Answer:**

Use `ILogger<T>`. Levels, templates, and sinks. I log `"User {UserId} logged in"`, not the token. Never log passwords or connection strings.

**Cross-question:** Why `{UserId}` not string concat?

**Cross-answer:**

The message stays the same. The id is a field I can search. Concat is easier to leak secrets into.

---

## Q12. BackgroundService / hosted service?

**Answer:**

Work that keeps running inside the app: a queue reader, a timer. `ExecuteAsync` gets a stopping token.

**Cross-question:** Hosted service vs Hangfire vs a Worker project?

**Cross-answer:**

Hosted: simple, dies with the app. Hangfire: saved jobs, retries, dashboard. Separate Worker: own process, own scale.

---

## Q13. Status codes you actually use?

**Answer:**

200 ok, 201 created, 204 no content, 400 bad input, 401 not logged in, 403 logged in but not allowed, 404, 409 conflict, 500 server error.

**Cross-question:** User is logged in but tries another tenant’s incident?

**Cross-answer:**

403, or 404 so we do not admit it exists. 401 means we do not know who you are.

---

## Q14. What is CORS?

**Answer:**

The browser blocks a page on origin A from reading origin B unless B allows it.

**Cross-question:** Does CORS protect a mobile app or Postman?

**Cross-answer:**

No. CORS is a browser rule. The API still needs login and permission. I do not turn on “allow everyone” just to make Angular work.

---

## Q15. What happens in `Program.cs`?

**Answer:**

Build the host, register services, build the app, set middleware, map endpoints, run. It is the composition root.

**Cross-question:** 200 lines of DI in Program?

**Cross-answer:**

I move them to `AddInfrastructure(config)` extension methods. Program stays short.

---

## Q16. What is routing?

**Answer:**

Matching a URL to an action, like `GET /api/incidents/5`. Attributes like `[HttpGet("{id}")]` or minimal API maps.

**Cross-question:** Conventional vs attribute routing?

**Cross-answer:**

APIs almost always use attributes. Conventional routes are more of an old MVC website style.

---

## Q17. What is `[ApiController]`?

**Answer:**

A marker that turns on API habits: automatic 400 on bad model, bind from body by default, problem details errors.

**Cross-question:** Can I skip it?

**Cross-answer:**

Yes, then I must handle `ModelState` myself. I keep it on API controllers.

---

## Q18. What is a filter in MVC?

**Answer:**

Code that runs around an action: auth, logging, wrapping exceptions. Different from middleware because it sees the action and arguments.

**Cross-question:** Middleware or filter for a correlation id?

**Cross-answer:**

Middleware. It should run for every request, not only controllers.

---

## Q19. How do you read the current user?

**Answer:**

`User` on the controller, or `IHttpContextAccessor` in a service. I read claims the token already has. I do not trust a `userId` the client puts in the JSON body as the only check.

**Cross-question:** Can I trust `TenantId` from the Angular request?

**Cross-answer:**

No. I take tenant from the host or from a claim I issued after login, then I check it server-side.

---

## Q20. What is EF Core in one line?

**Answer:**

It maps C# classes to tables so I query with LINQ instead of writing SQL strings in the app.

**Cross-question:** Can I still see the SQL?

**Cross-answer:**

Yes, with logging in development. I look at it when a query is slow.

---

## Q21. `app.MapControllers()` vs `MapGet`?

**Answer:**

`MapControllers` picks up your controller classes. `MapGet` is a single minimal endpoint. Same pipeline after that.

**Cross-question:** Can I mix them?

**Cross-answer:**

Yes. Some teams use minimal for health checks and controllers for the rest.

---

## Q22. What is HTTPS redirection?

**Answer:**

It sends HTTP to HTTPS so passwords are not sent in the clear. In production this should be on. Local http is a dev convenience, not a production setup.

**Cross-question:** Would you turn off HTTPS to fix a local Angular error?

**Cross-answer:**

No. I fix the URL or the certificate. I do not ship “HTTPS off”.

---

## Q23. What is a 500 vs a 400 in your API?

**Answer:**

400: the client sent something wrong. 500: we failed. I do not return a stack trace or a SQL error to the browser.

**Cross-question:** Unique key clash — 500 or 409?

**Cross-answer:**

409 or 400 with a clear message. It is not an unknown crash.

---

## Q24. What is `IOptions<T>`?

**Answer:**

A typed bag of settings. Better than string keys everywhere. I can validate at startup so the app fails fast if SMTP is missing.

**Cross-question:** Snapshot vs Monitor?

**Cross-answer:**

`IOptions` is fixed for the app. Snapshot can refresh per request. Monitor can notify when a file changes. Most settings are `IOptions`.

---

## Q25. How do you test a controller?

**Answer:**

I prefer testing the handler/service with a fake repository. For the HTTP surface I use `WebApplicationFactory` and call the URL. I do not hit a real production database.

**Cross-question:** Mock HttpContext?

**Cross-answer:**

If the action is fat, yes it gets messy. That is a sign to move logic out of the controller.
