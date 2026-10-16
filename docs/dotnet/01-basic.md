# .NET — Basic

Web APIs, DI, HTTP. Say it like you built a controller last week.

---

## Q1. What is ASP.NET Core?

**Answer:**

It is the web framework on .NET. I use it for HTTP APIs, MVC, SignalR, and so on. The host sets up logging, config, DI, and a pipeline of middleware for me.

In plain words, a framework is a ready-made toolkit. Instead of writing the code that listens for web requests, reads them and sends back answers, I get all of that out of the box and only write the parts that are about my own app. HTTP APIs are the endpoints an Angular or mobile app calls to get and save data. MVC is the pattern for pages and controllers. SignalR is for live, two-way updates like notifications.

The "host" is the part that starts the app. When it starts, it already gives me logging (writing down what happened), config (reading settings), DI (handing my classes the helpers they need) and middleware (the steps every request goes through). Without this I would have to build all of that plumbing myself before writing a single real feature. It also runs on Windows, Linux and macOS, so the same API can run on a Linux server or in Docker.

**Cross-question:** Controllers vs minimal APIs?

**Cross-answer:**

Controllers group actions in a class and use attributes. Minimal APIs map routes in `Program.cs` with less code. Both work. Bigger apps often keep controllers so the files stay tidy.

Think of controllers as folders: a `BookingsController` holds every booking endpoint in one place, and attributes like `[HttpGet]` say which URL each method answers. Minimal APIs are more like sticky notes: one line says "for this URL, run this function". That is great for a small service or a health check, but with hundreds of endpoints a single file gets messy, which is why bigger apps tend to stay with controllers.

---

## Q2. What is middleware?

**Answer:**

Middleware is a chain of small pieces that every request passes through. Each piece can run some code, call the next piece, or stop the request there. Order matters: error handling first, then HTTPS, then authentication, then authorization, then your endpoints.

A simple picture is airport security. You go through check-in, then the ID check, then the bag scan, then the gate. Each desk can let you through to the next one or send you back. In the same way, one middleware might log the request, the next checks the login token, the next checks permissions, and only then does the request reach my controller. On the way back, the response passes through the same pieces in reverse order.

Order matters because each piece only sees what came before it. Error handling goes first so it can catch errors from everything after it. Authentication has to come before authorization, because I can't check what you are allowed to do before I know who you are. If I get the order wrong, things like login checks can quietly stop working.

**Cross-question:** `Use` vs `Run` vs `Map`?

**Cross-answer:**

`Use` can call the next piece. `Run` is the end of the chain. `Map` branches the pipeline by path.

So `Use` is a normal step in the chain: do something, then pass the request on (or not). `Run` is the last stop: it handles the request and nothing after it runs. `Map` is like a fork in the road: for example, every request under `/health` can go down its own small chain, while everything else follows the main one.

---

## Q3. Dependency Injection in ASP.NET Core?

**Answer:**

My class does not `new` its helpers. The app creates them and passes them in, usually through the constructor.

For example, a `BookingService` needs a database context and an email sender. Instead of creating them inside the class, it just asks for them in its constructor, and ASP.NET Core's built-in container hands them over. I tell the container once, at startup, which class to use for each need.

Why bother? If a class creates its own helpers, it is stuck with them. I can't swap the real email sender for a fake one in a test, and changing one helper means editing every class that creates it. With DI the class only says "I need something that can send email", and the app decides what that is. It's like a restaurant kitchen: the chef doesn't go out to buy vegetables, the supplier delivers them, and the chef just cooks.

**Cross-question:** Transient, scoped, singleton?

**Cross-answer:**

Transient: a new one every time. Scoped: one per HTTP request. Singleton: one for the whole app.

I never put a scoped thing (like `DbContext`) inside a singleton. That scoped object would then live forever and get shared across users.

These three are called lifetimes, meaning how long one created object is kept and reused. A simple way to remember: transient is a paper cup (use once, get a new one), scoped is your plate at a buffet (yours for this one visit), singleton is the restaurant's one big fridge (everyone shares it the whole time). The danger in mixing them is that a singleton never goes away, so anything it holds on to also never goes away, even if that thing was only meant to live for one request.

---

## Q4. What is `DbContext`? Why scoped?

**Answer:**

It is EF's unit of work, meaning it keeps track of the entities I load and change. It is not safe to use from many threads at once. So the default is one per request.

In everyday words, `DbContext` is my working session with the database. When I load a booking and change its status, the context remembers that change. When I call `SaveChanges`, it writes all the changes I made in that session to the database together. Entities are just the C# objects that match rows in the tables.

"Not safe from many threads" means two pieces of code using the same context at the same moment can confuse its tracking and throw errors or save the wrong data. One context per request fits well: each request is one user doing one job, so it gets its own clean session, and that session is thrown away when the request ends. It's like each customer at a bank getting their own form to fill in, instead of everyone writing on the same sheet at once.

**Cross-question:** Hangfire / background job needs EF?

**Cross-answer:**

