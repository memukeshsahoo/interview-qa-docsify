# .NET — Basic

Web APIs, DI, HTTP. Say it like you built a controller last week.

---

## Q1. What is ASP.NET Core?

**Answer:**

It is the web framework on .NET. I use it for HTTP APIs, MVC, SignalR, and so on. The host sets up logging, config, DI, and a pipeline of middleware for me.

**Cross-question:** Controllers vs minimal APIs?

**Cross-answer:**

Controllers group actions in a class and use attributes. Minimal APIs map routes in `Program.cs` with less code. Both work. Bigger apps often keep controllers so the files stay tidy.

---

## Q2. What is middleware?

**Answer:**

Middleware is a chain of small pieces that every request passes through. Each piece can run some code, call the next piece, or stop the request there. Order matters: error handling first, then HTTPS, then authentication, then authorization, then your endpoints.

**Cross-question:** `Use` vs `Run` vs `Map`?

**Cross-answer:**

`Use` can call the next piece. `Run` is the end of the chain. `Map` branches the pipeline by path.

---

## Q3. Dependency Injection in ASP.NET Core?

**Answer:**

My class does not `new` its helpers. The app creates them and passes them in, usually through the constructor.

**Cross-question:** Transient, scoped, singleton?

**Cross-answer:**

Transient: a new one every time. Scoped: one per HTTP request. Singleton: one for the whole app.

I never put a scoped thing (like `DbContext`) inside a singleton. That scoped object would then live forever and get shared across users.

---

## Q4. What is `DbContext`? Why scoped?

**Answer:**

It is EF's unit of work, meaning it keeps track of the entities I load and change. It is not safe to use from many threads at once. So the default is one per request.

**Cross-question:** Hangfire / background job needs EF?

**Cross-answer:**

I open a **new scope**, get `DbContext` from that scope, and dispose the scope when I'm done. I do not reuse the request's context on another thread.

---

## Q5. GET vs POST vs PUT vs PATCH vs DELETE?

**Answer:**

GET reads, and it should not change data. POST creates something or runs an action. PUT replaces the whole thing. PATCH updates part of it. DELETE removes it.

**Cross-question:** Why must GET not change data?

**Cross-answer:**

Browsers, caches, and retries all treat GET as safe. Side effects belong on POST/PUT/PATCH. Also, I never put secrets in the query string.

---

## Q6. Model binding and validation?

**Answer:**

ASP.NET takes values from the URL, query, and body and maps them into my parameters. Then `[Required]` or FluentValidation checks the data. With `[ApiController]`, bad input turns into a 400 automatically.

**Cross-question:** Angular validation vs server validation?

**Cross-answer:**

The UI check is for the user's convenience. The **server** check is for safety. I always check again on the API.

---

## Q7. What is a DTO? Why not return the EF entity?

**Answer:**

A DTO is the shape of data I send to the client. Entities have extra fields and navigation properties, and they can loop forever in JSON. So I map to a DTO.

**Cross-question:** AutoMapper or hand mapping?

**Cross-answer:**

A hand-written `Select` is clear, and EF can turn it into SQL. AutoMapper is faster to type. Either is fine as long as I don't leak entities.

---

## Q8. Where does config live?

**Answer:**

`appsettings.json` first, then the environment file, then environment variables, then user secrets in dev. Production passwords never go in git.

**Cross-question:** Bind a section to a class?

**Cross-answer:**

I call `Configure<SmtpOptions>(...)` and inject `IOptions<SmtpOptions>`. I don't sprinkle `configuration["Smtp:Password"]` all over the code.

---

## Q9. Authentication vs authorization?

**Answer:**

Authentication: who are you?

Authorization: what are you allowed to do?

`UseAuthentication` must run before `UseAuthorization`.

**Cross-question:** Angular route guard vs `[Authorize]`?

**Cross-answer:**

A guard only hides a screen. Anyone can still call the API directly. So the API must do the check.

---

## Q10. What is Kestrel?

**Answer:**

It is the web server built into ASP.NET Core. In production we often put Nginx or IIS in front of it for HTTPS and process management.

**Cross-question:** Can Kestrel do HTTPS itself?

**Cross-answer:**

Yes. But many teams still end HTTPS at the proxy. In that case the app must trust forwarded headers, so it knows the real scheme and client IP.

---

## Q11. Logging vs `Console.WriteLine`?

**Answer:**

I use `ILogger<T>`. It gives me levels, message templates, and sinks (places the logs go). I log `"User {UserId} logged in"`, not the token. I never log passwords or connection strings.

**Cross-question:** Why `{UserId}` not string concat?

**Cross-answer:**

