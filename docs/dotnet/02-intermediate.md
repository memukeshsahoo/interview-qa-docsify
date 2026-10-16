# .NET — Intermediate

EF, auth, cache, real API habits.

---

## Q1. What is EF change tracking?

**Answer:**

When I load an entity, EF remembers its original values. On `SaveChanges` it writes only what changed. `AsNoTracking()` skips that for read-only lists, which means less memory and faster queries.

In plain words, EF (Entity Framework, the library that turns my C# objects into database rows) keeps a "before" photo of every row I load. When I call `SaveChanges`, it compares the "before" photo with the object as it is now, and it sends an `UPDATE` only for the columns that are different. I don't have to write the SQL or remember what I touched.

That photo costs memory and time. If I load 500 rows just to show them in a table, I'm never going to save them, so keeping photos is wasted work. That's when I add `AsNoTracking()`. It's like reading a library book versus checking it out: if I'm only reading, nobody needs to write my name in the register.

**Cross-question:** `Update()` vs changing a tracked property?

**Cross-answer:**

If the entity is already tracked, I just set the property. `Update()` marks **all** fields as dirty, so it can overwrite columns I did not mean to touch.

For example, if I load a user and set `user.Email = "new@mail.com"`, EF sees that only the email changed and updates only that column. If I instead build a user object from a request and call `Update()`, EF can't tell what changed, so it writes every column. Any field the request left empty can wipe out real data in the database.

---

## Q2. What is an N+1 query?

**Answer:**

Say I load 50 parents, and then for each parent I run a query for its children. That is 51 trips to the database. The fix is `Include`, or better, a `Select` into a DTO. Sometimes I use split queries if a join blows up the number of rows.

The name comes from "1 query for the list, plus N queries for the details". It's like going to the shop 51 times to buy 51 items instead of taking one list and going once. Each trip to the database has a fixed cost, so the page gets slower with every extra row, and it often only shows up when real data grows.

`Include` tells EF to bring the children in the same query. A `Select` into a DTO (a small class with only the fields the screen needs) is often better, because EF builds one query that fetches just those columns. Split queries (`AsSplitQuery()`) mean EF sends one query for the parents and one for each included collection, instead of one giant join.

**Cross-question:** Can `Include` be worse than N+1?

**Cross-answer:**

Yes. Including two collections in one query can duplicate the parent rows a lot. Then split queries, or two simple queries, can be cheaper. I look at the SQL to decide.

Here's why: if one order has 10 items and 10 payments, a single join returns 10 × 10 = 100 rows, and the order's data is repeated in every one of them. With a few hundred orders, that becomes a huge amount of repeated data going over the wire. Two smaller queries avoid that multiplication.

---

## Q3. Who owns migrations?

**Answer:**

On many teams, people generate and apply migrations. I can change entities and Fluent config. I do not run schema updates unless that is the agreed process.

A migration is a small script that changes the database structure, like adding a column or a table, so it matches my C# classes. Fluent config is the C# code where I describe things like column lengths, keys and relationships.

The reason I'm careful is that the database is shared. If two people apply different migrations, or someone runs one against the wrong database, the schema and the code can get out of step, and that's painful to undo. So I make my entity and config changes, and the migration gets applied the way the team has agreed, whether that's a lead, a pipeline or a review step.

**Cross-question:** `EnsureCreated` in production?

**Cross-answer:**

No. It is for samples and throwaway tests. Real apps use migrations.

`EnsureCreated` just creates the database from the current model if it doesn't exist. It doesn't keep a history of changes, so later you can't move the schema forward step by step with migrations. That's fine for a demo you'll delete tomorrow, not for a database that holds real data.

---

## Q4. LINQ EF cannot translate?

**Answer:**

Things like my own C# methods, some date tricks, or loops inside `Where`. EF throws an error for those. I rewrite the filter, or I filter in memory after SQL has already cut down the rows.

EF works by turning my LINQ into SQL. The database only understands SQL, so if I call my own C# method inside a `Where`, EF has no idea what SQL to write for it, and it throws an error instead of guessing.

The safe fix is to rewrite the condition using things EF can translate, like simple comparisons. If I really need my C# logic, I first let SQL do the heavy filtering, then bring the smaller result into memory and apply my method there. What I avoid is pulling the whole table into memory just to filter it in C#, because that loads every row over the network.

**Cross-question:** How do you see the SQL?

**Cross-answer:**

With logging in **dev**, or with `ToQueryString()`. I don't log parameter values that are secrets.

`ToQueryString()` gives me the SQL for one query as text, which is handy while debugging. Turning on EF logging in development shows every query that runs. I keep sensitive values out of logs, because logs get copied around and read by many people.

---

## Q5. How does JWT auth work on the API?

**Answer:**

The client sends `Authorization: Bearer ...`. The middleware checks the signature, issuer, audience, and expiry. The claims become `User`. Then `[Authorize]` runs.

A JWT (JSON Web Token) is like a signed ID card. The server gives it to the user at login, and the user shows it on every request. Middleware is code that runs on each request before it reaches my controller.

The middleware checks the card step by step: the signature (was it really made by us and not edited?), the issuer (who made it), the audience (is it meant for this API?) and the expiry (is it still valid?). If all is fine, the claims inside it, like user id, become the `User` my code can read. Then `[Authorize]` decides whether that user may call this endpoint. Without these checks, anyone could write their own token and pretend to be someone else.

**Cross-question:** Can I log the token?

**Cross-answer:**

Never. I can log the user id. The secret lives in user-secrets or a vault, not in git.

The token works like a key. Anyone who copies it from a log can call the API as that user until it expires. The user id tells me who did what without giving anyone access. The signing secret is even more sensitive, because with it someone could create valid tokens for any user, so it never goes into source control.

---

## Q6. Roles vs policies vs resource checks?

**Answer:**

A role is something like “Admin”. A policy is a named rule. A resource check asks “can this user edit **this** incident?”. That last one needs the actual record, not only the role.

Think of an office building. A role is your job title. A policy is a rule with a name, like "can approve refunds", which might check the role plus other things such as a claim or the user's department. A resource check is the question at a specific door: "is this person allowed into **this** room?"

The resource check matters because a title alone doesn't tell you which records belong to whom. To answer "can this user edit this incident", I have to load the incident and look at who owns it or which tenant it belongs to.

**Cross-question:** Why are roles not enough?

**Cross-answer:**

A user can be an admin in tenant A and nobody in tenant B. We also need checks on the specific object.

A tenant is one customer or organisation sharing the same app. If I only check "is this user an Admin?", an admin of company A could open company B's records just by changing an id in the URL. Checking the specific record stops that.

---

## Q7. Filters vs middleware?

**Answer:**

Middleware runs for every request. Filters run only around MVC actions, and they can see model state and arguments.

Middleware is like the security desk at the building entrance: everyone passes it, whether they're going to a controller, a static file or a health check. Filters are like a check at a particular office door: they only run when a request reaches a controller action.

Because filters run that close to the action, they know more about it. They can see the action's arguments and the model state (whether the incoming data passed validation). So I use middleware for things every request needs, like logging or error handling, and filters for things tied to actions, like validation rules.

**Cross-question:** Global exception middleware or exception filter?

**Cross-answer:**

I like middleware, because it gives one consistent error JSON. Filters miss some paths.

An exception filter only catches errors thrown inside MVC actions. Errors from other middleware, routing or non-MVC endpoints never reach it. Exception middleware sits near the start of the pipeline, so it catches errors from everything after it, and the client always gets the same error shape.

---

## Q8. ProblemDetails?

**Answer:**

It is a standard error JSON with title, status, and detail. It is fine for 400s. The detail should never be a stack trace or a SQL message.

Without a standard, every endpoint invents its own error format, and the frontend has to handle many shapes. ProblemDetails gives every error the same fields, so the client always knows where to find the status and the message.

The reason to keep stack traces and SQL messages out is security. They show table names, file paths and code structure, which helps an attacker. They also mean nothing to a normal user. The full details go in the server logs, where developers can read them.

**Cross-question:** Domain rule fail vs a bug?

**Cross-answer:**

A rule failure gets a 400/409 with a stable code. A bug gets a 500: I log it on the server and send generic text to the client.

A rule failure is something like "you can't book a date in the past". The user can fix it, so I return a clear message and a stable code, like "BOOKING_DATE_PAST", that the frontend can check for. 409 means conflict, for example the record was already changed. A bug is our mistake, so the client just gets "something went wrong", and the real error goes to our logs for us to fix.

---

## Q9. Caching?

**Answer:**

Memory cache lives in this process only. Distributed cache (Redis) is shared by all servers. HTTP caching uses headers so the browser or a proxy can cache.

A cache is like keeping your most-used files on your desk instead of walking to the cupboard each time. It saves repeated work, like calling the database for the same list of countries again and again.

The three types differ in where the "desk" is. Memory cache is inside one running app, so it's very fast, but if I have three servers, each one has its own copy and they can disagree. Redis is a separate shared store, so all servers see the same cached data. HTTP caching lets the browser or a proxy keep the response, based on headers like `Cache-Control`, so the request may never reach my server at all.

**Cross-question:** Multi-tenant cache key?

**Cross-answer:**

The key must include the tenant id. A cache without the tenant in the key is a data leak.

Say I cache a dashboard under the key `dashboard`. Company A loads it first, and it gets cached. Then company B asks for `dashboard` and gets company A's numbers. A key like `dashboard:{tenantId}` keeps each customer's data separate.

---

## Q10. Why `IHttpClientFactory`?

**Answer:**

It manages the handlers for me, so we don't run out of sockets and DNS doesn't go stale. Typed clients are clean, like `GitHubClient(HttpClient http)`.

Here's the problem it solves. If I create a new `HttpClient` for every call, each one opens its own connections, and closed connections hang around for a while, so under load the server can run out of sockets (the network connections). If I keep a single `HttpClient` forever, it can keep using an old DNS result and miss that the other service moved to a new address.

The factory sits in the middle. It pools and reuses the underlying handlers and replaces them every so often, so I get reuse without stale DNS. A typed client is a class that receives an `HttpClient` already set up for one API, so the rest of my code just calls clear methods like "get this repo" instead of building URLs everywhere.

**Cross-question:** Typed vs named client?

**Cross-answer:**

Typed is nicer when it's for one API. Named is `CreateClient("github")`.

A named client is set up with a name, and I ask the factory for it by that name string. It works, but a typo in the name only fails at runtime. A typed client is injected like any other service, and it wraps all calls to that API in one class, which is easier to read and test.

---

## Q11. Transactions and two `SaveChanges`?

**Answer:**

One `SaveChanges` is usually one transaction. With two saves, if the second one fails, the data can be left half-written. I prefer one save at the end, or an explicit transaction around both.

A transaction is an all-or-nothing package. Think of a bank transfer: taking money from one account and adding it to another must both happen, or neither. If I call `SaveChanges` once to debit and once to credit, and the second call fails, the money has left one account and never arrived in the other.

So I make all the changes first and call `SaveChanges` once. If I really need two saves, for example because I need an id generated by the first, I wrap both in an explicit transaction, so a failure rolls back both.

**Cross-question:** Background job plus request in one transaction?

**Cross-answer:**

Usually no, because they run in different scopes. If both must happen together, I use an outbox.

A background job runs later, with its own `DbContext`, so it can't share the request's transaction. The outbox pattern means that in the same transaction as my main change, I also save a row saying "this work still needs to be done". A background process reads those rows and does the work. If the main change fails, the outbox row is never saved, so nothing runs for a change that didn't happen.

---

## Q12. Health checks?

**Answer:**

A `/health` endpoint can ping the database. Orchestrators use it.

A health check is like a nurse taking your pulse: a quick way to ask "are you OK?". An orchestrator is the system that runs and manages my app's containers, like Kubernetes, and load balancers use the same idea.

They call `/health` regularly. If it fails, they can stop sending traffic to that copy of the app or restart it. Without it, a broken server could keep receiving requests and return errors to users until someone notices.

**Cross-question:** Liveness vs readiness?

**Cross-answer:**

Liveness asks: is the process stuck? If yes, restart it. Readiness asks: can it take traffic? An app that is still starting can be alive but not ready.

Think of a shop. Liveness asks "is anyone inside at all?". Readiness asks "is the door open for customers?". During startup the app may still be warming up or connecting to the database, so it's alive but should not get requests yet. Keeping the two checks separate stops the orchestrator from restarting an app that just needs a few more seconds.

---

## Q13. SignalR in simple words?

**Answer:**

Live messages over WebSockets. The server has a hub, and it calls methods on the client. With more than one server you need a backplane like Redis, or messages get lost between nodes.

Normally the browser asks and the server answers. SignalR lets the server push messages to the browser whenever it wants, like a new chat message or a status change, without the page refreshing or asking again. A WebSocket is a connection that stays open both ways. A hub is the class on the server where I write those live methods.

The backplane matters when there are several servers. If user A is connected to server 1 and user B to server 2, a message sent on server 1 won't reach B by itself. A backplane like Redis passes messages between servers, so everyone gets them.

**Cross-question:** How do you auth a SignalR connection?

**Cross-answer:**

It is the same user as for HTTP. I authorize hub methods and never broadcast another tenant’s data. I treat the token as a secret, even if it ends up in a query string for the browser.

The browser's WebSocket API can't set an `Authorization` header, so the token is often sent in the query string when connecting. Query strings can end up in server logs, so I'm careful not to log them. Inside the hub, I check permissions on each method and send messages only to groups the user belongs to, so one tenant never sees another tenant's updates.

---

## Q14. API versioning?

**Answer:**

Either `/v1/...` in the URL or a header. Adding a field is usually ok. Renaming a field or changing its meaning needs a new version.

Other apps, like a mobile app or a partner system, are built against my API as it is today. If I change it under them, they break, and I can't force everyone to update on the same day. Versioning lets old clients keep using `/v1` while new clients move to `/v2`.

Adding a field is usually safe, because most clients ignore fields they don't know. Renaming a field is not safe: the old client still looks for the old name, finds nothing, and shows blanks or crashes. Changing what a field means is even sneakier, because nothing errors and the client quietly shows the wrong thing.

**Cross-question:** `int` to `string` — breaking?

**Cross-answer:**

Yes. Clients will break.

A client that expects `"age": 30` and suddenly gets `"age": "30"` may fail to read the JSON, or its maths and comparisons stop working. Even though the value looks the same to a person, the type is part of the contract.

---

## Q15. Soft delete vs hard delete?

**Answer:**

Soft delete means an `IsDeleted` flag, and a global filter hides those rows. History stays, but things like unique emails get messy. Hard delete means the row is really gone. I pick per table.

Soft delete is like moving a file to the recycle bin instead of shredding it. The row stays, it's just marked as deleted, and the app hides it. That keeps history, lets me undo a mistake, and keeps old reports correct.

The cost is that the row still counts in the database. If emails must be unique and a deleted user still has `a@b.com`, a new user can't sign up with that email. So I decide per table: soft delete for things like bookings or payments where history matters, hard delete for things like temporary data that has no value later.

**Cross-question:** `IgnoreQueryFilters()`?

**Cross-answer:**

Only for admin screens or jobs that must see deleted rows. Never on a public get-by-id without extra checks.

`IgnoreQueryFilters()` turns off all global filters for that query. That includes the soft delete filter, and also any tenant filter. If I use it on a public endpoint, someone could fetch deleted records, or even other tenants' records, just by guessing ids.

---

## Q16. What is the options pattern validation?

**Answer:**

I bind config to a class and fail at startup if required values are missing. That is much better than a null reference at noon on Monday.

The options pattern means I read a section of config, like the email settings in `appsettings.json`, into a C# class, instead of reading raw strings all over the code. Validation means I mark which values are required, or what range they must be in, and the app checks them when it starts.

Why it matters: if a setting is missing and nothing checks it, the app starts fine and then crashes later, the first time some user hits that feature, maybe during the busiest hour. With validation, the app refuses to start and tells me exactly which value is missing, so I find the problem during deployment, not from a user complaint.

**Cross-question:** Can options hold a secret?

**Cross-answer:**

The object can, but I still don't log it. I load it from a secret store.

An options class can hold a password or API key like any other value. The value itself should come from user-secrets in development or a vault in production, not from a file in git. And I never print the whole options object to logs, because the secret would go with it.

---

## Q17. `AsNoTrackingWithIdentityResolution` — in simple words?

**Answer:**

No tracking, but if the same person appears twice in the result graph, I get one object, not two copies. It is useful when reading object graphs.

An object graph is a set of objects linked together, like orders that each point to a customer. Say I load 10 orders and 5 of them belong to the same customer. With identity resolution, all 5 orders point to one shared customer object. EF does this by remembering, only for that one query, which keys it has already built.

That saves memory and keeps the data consistent. I still don't pay for full change tracking, because I'm only reading.

**Cross-question:** Default `AsNoTracking`?

**Cross-answer:**

With that, the same key can turn into two separate instances. That can surprise you when you compare references.

In the example above, plain `AsNoTracking` may build a separate customer object for each order. They hold the same data, but `order1.Customer == order2.Customer` compares references (whether they are the same object in memory), so it can come out false. Changing one copy also won't change the other.

---

## Q18. Pagination — skip/take vs keyset?

**Answer:**

`Skip/Take` is easy, but it gets slow on big offsets. Keyset (“where id > lastId take 20”) stays fast. For admin pages, Skip is often fine.

Pagination means showing data a page at a time. With `Skip/Take`, to show page 1000 the database still has to walk past the first 20,000 rows and throw them away, like counting from the start of a book every time you want to reach a page.

Keyset pagination is more like a bookmark. I remember the last id I showed and ask for the next 20 after it. With an index on that column, the database jumps straight there, so page 1000 is as fast as page 1. The trade-off is that you can't jump to "page 57" directly, which is why `Skip` is still fine for small admin lists.

**Cross-question:** Should the API return the total count always?

**Cross-answer:**

A count can be expensive. I return it when the UI needs it, not out of habit.

To count, the database often has to go through every matching row, which on a big table can cost more than fetching the page itself. If the screen shows "Page 3 of 120", I need it. If it only has a "Load more" button, I can skip it.

---

## Q19. File upload in ASP.NET Core — what do you check?

**Answer:**

Size, type, and extension. I don't trust the content type the client sends. I store the file under a generated name, not the user's file name. And no `../` in paths.

Uploads are one of the easiest ways to attack a server, so I treat every file as suspicious. A size limit stops someone from filling the disk with huge files. Checking the type and extension blocks files I don't expect, like a script dressed up as a picture.

The content type is just a label the client sends, and anyone can fake it, so I check the file's extension and, where it matters, the actual bytes. I save files under a name I generate, like a GUID (a random unique id), because user file names can clash, contain odd characters, or include `../`. That `../` means "go up one folder", and it could let someone write a file outside the upload folder.

**Cross-question:** Save inside `wwwroot`?

**Cross-answer:**

Only if it must be public. Private files go outside the web root, and users get them through a download action that checks permission.

Anything in `wwwroot` can be downloaded by anyone who knows or guesses the URL, with no login. That's fine for a public logo. For something like an ID document, I store it elsewhere and serve it through a controller action that first checks who is asking.

---

## Q20. What is CORS preflight?

**Answer:**

For some cross-origin calls, the browser first sends an `OPTIONS` request. The API must answer with the right allow headers. If that fails, the real POST from JS never runs.

An origin is the scheme, domain and port together, like `http://localhost:4200`. By default, browsers stop JavaScript on one origin from calling an API on another, to protect users from shady websites. CORS (Cross-Origin Resource Sharing) is how the API says "I allow calls from this origin".

For requests that aren't simple, for example ones sending JSON or an `Authorization` header, the browser first asks permission with an `OPTIONS` request. It's like knocking before you enter. If the API's answer doesn't allow that origin, method or header, the browser blocks the real request, and you just see a CORS error in the console.

**Cross-question:** Why does Swagger work but Angular does not?

**Cross-answer:**

Swagger is often on the same origin. Angular on another port counts as another origin.

Swagger UI is usually served by the API itself, so its calls are same-origin and CORS doesn't apply. Angular in development often runs on `localhost:4200` while the API runs on another port. A different port alone makes it a different origin, so the API needs a CORS policy that allows the Angular address.

---

## Q21. Cookie auth vs JWT?

**Answer:**

With cookies, the browser sends them on its own, so CSRF is a concern. With a JWT in a header, Angular attaches it, so CSRF is less of a risk, but XSS can steal a token kept in storage. Both need HTTPS.

CSRF (cross-site request forgery) is when a bad website gets your browser to send a request to a site you're logged into. Because the browser attaches cookies by itself, that request arrives with your login, and the server thinks you sent it. A JWT in a header has to be added by my own code, so another site can't make the browser send it.

XSS (cross-site scripting) is when an attacker manages to run their own JavaScript on my page. That script can read anything JavaScript can read, including a token in `localStorage`. HTTPS is needed either way, because without it the cookie or token travels as plain text and anyone on the network can read it.

**Cross-question:** JWT in localStorage?

**Cross-answer:**

It is common, and risky if you have XSS. An HttpOnly cookie is the other approach. I never put the token in the URL if I can help it.

An HttpOnly cookie can't be read by JavaScript at all, so an XSS script can't steal it, though then CSRF protection is needed again. URLs are the worst place for a token, because they get saved in browser history, server logs and proxy logs.

---

## Q22. What does `SaveChanges` return?

**Answer:**

The number of state entries written to the database. I don't use that as my only success check for business rules.

A state entry is roughly one tracked object that was added, changed or deleted. If I add an order and two order lines, it returns 3.

That number tells me how many rows the database touched. It doesn't tell me whether the action made sense: whether the user was allowed, whether the record was the right one, or whether a business rule was met. Those checks happen in my code before I save.

**Cross-question:** 0 rows updated — success?

**Cross-answer:**

Maybe the id was wrong. I check that the entity existed and that the user was allowed to change it.

Zero isn't an error by itself, so nothing throws. It might mean nothing actually changed, or that the record I meant to update wasn't found. So I load the entity first: if it's missing I return a 404, and if the user can't touch it I return a 403, instead of quietly saying "OK".

---

## Q23. Global query filters?

**Answer:**

It is a default `Where` added to every query, like `!IsDeleted`. The catch is that it is easy to forget about when you really do need the hidden rows.

I set it once in the model config, and from then on EF adds that condition to every query on that entity for me. So nobody has to remember to write `Where(x => !x.IsDeleted)` everywhere, and nobody forgets it on a new screen.

The catch works the other way too. Because the filter is invisible in the query code, someone looking for deleted records can get confused about why they're "missing". To see them, the query has to use `IgnoreQueryFilters()` on purpose.

**Cross-question:** Filter by tenant in a shared database?

**Cross-answer:**

Yes, a filter on `TenantId` helps, but I still don't trust the tenant id the client sends. I set the tenant from the server context.

If the tenant id came from the request, a user could just send another company's id and the filter would happily show that company's data. So the tenant comes from something the server controls, like the claims in a validated token or the server-side session.

---

## Q24. How do you handle concurrency (two users edit the same row)?

**Answer:**

I use a row version (`xmin` / timestamp). The second save gets a concurrency exception, and I show “someone else changed this, please reload”.

Picture two people editing the same document. Both open it, both make changes, and whoever saves second quietly wipes out the first person's work. A row version is a value that changes every time the row is saved. In PostgreSQL `xmin` is a built-in system column that does this, and in SQL Server a `rowversion` (timestamp) column does the same job.

When I load the row, I also get its version. When I save, EF adds "and the version is still what I loaded" to the `UPDATE`. If someone else saved in between, the version no longer matches, no row is updated, and EF throws a `DbUpdateConcurrencyException`. I catch that and ask the user to reload, so nobody's changes are lost without them knowing.

**Cross-question:** Last write wins?

**Cross-answer:**

That's fine for some fields, but bad for money and status. I pick per feature.

For something like a profile bio, if two edits clash, keeping the latest is usually fine. For a wallet balance or a booking status, losing an update can mean wrong money or a booking in the wrong state, so there I use the concurrency check.

---

## Q25. What is compiled query in EF?

**Answer:**

EF caches the translation of a query you run a lot. It helps on hot paths. I don't compile every random LINQ line.

Every time a LINQ query runs, EF has to work out what it means and which SQL it maps to. A compiled query, made with `EF.CompileQuery`, does that work once up front and gives me a ready-to-call function, so later calls skip most of that step.

A hot path is code that runs very often, like an endpoint hit thousands of times a minute. There, saving a little time per call adds up. For a query that runs a few times a day, the gain is too small to be worth the extra code.

**Cross-question:** First call still slow?

**Cross-answer:**

Yes. You pay the compile cost once. After that it is cheaper.

It's like setting up a coffee machine: the first cup takes longer because of the setup, and every cup after that is quick. Compiling doesn't remove the work, it just does it once instead of on every call.

---

## Q26. Rate limiting?

**Answer:**

It stops one client from flooding the API. ASP.NET has middleware for this. I pair it with auth, so a public login endpoint cannot be guessed at all day.

Rate limiting is like a bouncer letting only so many people in per minute. I set rules such as "each user or IP gets 100 requests a minute", and anything above that gets rejected for a while.

Without it, one buggy client in a loop, or one attacker, can use up the server and slow it down for everyone. On a login endpoint it also matters for security: without a limit, someone can try thousands of passwords a minute until one works. With a limit, that kind of guessing becomes far too slow to be useful.

**Cross-question:** 429 meaning?

**Cross-answer:**

Too many requests, retry later. Clients should back off.

429 is the HTTP status code for "Too Many Requests". Backing off means the client waits before trying again, and waits a bit longer after each failure, instead of hammering the API straight away. The server can also send a `Retry-After` header saying how long to wait.

---

## Q27. How does soft delete work in NriCare?

**Answer:**

Entities implement `IBaseEntity`, which has `IsDeleted`. In `SaveChanges`, a hard delete is turned into an update that sets `IsDeleted = true`. A global query filter hides those rows. The audit fields `CreatedBy` and `UpdatedBy` come from `IUserSession`.

So when any code calls `Remove()` on an entity, it isn't really deleted. Our `SaveChanges` override catches it before it reaches the database and turns it into "mark as deleted". This happens in one place, so no developer has to remember to do soft delete by hand.

The global filter then hides those rows from normal queries, so the rest of the app behaves as if they're gone. `IUserSession` holds who the current user is, so every save also records who created or last changed the row, without each feature writing that code itself.

**Cross-question:** How do you still load a deleted wallet?

**Cross-answer:**

I use `IgnoreQueryFilters()` plus `!x.IsDeleted` when I really mean “include filtered types but still skip deleted”. Booking payout does this for wallets across associations.

`IgnoreQueryFilters()` switches off all the global filters at once, including the association filter and the soft delete filter. In booking payout we need wallets from other associations, so we turn the filters off, but we still don't want deleted wallets. Adding `!x.IsDeleted` back by hand keeps that one rule while lifting the others.

---

## Q28. How do you stop one association seeing another association’s data?

**Answer:**

It is not a separate database. Entities like Booking, NRUser, and ServiceProvider implement `IMainAssociationEntity`. The global filter is `MainAssociationId == current association`. SuperAdmin sets `DisableMainAssociationFilter`. On insert, SaveChanges stamps the association id from the session.

All associations share one database, like several companies sharing one office building. Each row carries a `MainAssociationId` that says which association owns it, like a label on a filing box.

The global filter adds `MainAssociationId == current association` to every query on those entities, so a user only sees their own association's rows, even if a developer forgets to filter. When a new row is saved, `SaveChanges` fills in the association id from the session, so the label is always set by the server. Only SuperAdmin, who needs to see everything, turns the filter off.

**Cross-question:** Can the Angular app send `MainAssociationId` and switch tenant?

**Cross-answer:**

It should not win. The server session is the source of truth. SuperAdmin is the only role that turns the filter off.

Anything the browser sends can be edited by the user, so it can't decide which tenant they belong to. Even if a request includes another association's id, the filter and the insert stamping both use the value from the server session, not the one in the request.

---

## Q29. JWT claims in NriCare — what is inside?

**Answer:**

Custom claims, not ASP.NET Identity roles: `user_id`, `session_id`, `user_access_type`, plus ids for service provider, association, NR user, and verifier. The access token lives for about 20 minutes. `ClockSkew` is zero, so expiry is strict.

Claims are facts written inside the token, like fields on an ID card. Because the token carries the user's id, access type and related ids, the API knows who is calling and which records relate to them without looking that up again on every request.

A short life of about 20 minutes limits the damage if a token is stolen, because it stops working soon. `ClockSkew` is a grace period ASP.NET normally allows for small clock differences between servers. With it set to zero, the token stops working at its exact expiry time, not a few minutes later.

**Cross-question:** `[Authorize(Roles = "Admin")]`?

**Cross-answer:**

We don't use that. We put `[Authorize]` on the controller, and then the code checks `UserAccessType`. If the role matrix grew, I would add policies.

`[Authorize]` on the controller makes sure the caller has a valid token. Which kind of user they are is read from the `UserAccessType` claim in code. A role matrix is the table of "who can do what". If it got bigger, named policies would keep those rules in one place instead of spread across many methods.

---

## Q30. How do you paginate lists?

**Answer:**

We have an extension, `ApplyPaginationAsync`, on `IQueryable`. Search, date, and skip/take stay on the query, so EF turns them into SQL. Wallet transactions and bookings use this. I don't `ToList` the whole table and then page in memory.

`IQueryable` is a query that hasn't run yet. As long as I keep adding search, date filters and `Skip/Take` to it, EF folds them all into one SQL statement, and the database sends back only the one page I asked for.

If I called `ToList` first, the whole table would come into the app's memory and I'd cut out 20 rows in C#. That works with 100 rows and falls over with a million. Having one shared extension means every list does paging the same, correct way.

**Cross-question:** Why `CreatedDate.Date` in some queries?

**Cross-answer:**

It is easy to read, but it can stop the database from using an index because of the `.Date` conversion. For hot paths I would compare against a UTC start/end instead.

An index is like the index at the back of a book: it lets the database jump straight to matching rows. If the query wraps the column in a conversion like `.Date`, the database often has to convert every row before comparing, so it can't use the index. Writing `CreatedDate >= start && CreatedDate < end` compares the raw column, so the index still works.

---

## Q31. How do you map entities to DTOs?

**Answer:**

Most list APIs use a LINQ `Select` into a DTO, so EF does the projection in SQL. Mapster is registered, but we barely use it. I prefer `Select` for lists so we don't load full graphs.

A DTO (Data Transfer Object) is a simple class with only the fields the client needs. Mapping means copying data from the database entity into that class. Projection means choosing which columns to fetch.

When I write a `Select` that creates a new DTO with just a few fields, EF builds SQL that fetches only those columns. If I loaded full entities and mapped them afterwards, EF would fetch every column and maybe related data too, just to throw most of it away. Mapster is a mapping library, but for lists the plain `Select` keeps the query small and easy to see.

**Cross-question:** Returning the Booking entity?

**Cross-answer:**

No. It has the fee breakdown, holds, chat, and histories. That is a huge JSON and it can loop. DTO only.

The loop happens because entities point to each other: a booking has a user, the user has bookings, and each booking has a user again. The JSON serializer can keep following those links until it errors. Returning the entity also exposes internal fields the client should never see. A DTO sends exactly what the screen needs.

---

## Q32. File uploads — where do files go?

**Answer:**

To DigitalOcean Spaces, through the AWS S3 SDK. `FileUploadHelper` stores public or private objects. Images can get a thumbnail. Private files are served with a short-lived presigned URL. The `File` row in Postgres holds the metadata.

Spaces is file storage that speaks the same language as Amazon S3, so we can use the AWS S3 SDK to talk to it. Keeping files there instead of on our own server means they survive server restarts and don't fill up the app's disk.

A presigned URL is a link with a built-in signature and expiry time, like a guest pass that works for a few minutes. The user can download the private file directly with it, and once it expires, the link is useless if someone shares it. The `File` row in Postgres records details like the name, type and where it's stored, so the app can find and list files without asking Spaces.

**Cross-question:** If S3 succeeds and the DB insert fails?

**Cross-answer:**

You can end up with an orphan object in Spaces. I log that. A cleanup job or delete-on-failure would be the improvement.

These are two separate systems, so one transaction can't cover both. If the file is uploaded but the database row isn't saved, the file sits in storage with nothing pointing to it. That's an orphan: it costs storage and nobody can find it. Deleting the file when the insert fails, or a job that clears out files with no matching row, would tidy that up.

---

## Q33. How does OTP work in your signup?

**Answer:**

There are anonymous endpoints for request, verify, and resend. We store a 6-digit code in `OtpVerification` with a 5-minute expiry. Login itself is phone + password, not OTP. OTP is only for signup verification.

OTP means one-time password: a short code that works once and only for a few minutes. It proves the person signing up really has that phone number. The endpoints are anonymous because the user doesn't have an account yet, so they can't be logged in.

The 5-minute expiry means an old code can't be reused later. After signup, the user logs in with their phone number and password as normal. The OTP is just the one-time check at the start.

**Cross-question:** How is the SMS sent?

**Cross-answer:**

A real SMS provider is not wired up yet. Right now the verify API returns the OTP in the message for testing. I would not ship that. Production needs SMS/email and no OTP in the response.

Returning the code in the API response makes testing easy, but it defeats the point. Anyone could sign up with someone else's phone number and read the code straight from the response. In production, the code must reach the user only through SMS or email, so only the real owner of that number or address sees it.

---

## Q34. Why is almost every repository scoped?

**Answer:**

They need `DbContext` and `IUserSession`, which are per request. If I made `BookingRepository` a singleton, it would hold on to one context and one user for the whole app.

Scoped means a new instance is created for each HTTP request and thrown away at the end. Singleton means one instance is shared by the whole app for as long as it runs.

`DbContext` isn't safe to share between requests running at the same time, and `IUserSession` holds the current user. If the repository were a singleton, it would grab the first request's context and user and keep them forever. Every later request would then use the wrong user's session, and the shared context could break when two requests use it at once. Scoped keeps each request's data separate.

**Cross-question:** Background job then?

**Cross-answer:**

TickerQ jobs create a **new scope**, get `ApplicationDbContext` from that scope, and then dispose it. Same rule as with Hangfire.

A background job isn't part of an HTTP request, so there's no request scope for it to use. The job creates its own scope, which acts like a mini request: it gets a fresh `ApplicationDbContext`, does its work, then disposes the scope so the context and its connection are released.


---
## Additional Questions
### Q26. Tracking vs AsNoTracking?
**Answer:** Tracking is useful when I'm going to update the entity. For read-only lists, AsNoTracking cuts the tracking overhead and memory.

With tracking, EF keeps a copy of each loaded row's original values so it can work out what changed when I save. That's what I want when I'm loading something to edit it. For a list I'm only going to display, that copy is wasted effort, so `AsNoTracking()` makes the query lighter and faster.

### Q27. How do you avoid an N+1 query?
**Answer:** First I check the SQL. Usually I project straight into a DTO with Select, or I use Include carefully. I avoid querying children inside a loop.

N+1 means one query for a list, then one more query for each item in it, so 50 items become 51 database trips. Looking at the SQL in the logs shows it straight away as the same query repeated many times. A `Select` into a DTO gets everything in one query with only the columns I need. `Include` also works, but including several collections can blow up the number of rows, so I use it with care.

### Q28. When would you use keyset pagination?
**Answer:** For very large tables where Skip gets expensive. Instead of skipping thousands of rows, I ask for the records after the last id or timestamp I've seen.

It works like a bookmark in a book. With `Skip`, the database has to count past every earlier row to reach page 500. With keyset, I say "give me 20 rows after id 10,240", and with an index on that column the database jumps straight there. It suits infinite scroll and big feeds, where users go forward page by page rather than jumping to a page number.