I open a **new scope**, get `DbContext` from that scope, and dispose the scope when I'm done. I do not reuse the request's context on another thread.

A background job has no HTTP request, so there is no request scope to hang the context on. Opening my own scope gives the job its own fresh `DbContext`, just like a request would get. Disposing the scope at the end closes that context and its database connection. If I reused the request's context instead, the request might already be finished and the context disposed, or both might use it at the same time, which is exactly what it can't handle.

---

## Q5. GET vs POST vs PUT vs PATCH vs DELETE?

**Answer:**

GET reads, and it should not change data. POST creates something or runs an action. PUT replaces the whole thing. PATCH updates part of it. DELETE removes it.

These are HTTP methods, the "verb" that says what a request wants to do. Using them the expected way makes an API easy to guess. For example, with bookings: `GET /api/bookings/5` reads booking 5, `POST /api/bookings` creates a new one, `PUT /api/bookings/5` sends the full booking to replace it, `PATCH /api/bookings/5` sends only the field that changed (say, the status), and `DELETE /api/bookings/5` removes it.

The difference between PUT and PATCH is like editing a document. PUT is handing over a whole new copy of the document. PATCH is handing over a note that says "change line 3". With PUT, any field I leave out may be treated as empty, so PUT should carry the whole object.

**Cross-question:** Why must GET not change data?

**Cross-answer:**

Browsers, caches, and retries all treat GET as safe. Side effects belong on POST/PUT/PATCH. Also, I never put secrets in the query string.

Because GET is expected to be harmless, other tools feel free to repeat it. A browser may prefetch a link, a proxy may cache the answer, and a client may retry it after a timeout. If `GET /cancel-booking/5` actually cancelled the booking, any of those could cancel it by accident. Secrets don't belong in the query string because URLs end up in browser history, server logs and proxy logs, where anyone reading those logs can see them.

---

## Q6. Model binding and validation?

**Answer:**

ASP.NET takes values from the URL, query, and body and maps them into my parameters. Then `[Required]` or FluentValidation checks the data. With `[ApiController]`, bad input turns into a 400 automatically.

Model binding is the step that fills in my method's inputs for me. If the URL is `/api/bookings/5?includeItems=true` and the body has some JSON, ASP.NET puts `5` into my `id` parameter, `true` into `includeItems`, and turns the JSON into a C# object. I don't have to read the raw request text myself.

Validation then checks that the data makes sense: a required field is filled, an email looks like an email, a number is in range. `[Required]` is an attribute I put on a property. FluentValidation is a library where I write those rules in a separate class. Without validation, bad data would get all the way into the database or cause confusing errors later. With `[ApiController]`, if the check fails, the client gets a 400 with the list of problems, and my action doesn't even run.

**Cross-question:** Angular validation vs server validation?

**Cross-answer:**

The UI check is for the user's convenience. The **server** check is for safety. I always check again on the API.

The Angular check gives quick feedback, like a red message under an empty field, without waiting for the server. But anyone can skip the UI and call the API directly with Postman or a script, so the UI check protects nothing. It's like a shop's "please pay at the counter" sign: helpful for honest customers, but the cashier still has to check you actually paid.

---

## Q7. What is a DTO? Why not return the EF entity?

**Answer:**

A DTO is the shape of data I send to the client. Entities have extra fields and navigation properties, and they can loop forever in JSON. So I map to a DTO.

DTO stands for Data Transfer Object. It is a simple class with only the fields the client needs, and nothing else. The EF entity is the class that matches the database table, and it often carries things the client should never see, like a password hash, internal flags or audit columns.

Navigation properties are links from one entity to another, like `Booking.User` and `User.Bookings`. If I send the entity straight to JSON, the serializer follows Booking to User, then back to the User's Bookings, then to each Booking's User again, and so on. That is the "loop forever" problem. A DTO cuts this off, keeps private fields private, and means I can change the database later without breaking the shape the Angular app expects.

**Cross-question:** AutoMapper or hand mapping?

**Cross-answer:**

A hand-written `Select` is clear, and EF can turn it into SQL. AutoMapper is faster to type. Either is fine as long as I don't leak entities.

With a hand-written `Select`, I can read exactly which field goes where, and because EF translates it into SQL, the database only returns the columns I asked for. AutoMapper is a library that copies fields with matching names for me, so there's less typing, but the mapping is hidden in a config and mistakes are easier to miss. The choice is about taste. The rule that matters is that the client gets a DTO, not the entity.

---

## Q8. Where does config live?

**Answer:**

`appsettings.json` first, then the environment file, then environment variables, then user secrets in dev. Production passwords never go in git.

Config means settings that can change without changing code: connection strings, API URLs, email settings and so on. ASP.NET Core reads them from several places in this order, and a later source overrides an earlier one with the same key. So `appsettings.json` holds the defaults, `appsettings.Production.json` (the environment file) changes what is different in production, and environment variables on the server can override both.