The message text stays the same, and the id becomes a field I can search on. With concat it is also easier to leak secrets into the message.

---

## Q12. BackgroundService / hosted service?

**Answer:**

It is work that keeps running inside the app, like a queue reader or a timer. `ExecuteAsync` gets a stopping token so it knows when the app is shutting down.

**Cross-question:** Hosted service vs Hangfire vs a Worker project?

**Cross-answer:**

Hosted service: simple, but it dies with the app. Hangfire: saved jobs, retries, and a dashboard. A separate Worker: its own process that scales on its own.

In NriCare I do not use Hangfire. Background work runs on **TickerQ**, stored in the same Postgres database, with a dashboard at `/tickerq`.

---

## Q13. Status codes you actually use?

**Answer:**

200 ok, 201 created, 204 no content, 400 bad input, 401 not logged in, 403 logged in but not allowed, 404 not found, 409 conflict, 500 server error.

**Cross-question:** User is logged in but tries another association’s booking?

**Cross-answer:**

403, or 404 so we don't even admit it exists. 401 is wrong here, because 401 means we don't know who you are.

---

## Q14. What is CORS?

**Answer:**

It is a browser rule. The browser blocks a page on origin A from reading responses from origin B, unless B allows it.

**Cross-question:** Does CORS protect a mobile app or Postman?

**Cross-answer:**

No. CORS is only a browser rule. The API still needs login and permission checks. I don't turn on “allow everyone” just to make Angular work.

---

## Q15. What happens in `Program.cs`?

**Answer:**

I build the host, register services, build the app, set up middleware, map endpoints, and run it. It is the composition root, the one place where everything gets wired together.

**Cross-question:** 200 lines of DI in Program?

**Cross-answer:**

I move them into extension methods like `AddInfrastructure(config)`. That keeps Program short.

---

## Q16. What is routing?

**Answer:**

Routing matches a URL to an action, like `GET /api/incidents/5`. I set it with attributes like `[HttpGet("{id}")]` or with minimal API maps.

**Cross-question:** Conventional vs attribute routing?

**Cross-answer:**

APIs almost always use attributes. Conventional routes are more of an old MVC website style.

---

## Q17. What is `[ApiController]`?

**Answer:**

It is a marker that turns on API-friendly behaviour: an automatic 400 on a bad model, binding from the body by default, and problem details errors.

**Cross-question:** Can I skip it?

**Cross-answer:**

Yes, but then I have to handle `ModelState` myself. I keep it on API controllers.

---

## Q18. What is a filter in MVC?

**Answer:**

A filter is code that runs around an action, for things like auth, logging, or wrapping exceptions. It is different from middleware because it can see the action and its arguments.

**Cross-question:** Middleware or filter for a correlation id?

**Cross-answer:**

Middleware. It should run for every request, not only for controllers.

---

## Q19. How do you read the current user?

**Answer:**

I use `User` on the controller, or `IHttpContextAccessor` in a service. I read the claims that are already in the token. I don't trust a `userId` the client puts in the JSON body as the only check.

**Cross-question:** Can I trust `TenantId` from the Angular request?

**Cross-answer:**

No. I take the tenant from the host or from a claim I issued after login, and then I check it on the server.

---

## Q20. What is EF Core in one line?

**Answer:**

It maps C# classes to tables, so I query with LINQ instead of writing SQL strings in the app.

**Cross-question:** Can I still see the SQL?

**Cross-answer:**

Yes, by turning on logging in development. I look at it when a query is slow.

---

## Q21. `app.MapControllers()` vs `MapGet`?

**Answer:**

`MapControllers` picks up all your controller classes. `MapGet` maps a single minimal endpoint. After that, both go through the same pipeline.

**Cross-question:** Can I mix them?

**Cross-answer:**

Yes. Some teams use minimal endpoints for health checks and controllers for the rest.

---

## Q22. What is HTTPS redirection?

**Answer:**

It sends HTTP requests over to HTTPS, so passwords are not sent in plain text. In production it should be on. Plain http locally is just a dev convenience, not a production setup.

**Cross-question:** Would you turn off HTTPS to fix a local Angular error?

**Cross-answer:**

No. I fix the URL or the certificate. I don't ship “HTTPS off”.

---

## Q23. What is a 500 vs a 400 in your API?

**Answer:**

400 means the client sent something wrong. 500 means we failed. I never return a stack trace or a SQL error to the browser.

**Cross-question:** Unique key clash — 500 or 409?

**Cross-answer:**

409, or 400 with a clear message. It is not an unknown crash, so it should not be a 500.

---

## Q24. What is `IOptions<T>`?

**Answer:**