User secrets are a small file stored on my own machine, outside the project folder, for development only. That way my local passwords never end up in the repository by mistake. Production passwords stay out of git because anyone with access to the code, now or in the future, could read them. Instead they are set on the server, for example as environment variables.

**Cross-question:** Bind a section to a class?

**Cross-answer:**

I call `Configure<SmtpOptions>(...)` and inject `IOptions<SmtpOptions>`. I don't sprinkle `configuration["Smtp:Password"]` all over the code.

So I make a small class like `SmtpOptions` with properties `Host`, `Port`, `Password`, and tell ASP.NET to fill it from the `Smtp` section of the config. Any class that needs email settings just asks for `IOptions<SmtpOptions>`. This is safer than string keys, because a typo in `"Smtp:Pasword"` only shows up at runtime as a missing value, while a typo in a property name won't even compile.

---

## Q9. Authentication vs authorization?

**Answer:**

Authentication: who are you?

Authorization: what are you allowed to do?

`UseAuthentication` must run before `UseAuthorization`.

A hotel is a good comparison. At the front desk you show your ID and get a key card. That is authentication: the hotel now knows who you are. Later the key card opens your room but not someone else's room. That is authorization: deciding what you are allowed to do.

In an API, authentication usually means checking the login token sent with the request and reading the user from it. Authorization then checks rules like "only admins can delete" or "you can only see your own bookings". The order matters because authorization needs to know who the user is. If authorization ran first, it would see every user as anonymous.

**Cross-question:** Angular route guard vs `[Authorize]`?

**Cross-answer:**

A guard only hides a screen. Anyone can still call the API directly. So the API must do the check.

A route guard is Angular code that stops a user from opening a page, for example the admin page. That's good for user experience, but the code runs in the user's own browser, so they can change it or ignore it. They can also skip the website and send requests straight to the API. `[Authorize]` runs on the server, where the user can't touch it, so that is the check that actually protects the data.

---

## Q10. What is Kestrel?

**Answer:**

It is the web server built into ASP.NET Core. In production we often put Nginx or IIS in front of it for HTTPS and process management.

A web server is the program that listens on a port, accepts incoming HTTP requests and passes them to my code. Kestrel comes with ASP.NET Core, so when I press run, Kestrel is what's actually listening on `localhost`.

Nginx or IIS in front of it act as a reverse proxy, which is like a receptionist at the building door. Visitors talk to the receptionist, and the receptionist passes the request on to the right office. The proxy can handle HTTPS certificates, restart the app if it crashes, and serve several apps on the same server, so Kestrel can focus on running my app.

**Cross-question:** Can Kestrel do HTTPS itself?

**Cross-answer:**

Yes. But many teams still end HTTPS at the proxy. In that case the app must trust forwarded headers, so it knows the real scheme and client IP.

"End HTTPS at the proxy" means the encrypted connection stops at Nginx, and Nginx talks to Kestrel over plain HTTP inside the server. The catch is that the app now sees every request as coming from the proxy, over HTTP. The proxy adds headers like `X-Forwarded-For` (the real client IP) and `X-Forwarded-Proto` (the original scheme, https). The app has to be told to read those headers, otherwise HTTPS redirects can loop and logs show the proxy's IP for every user.

---

## Q11. Logging vs `Console.WriteLine`?

**Answer:**

I use `ILogger<T>`. It gives me levels, message templates, and sinks (places the logs go). I log `"User {UserId} logged in"`, not the token. I never log passwords or connection strings.

`Console.WriteLine` just prints text to a screen that nobody watches in production, and it's gone once the app restarts. `ILogger<T>` is the built-in logging tool. The `T` is the class doing the logging, so every log line says where it came from.

Levels like Information, Warning and Error let me choose how much detail to keep. For example, I can show everything in development but only warnings and errors in production. Sinks mean the same log line can go to the console, a file, or a log service, just by changing config. I keep secrets out of logs because logs are read by many people and kept for a long time, so a password in a log is a leaked password.

**Cross-question:** Why `{UserId}` not string concat?

**Cross-answer:**

The message text stays the same, and the id becomes a field I can search on. With concat it is also easier to leak secrets into the message.

With `"User {UserId} logged in"`, the logger stores the template and the value separately. In a log tool I can then filter "all lines where UserId = 42", or count logins grouped by that template. With `"User " + id + " logged in"`, every line is a different string, so all I can do is a text search. And when people get used to gluing whole objects into a string, it's easy to glue in a token or a password by mistake.

---

## Q12. BackgroundService / hosted service?

**Answer:**

It is work that keeps running inside the app, like a queue reader or a timer. `ExecuteAsync` gets a stopping token so it knows when the app is shutting down.

A normal API only does work when a request comes in. Some work has no request behind it: checking a queue for new messages, sending reminder emails every hour, cleaning old records at night. A hosted service is a class that starts with the app and keeps running in the background. `BackgroundService` is the base class I usually inherit from, and I put my loop inside `ExecuteAsync`.

The stopping token is like a "we're closing" announcement in a shop. When the app is shutting down, the token is cancelled, and my loop should notice it, finish what it's doing and exit cleanly. If I ignore it, shutdown can hang or a job can be cut off halfway.

**Cross-question:** Hosted service vs Hangfire vs a Worker project?

**Cross-answer:**

Hosted service: simple, but it dies with the app. Hangfire: saved jobs, retries, and a dashboard. A separate Worker: its own process that scales on its own.

In NriCare I do not use Hangfire. Background work runs on **TickerQ**, stored in the same Postgres database, with a dashboard at `/tickerq`.

To put it simply: a hosted service keeps its work in memory, so if the app restarts, any job that was in the middle of running is lost. A job library like Hangfire saves jobs in a database, so they survive restarts, can retry after a failure, and can be watched on a dashboard. A Worker project is a separate app just for background work, so heavy jobs don't slow down the API and I can run more copies of it when needed.

---

## Q13. Status codes you actually use?

**Answer:**

200 ok, 201 created, 204 no content, 400 bad input, 401 not logged in, 403 logged in but not allowed, 404 not found, 409 conflict, 500 server error.

A status code is the number at the top of every HTTP response that tells the client, in one glance, how things went. Codes starting with 2 mean success, 4 means the client did something wrong, and 5 means the server failed. Angular and mobile apps decide what to do from this number: show the data, show a form error, send the user to login, or show "something went wrong".

Some examples: after creating a booking I return 201. After a delete that has nothing to send back, 204. If a required field is missing, 400. If the token is missing or expired, 401, so the app can send the user to login. If two people try to book the same slot and one gets there first, 409 for the second one. Getting these right means the client can react correctly without reading the message text.

**Cross-question:** User is logged in but tries another association’s booking?

**Cross-answer:**

403, or 404 so we don't even admit it exists. 401 is wrong here, because 401 means we don't know who you are.

Here we know exactly who the user is, they are just not allowed to see this booking, so it's about permission, not login. 403 says that honestly. 404 hides even the fact that the booking exists, which stops someone from guessing ids to find out which bookings are real. Returning 401 would also confuse the client into logging the user out for no reason.

---

## Q14. What is CORS?

**Answer:**

It is a browser rule. The browser blocks a page on origin A from reading responses from origin B, unless B allows it.

An origin is the scheme, domain and port together, like `https://app.example.com`. If my Angular app runs on `localhost:4200` and my API runs on `localhost:5000`, those are two different origins. By default the browser won't let the Angular page read the API's response. This protects users: a random website you visit can't quietly call your bank's API using your logged-in cookies and read your data.

CORS (Cross-Origin Resource Sharing) is how the API says "I allow these origins". The API sends headers listing which sites, methods and headers are allowed, and the browser checks them. So when Angular shows a CORS error, the fix is on the API side: allow the Angular app's origin.

**Cross-question:** Does CORS protect a mobile app or Postman?

**Cross-answer:**

No. CORS is only a browser rule. The API still needs login and permission checks. I don't turn on “allow everyone” just to make Angular work.

Postman, mobile apps and server scripts don't enforce CORS at all, they just send the request and read the answer. So CORS is not a lock on the API, it only protects users inside a browser. "Allow everyone" makes the error go away, but it also removes that protection for every site on the internet. The right fix is to list the exact origins of my own frontends.

---

## Q15. What happens in `Program.cs`?

**Answer:**

I build the host, register services, build the app, set up middleware, map endpoints, and run it. It is the composition root, the one place where everything gets wired together.

Step by step: `WebApplication.CreateBuilder` creates the builder and loads config and logging. Then I register services, which means telling DI which classes to create for which needs (database, repositories, email and so on). `builder.Build()` creates the app. Then I add middleware in order, like exception handling, authentication and authorization. Then I map endpoints, for example `MapControllers()`. Finally `app.Run()` starts the server and waits for requests.

"Composition root" sounds fancy, but it just means the one place where all the parts are connected. Like the fuse box in a house, it's the single spot where everything is wired up. The rest of the code just asks for what it needs and doesn't care how it was built.

**Cross-question:** 200 lines of DI in Program?

**Cross-answer:**

I move them into extension methods like `AddInfrastructure(config)`. That keeps Program short.

An extension method lets me write `builder.Services.AddInfrastructure(config)` and keep the 50 lines of database and storage registrations in their own file. I group them by area, for example infrastructure, auth and repositories. `Program.cs` then reads like a short table of contents, and when I need to change how email is set up, I know which file to open.

---

## Q16. What is routing?

**Answer:**

Routing matches a URL to an action, like `GET /api/incidents/5`. I set it with attributes like `[HttpGet("{id}")]` or with minimal API maps.

When a request arrives, ASP.NET has to decide which method should handle it. Routing is that decision. It looks at the HTTP method and the path, and finds the matching action. In `GET /api/incidents/5`, the controller route gives `api/incidents`, `[HttpGet("{id}")]` says "a GET with one more piece", and that piece, `5`, is passed into my `id` parameter.

It's like a building directory in a lobby: "Floor 3, Room 5". Without routing, every request would land in one place and I would have to write my own giant `if` block to send it to the right code.