It is a typed bag of settings, which is better than string keys everywhere. I can also validate it at startup, so the app fails fast if, say, the SMTP settings are missing.

**Cross-question:** Snapshot vs Monitor?

**Cross-answer:**

`IOptions` is fixed for the life of the app. Snapshot can refresh per request. Monitor can notify you when the file changes. Most settings just use `IOptions`.

---

## Q25. How do you test a controller?

**Answer:**

I prefer to test the handler or service with a fake repository. For the HTTP side I use `WebApplicationFactory` and call the actual URL. I never hit a real production database.

**Cross-question:** Mock HttpContext?

**Cross-answer:**

If the action is fat, yes, and it gets messy. That is a sign the logic should move out of the controller.

---

## Q26. How is the NriCare API structured?

**Answer:**

It is not full Clean Architecture. There are two main .NET projects: `NriCare.Api` for HTTP, middleware, and jobs, and `NriCare.Domain` for entities, EF, repositories, and DTOs. There is also `NriCare.Fcm` for push notifications. Controllers stay thin. Most business logic lives in repositories like `BookingRepository` and `WalletRepository`.

**Cross-question:** Why not a separate Application layer?

**Cross-answer:**

It grew as a product API, and the repositories act like application services. If I started again I would split the use-cases out, but I won't pretend we have a four-layer design when we don't.

---

## Q27. Why do you have Web, App, and Common controllers?

**Answer:**

Because we have two clients. The Angular web app uses `api/...` with the Swagger group `web`. The mobile app uses `api/app/...` with the group `app`. Shared things like login, OTP, and file upload sit in Common and show up in both Swagger docs.

**Cross-question:** Same booking on both?

**Cross-answer:**

Yes, but with different actions. On web, the NR user makes the booking. The app is mostly for the service provider: list, change status, complete.

---

## Q28. How do you get the current user in a repository?

**Answer:**

First the JWT is validated. Then `SessionMiddleware` copies the claims into a scoped `IUserSession`: user id, access type, association id, service provider id, and so on. Repositories inject `IUserSession`. For “who am I”, I don't trust a user id from the JSON body.

**Cross-question:** Why middleware, not reading claims in every method?

**Cross-answer:**

It keeps it in one place. `ApplicationDbContext` also uses the same session for CreatedBy and the association filter.

---

## Q29. What is `ApiResponse` in your APIs?

**Answer:**

It is a common wrapper for every response: `isSuccess`, `statusCode`, `message`, `errors`, and `data`. Controllers use helpers like `OkResponse` and `BadRequestResponse`. Validation failures return **422**, not only 400.

**Cross-question:** Why 422?

**Cross-answer:**

400 is a bad request. 422 means the JSON parsed fine, but the model is invalid. Angular can then show field errors from `errors`.

---

## Q30. What database and ORM do you use?

**Answer:**

PostgreSQL with EF Core and Npgsql. It is one shared database, not one database per customer. Soft delete and association filters sit on `ApplicationDbContext`.

**Cross-question:** Why Postgres?

**Cross-answer:**

It is what the project uses. The features we actually rely on are transactions, `FOR UPDATE` on wallet rows, and UTC timestamps.

---

## Q31. How is DI registered in NriCare?

**Answer:**

`Program.cs` stays as the host. Repositories and helpers are registered in `ServiceRegistration()`. Almost everything is **scoped**. `IAmazonS3`, which we use for DigitalOcean Spaces, is a **singleton**. SignalR connection tracking is also a singleton.

**Cross-question:** Why is S3 a singleton?

**Cross-answer:**

The AWS client is thread-safe and expensive to create, so a new client per request is a waste. DbContext stays scoped.

---

## Q32. How do you handle unhandled exceptions?

**Answer:**

`ExceptionHandlingMiddleware` wraps the pipeline, logs the error, and returns a 500 `ApiResponse`. In production I would not send `exception.Message` to the client. Right now the middleware does include the message. I would change that to a generic text and keep the detail in the logs.

**Cross-question:** Does it cover SignalR?

**Cross-answer:**

No. Hub errors come back as `HubException` on the socket, and HTTP middleware does not wrap that.


---
## Additional Questions
### Q33. What is the difference between scoped, transient and singleton?
**Answer:** Scoped is normally one instance per request. Transient creates a new instance every time. Singleton lives for the whole life of the app. I keep DbContext scoped.

### Q34. Why should middleware order matter?
**Answer:** Middleware runs in the order I register it. Authentication must run before authorization, and exception handling should wrap the parts whose errors I want to catch.

### Q35. What is a health check?
**Answer:** It is an endpoint that tells deployment or monitoring tools whether the app is alive or ready. For readiness, I can check dependencies like PostgreSQL.