**Cross-question:** Conventional vs attribute routing?

**Cross-answer:**

APIs almost always use attributes. Conventional routes are more of an old MVC website style.

Conventional routing sets one general pattern for the whole app, like `{controller}/{action}/{id?}`, and URLs are built from class and method names. Attribute routing puts the exact URL right on each controller and action. For APIs, URLs are a contract that clients depend on, so I want them clear and written next to the code, which is what attributes give me.

---

## Q17. What is `[ApiController]`?

**Answer:**

It is a marker that turns on API-friendly behaviour: an automatic 400 on a bad model, binding from the body by default, and problem details errors.

I put this attribute on a controller class, and it switches on a set of sensible defaults for APIs. First, if validation fails, ASP.NET returns a 400 by itself before my action runs, so I don't need `if (!ModelState.IsValid)` at the top of every method. Second, complex parameters like a `CreateBookingDto` are read from the JSON body without me adding `[FromBody]`. Third, errors come back in a standard JSON format called problem details, so every client sees the same shape for errors.

Without it, each developer handles these things a bit differently, and it's easy to forget the validation check in one action and let bad data through.

**Cross-question:** Can I skip it?

**Cross-answer:**

Yes, but then I have to handle `ModelState` myself. I keep it on API controllers.

`ModelState` is where ASP.NET stores the validation results. Without `[ApiController]`, an invalid model still reaches my action, and if I forget to check `ModelState`, I end up working with bad data. Keeping the attribute on means the check can't be forgotten.

---

## Q18. What is a filter in MVC?

**Answer:**

A filter is code that runs around an action, for things like auth, logging, or wrapping exceptions. It is different from middleware because it can see the action and its arguments.

"Around an action" means it can run just before my controller method and just after it. For example, a filter could log how long each action took, or turn certain exceptions into a clean error response. `[Authorize]` itself works as a filter.

Middleware sits earlier, at the level of the raw request. It knows the URL and headers but doesn't know which controller method will run. A filter runs after routing and model binding, so it knows the action, its attributes and the parsed parameters. That's useful when I want behaviour only on some actions, for example "check this permission only on methods marked with my custom attribute".

**Cross-question:** Middleware or filter for a correlation id?

**Cross-answer:**

Middleware. It should run for every request, not only for controllers.

A correlation id is a unique id given to each request so I can follow it across all the log lines (and across other services) it touches. Filters only run for MVC actions, so health checks, minimal endpoints, static files or errors that happen before the controller would get no id. Middleware sees every request from the very start, which is what I need here.

---

## Q19. How do you read the current user?

**Answer:**

I use `User` on the controller, or `IHttpContextAccessor` in a service. I read the claims that are already in the token. I don't trust a `userId` the client puts in the JSON body as the only check.

Claims are small facts about the user stored inside the login token, like their id, email or role. The server signed that token when the user logged in, so if anyone changes it, the signature check fails. `User` on a controller gives me those claims directly. In a service class that has no controller, `IHttpContextAccessor` gives me access to the current request and its user.

A `userId` in the JSON body is just text the client typed. Anyone can change it to someone else's id in Postman. If I trusted it, a user could read or change another user's data just by changing a number.

**Cross-question:** Can I trust `TenantId` from the Angular request?

**Cross-answer:**

No. I take the tenant from the host or from a claim I issued after login, and then I check it on the server.

A tenant is one customer organisation in a shared system. If the tenant comes from the request body, a user could switch it and see another organisation's data. The host (like `clientA.example.com`) or a claim the server put into the token after login are both decided by the server side, not typed freely by the client, and the server still checks that this user really belongs to that tenant.

---

## Q20. What is EF Core in one line?

**Answer:**

It maps C# classes to tables, so I query with LINQ instead of writing SQL strings in the app.

EF Core (Entity Framework Core) is an ORM, a tool that translates between C# objects and database rows. I write `context.Bookings.Where(b => b.Status == "Open")`, and EF turns it into a SQL `SELECT` for me and fills C# objects with the results. When I change an object and call `SaveChanges`, it writes the `UPDATE` too.

This helps because the compiler checks my queries (a wrong property name won't build), I don't have to hand-write SQL strings full of typos, and parameters are handled for me, which protects against SQL injection.

**Cross-question:** Can I still see the SQL?

**Cross-answer:**

Yes, by turning on logging in development. I look at it when a query is slow.

EF writes the generated SQL to the logs when that log category is turned on. Reading it shows things that are hard to see from the LINQ alone, like a missing `WHERE`, a query that loads far more columns than needed, or many small queries where one would do. I can also copy the SQL into the database tool and check how the database runs it.

---

## Q21. `app.MapControllers()` vs `MapGet`?

**Answer:**

`MapControllers` picks up all your controller classes. `MapGet` maps a single minimal endpoint. After that, both go through the same pipeline.

`MapControllers()` is one line that finds every controller in the project and registers all their routes from the attributes. `MapGet("/health", ...)` registers just one URL for GET requests and points it to one function.

"The same pipeline" means both kinds of endpoints pass through the same middleware: the same exception handling, authentication, authorization and logging. So choosing one or the other doesn't mean losing security or error handling.

**Cross-question:** Can I mix them?

**Cross-answer:**

Yes. Some teams use minimal endpoints for health checks and controllers for the rest.

A health check is a tiny endpoint that just returns "OK", so a whole controller class for it would be overkill. Meanwhile the main business endpoints get the structure of controllers. Both live in the same app without any conflict, as long as two of them don't claim the same URL.

---

## Q22. What is HTTPS redirection?

**Answer:**

It sends HTTP requests over to HTTPS, so passwords are not sent in plain text. In production it should be on. Plain http locally is just a dev convenience, not a production setup.

HTTP sends everything as readable text. Anyone on the same Wi-Fi or network path could read the login password or token. HTTPS encrypts the connection so only the browser and the server can read it. HTTPS redirection is middleware that, when a request comes in over `http://`, replies "please go to the `https://` address instead", and the browser follows that.

It's like a shop with two doors where one has no lock: the redirect is a sign on the unlocked door pointing everyone to the safe one.

**Cross-question:** Would you turn off HTTPS to fix a local Angular error?

**Cross-answer:**

No. I fix the URL or the certificate. I don't ship “HTTPS off”.

Local HTTPS errors usually come from Angular calling the wrong address (http vs https, or the wrong port) or from the local development certificate not being trusted yet. Both are quick to fix properly. Turning HTTPS off "just for now" has a habit of getting committed and ending up in production, where it would expose real passwords.

---

## Q23. What is a 500 vs a 400 in your API?

**Answer:**

400 means the client sent something wrong. 500 means we failed. I never return a stack trace or a SQL error to the browser.

The difference tells the client who has to fix the problem. With a 400, the user can correct the input and try again, like fixing a wrong date. With a 500, the user did nothing wrong, the bug or outage is on our side, and retrying with the same data won't help until we fix it.

A stack trace or SQL error shows the inside of the system: file paths, table names, library versions, sometimes even data. That's useful to an attacker and confusing to a normal user. So the client gets a short, plain message, and the full detail goes into our logs where only we can see it.

**Cross-question:** Unique key clash — 500 or 409?

**Cross-answer:**

409, or 400 with a clear message. It is not an unknown crash, so it should not be a 500.

A unique key clash means the user tried to save something that already exists, like registering with an email that's already taken. That's a known, expected situation, not a server bug. 409 Conflict says "this clashes with the current data", and a clear message like "This email is already registered" tells the user what to do. A 500 would make it look like the system broke and send us chasing a problem that isn't there.

---

## Q24. What is `IOptions<T>`?

**Answer:**

It is a typed bag of settings, which is better than string keys everywhere. I can also validate it at startup, so the app fails fast if, say, the SMTP settings are missing.

"Typed" means the settings come as a C# class with real properties, like `options.Value.Host`, instead of looking up `"Smtp:Host"` as a string. The compiler catches typos, and my editor shows me which settings exist.

Validating at startup means the app checks the settings the moment it starts, and refuses to start if something required is missing. Without that, the app starts fine and only crashes days later, the first time someone triggers an email. Failing fast is like a pilot's checklist before take-off: it's much better to find a problem on the ground than in the air.

**Cross-question:** Snapshot vs Monitor?

**Cross-answer:**

`IOptions` is fixed for the life of the app. Snapshot can refresh per request. Monitor can notify you when the file changes. Most settings just use `IOptions`.

So with `IOptions`, the values are read once and stay the same until the app restarts. `IOptionsSnapshot` reads them again for each request, so if the config file changes, the next request sees the new values. `IOptionsMonitor` always gives the latest values and also lets me run some code the moment they change. For most settings, like SMTP details or API URLs, a restart to pick up changes is fine, so plain `IOptions` is enough.

---

## Q25. How do you test a controller?

**Answer:**

I prefer to test the handler or service with a fake repository. For the HTTP side I use `WebApplicationFactory` and call the actual URL. I never hit a real production database.

Most of the real logic should live in a service, not in the controller. So I test the service directly and give it a fake repository that returns the data I set up, like "a booking that is already cancelled". These tests are fast and test one rule at a time.

To make sure the HTTP part works too (routes, validation, auth, status codes), I use `WebApplicationFactory`. It starts the whole app in memory, and my test sends real HTTP requests to it, like `GET /api/bookings/5`, and checks the response. The database for those tests is a test one, never production, because tests create and delete data and must never touch real customers' records.

**Cross-question:** Mock HttpContext?

**Cross-answer:**

If the action is fat, yes, and it gets messy. That is a sign the logic should move out of the controller.

`HttpContext` holds everything about the current request: user, headers, query and more. Faking all of that in a unit test takes a lot of setup code. If I find myself doing it a lot, it usually means the controller is doing too much. When the logic moves into a service that just takes plain values like a user id, the test becomes simple again.

---

## Q26. How is the NriCare API structured?

**Answer:**

It is not full Clean Architecture. There are two main .NET projects: `NriCare.Api` for HTTP, middleware, and jobs, and `NriCare.Domain` for entities, EF, repositories, and DTOs. There is also `NriCare.Fcm` for push notifications. Controllers stay thin. Most business logic lives in repositories like `BookingRepository` and `WalletRepository`.

Clean Architecture is a style where the code is split into several layers (domain, application, infrastructure, web), each with strict rules about who can depend on whom. NriCare uses a simpler split. The Api project is the front door: it receives requests, runs middleware and hosts background jobs. The Domain project holds the data model and the code that reads and writes the database.

"Controllers stay thin" means a controller just takes the request, calls the right repository method and returns the result. The actual rules, like how a booking moves between statuses or how wallet money is handled, sit in the repositories, so they're kept in one place.

**Cross-question:** Why not a separate Application layer?

**Cross-answer:**

It grew as a product API, and the repositories act like application services. If I started again I would split the use-cases out, but I won't pretend we have a four-layer design when we don't.

An Application layer would hold the use-cases ("create booking", "top up wallet") separate from the code that talks to the database. Here those two jobs live together in the repositories. That works, but it mixes business rules with data access, so splitting them would make each part easier to test and change on its own.

---

## Q27. Why do you have Web, App, and Common controllers?

**Answer:**

Because we have two clients. The Angular web app uses `api/...` with the Swagger group `web`. The mobile app uses `api/app/...` with the group `app`. Shared things like login, OTP, and file upload sit in Common and show up in both Swagger docs.

Each client needs different endpoints, so splitting them keeps each side clear. Swagger is the auto-generated API documentation page, and grouping means the web team sees only web endpoints in their doc and the mobile team sees only app endpoints in theirs. Neither has to dig through endpoints meant for the other client.

The `api/app/...` prefix also makes it obvious from a URL alone which client an endpoint is for. Common holds the features both clients need in the same form, so login or file upload is written once instead of twice.

**Cross-question:** Same booking on both?

**Cross-answer:**

Yes, but with different actions. On web, the NR user makes the booking. The app is mostly for the service provider: list, change status, complete.

So it's the same booking data in the same database, seen from two sides. The web endpoints cover what the NR user needs, like creating a booking. The app endpoints cover what the service provider does with it, like seeing their list, updating the status and marking the job complete. Each client only gets the actions that make sense for its user.

---

## Q28. How do you get the current user in a repository?

**Answer:**

First the JWT is validated. Then `SessionMiddleware` copies the claims into a scoped `IUserSession`: user id, access type, association id, service provider id, and so on. Repositories inject `IUserSession`. For “who am I”, I don't trust a user id from the JSON body.

A JWT (JSON Web Token) is the signed login token the client sends with each request. "Validated" means the server checks the signature and expiry, so it knows the token is real and wasn't changed. The claims inside it are the facts about the user.

`IUserSession` is a small object that holds those facts for the current request. Because it's scoped, each request gets its own copy, so one user's details never mix with another's. Any repository that needs to know who is calling just asks for `IUserSession` in its constructor, instead of digging into the HTTP request itself.

**Cross-question:** Why middleware, not reading claims in every method?

**Cross-answer:**

It keeps it in one place. `ApplicationDbContext` also uses the same session for CreatedBy and the association filter.

If every method read the claims by itself, the same parsing code would be copied everywhere, and one mistake (like a wrong claim name) would have to be fixed in many places. With the middleware filling the session once per request, everyone reads the same values. That's also how `ApplicationDbContext` can fill in CreatedBy automatically and limit queries to the user's association without each repository remembering to do it.

---

## Q29. What is `ApiResponse` in your APIs?

**Answer:**

It is a common wrapper for every response: `isSuccess`, `statusCode`, `message`, `errors`, and `data`. Controllers use helpers like `OkResponse` and `BadRequestResponse`. Validation failures return **422**, not only 400.

A wrapper means every response, success or failure, has the same outer shape, with the real result inside `data`. The Angular and mobile apps can then handle all responses with the same code: check `isSuccess`, show `message`, read `data`, or list the `errors`. They don't need special handling for each endpoint.

The helpers like `OkResponse` build that shape for me, so each controller doesn't build it by hand and the format stays consistent across the whole API.

**Cross-question:** Why 422?

**Cross-answer:**

400 is a bad request. 422 means the JSON parsed fine, but the model is invalid. Angular can then show field errors from `errors`.

So 400 covers "I couldn't even understand what you sent", for example broken JSON. 422 (Unprocessable Entity) covers "I understood it, but the values break the rules", like an empty required name or a past date for a booking. When the client sees 422, it knows `errors` holds field-level messages it can show right next to each form field.

---

## Q30. What database and ORM do you use?

**Answer:**

PostgreSQL with EF Core and Npgsql. It is one shared database, not one database per customer. Soft delete and association filters sit on `ApplicationDbContext`.

Npgsql is the driver that lets .NET and EF Core talk to PostgreSQL. A shared database means all customers' data lives in the same tables, and each row records which association it belongs to.

Soft delete means a "deleted" row isn't actually removed, it's just marked as deleted and hidden from normal queries, so it can be checked or restored later. The association filter automatically adds "only rows for this user's association" to queries. Putting both on `ApplicationDbContext` means they apply everywhere, so a developer can't forget them in one query and show deleted records or another customer's data.

**Cross-question:** Why Postgres?

**Cross-answer:**

It is what the project uses. The features we actually rely on are transactions, `FOR UPDATE` on wallet rows, and UTC timestamps.

A transaction groups several changes so they all save or none do. For a wallet, debiting one place and recording the entry must happen together. `FOR UPDATE` locks the selected wallet row until the transaction ends, so two requests can't read the same balance at the same moment and both spend it. UTC timestamps store time in one standard time zone, which avoids confusion when users and servers are in different time zones.

---

## Q31. How is DI registered in NriCare?

**Answer:**

`Program.cs` stays as the host. Repositories and helpers are registered in `ServiceRegistration()`. Almost everything is **scoped**. `IAmazonS3`, which we use for DigitalOcean Spaces, is a **singleton**. SignalR connection tracking is also a singleton.

`ServiceRegistration()` is the one method that lists all the repositories and helpers, which keeps `Program.cs` short and gives me one place to look. Most services are scoped (one per request) because they work with the request's `DbContext` and the current user's session, and those belong to a single request.

DigitalOcean Spaces is a file storage service that speaks the same API as Amazon S3, so the AWS S3 client works with it. SignalR connection tracking is a singleton because it has to remember which users are connected across all requests, and a per-request copy would forget everything as soon as the request ended.

**Cross-question:** Why is S3 a singleton?

**Cross-answer:**

The AWS client is thread-safe and expensive to create, so a new client per request is a waste. DbContext stays scoped.

Thread-safe means many requests can use the same client at the same time without problems. Since creating it costs time and resources, it's better to create it once and share it, like one shared printer in an office instead of buying a new one for each document. `DbContext` is the opposite case: it's not thread-safe, so it stays one per request.

---

## Q32. How do you handle unhandled exceptions?

**Answer:**

`ExceptionHandlingMiddleware` wraps the pipeline, logs the error, and returns a 500 `ApiResponse`. In production I would not send `exception.Message` to the client. Right now the middleware does include the message. I would change that to a generic text and keep the detail in the logs.

An unhandled exception is an error nobody caught in the code, like a null value or a lost database connection. Without a handler, the client might get an empty or confusing response. This middleware sits near the start of the pipeline, so any error that happens later bubbles up to it. It writes the full error to the logs for us and sends the client a clean 500 in the same `ApiResponse` shape as every other response.

The reason to hide `exception.Message` is that it can show internal details, like table names or file paths. A generic text like "Something went wrong" tells the user enough, and the logs still have the full story for us.

**Cross-question:** Does it cover SignalR?

**Cross-answer:**

No. Hub errors come back as `HubException` on the socket, and HTTP middleware does not wrap that.

SignalR starts with an HTTP request, but then keeps a long-lived connection open (usually a WebSocket) and sends messages over it. Errors inside hub methods happen inside that connection, not in a normal request-response, so they never pass back through the HTTP middleware. SignalR handles them itself and sends the client a `HubException`, so hub errors need their own handling and logging.


---
## Additional Questions
### Q33. What is the difference between scoped, transient and singleton?
**Answer:** Scoped is normally one instance per request. Transient creates a new instance every time. Singleton lives for the whole life of the app. I keep DbContext scoped.

These are the three DI lifetimes, meaning how long the app keeps and reuses one object. Transient suits small, light helpers with no state. Scoped suits things tied to one request, like `DbContext` or the current user's session. Singleton suits shared, thread-safe things that are costly to create, like an HTTP or storage client. Picking the wrong one causes real bugs: a `DbContext` held by a singleton would be shared by all users and used from many threads at once, which it can't handle.

### Q34. Why should middleware order matter?
**Answer:** Middleware runs in the order I register it. Authentication must run before authorization, and exception handling should wrap the parts whose errors I want to catch.

It's like an assembly line: each station only works on what earlier stations have already done. Authorization needs to know who the user is, and only authentication sets that, so authentication has to come first. Exception handling only catches errors from the steps that come after it, so I register it early. If I put it at the end, errors from authentication or other middleware would slip past it and reach the client unhandled.

### Q35. What is a health check?
**Answer:** It is an endpoint that tells deployment or monitoring tools whether the app is alive or ready. For readiness, I can check dependencies like PostgreSQL.

Think of it as the app answering "are you OK?". "Alive" means the process is running and responding. "Ready" means it can actually do its job, for example it can reach the database. Tools like a load balancer or a container platform call this endpoint regularly. If the app isn't ready, they stop sending users to it, and if it's not alive, they can restart it. Without a health check, users would keep landing on a broken copy of the app until someone noticed.
