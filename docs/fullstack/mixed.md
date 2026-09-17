# Full-stack mix

These join C#, .NET, Angular, and JavaScript. Interviewers use them to see if you can think across the stack.

Answers are in simple spoken English.

---

## Q1. Walk me through a login from the Angular button to the database.

**Answer:**

The user types email and password. Angular validates empty fields, then `HttpClient` posts to `/api/auth/login` over HTTPS. The API checks the user in that tenant’s database, verifies the password hash, and returns a token or sets a cookie. Angular stores the session the way we agreed (memory, cookie, or storage). Later calls add the token. The API reads the user from the token, not from a user id in the body.

**Cross-question:** Where do you *not* put the password?

**Cross-answer:**

Not in the URL, not in logs, not in localStorage as extra copy, not in error messages. Hash on the server. Never log the token either.

---

## Q2. A list is empty in Angular but the API returns 200. How do you debug?

**Answer:**

I open Network. I look at the URL, status, and JSON. If the body has rows, the bug is mapping or `*ngIf`. If the body is `[]`, I look at tenant, filters, and the user. I do not start by rewriting the UI.

**Cross-question:** CORS error vs empty list?

**Cross-answer:**

CORS fails in the console and the call looks blocked. Empty list is a successful call with no rows. Different problems.

---

## Q3. How do you keep Angular types in sync with C# DTOs?

**Answer:**

I treat the JSON as a contract. Same names, or one mapper. OpenAPI can generate types if the team uses it. I do not type everything as `any`. If the API changes a field, both sides change in the same PR when I own both.

**Cross-question:** C# `long` id in Angular?

**Cross-answer:**

JSON numbers are doubles. Big ids can round. I send ids as strings if they are 64-bit.

---

## Q4. Validation: Angular vs FluentValidation vs both?

**Answer:**

Angular is for instant UX. The API is the real gate. I duplicate required/email on both. I never skip the server because “the button is disabled”.

**Cross-question:** Disabled button as security?

**Cross-answer:**

No. Anyone can call the API with Postman.

---

## Q5. How do you upload a file from Angular to .NET?

**Answer:**

`FormData` + `HttpClient`. The API checks size and type, saves with a generated name, and checks the user can upload to that record. I do not trust the file name or the MIME type the browser sent.

**Cross-question:** Put the file in git? In `wwwroot`?

**Cross-answer:**

Not git. `wwwroot` only if it is public. Private files go through a download action that checks permission.

---

## Q6. Real-time: SignalR vs polling?

**Answer:**

Polling: simple, extra load. SignalR: live, more moving parts, needs a backplane if more than one server. I poll if the screen can be 30 seconds stale. I use SignalR for chat or live status.

**Cross-question:** Auth on the socket?

**Cross-answer:**

Same user as HTTP. Do not push another tenant’s events.

---

## Q7. Where should filtering happen — Angular or API?

**Answer:**

If the data is large or secret, the API. Angular can filter a small already-downloaded list for extra UX. I do not download 50,000 rows to hide them in the UI.

**Cross-question:** Tenant filter in Angular only?

**Cross-answer:**

Never. That is not security.

---

## Q8. Error handling end to end?

**Answer:**

API: ProblemDetails, no stack traces. Angular interceptor: show a toast for 500, redirect on 401, a clear message on 400. I log a correlation id on both sides so support can find the server log.

**Cross-question:** Show the SQL error in the toast?

**Cross-answer:**

No.

---

## Q9. How do you version an API the Angular app uses?

**Answer:**

Additive changes are safest. Breaking changes get `/v2` or a new field next to the old one for a while. Angular ships after the API, or together if we own both.

**Cross-question:** Two Angular versions in the wild?

**Cross-answer:**

The API must support the old contract until clients upgrade. I do not break mobile users on a Friday.

---

## Q10. State: Angular signal vs .NET session vs database?

**Answer:**

Database is the source of truth. Angular signals are UI state. Server session/cookie is “who is logged in”. I do not store the shopping cart only in a memory singleton on the API if I have more than one server.

**Cross-question:** In-memory cache as source of truth?

**Cross-answer:**

It vanishes on restart and differs per server. Fine as a cache, not as the record.

---

## Q11. How do you test this stack without a full browser?

**Answer:**

API: WebApplicationFactory plus a test database. Angular: TestBed and HttpTestingController. A few e2e tests for login and one main flow. I do not e2e every field.

**Cross-question:** Mock the API in e2e only?

**Cross-answer:**

Then I never see real contract bugs. I want at least one test against a real local API.

---

## Q12. Performance: slow page. Frontend or backend?

**Answer:**

Network tab: TTFB high → server/SQL. Download huge → payload or bundle. CPU in Performance panel → Angular list. I measure before I guess.

**Cross-question:** First fix?

**Cross-answer:**

The biggest number. Often N+1 SQL or a list without trackBy. Not a new framework.

---

## Q13. How do you pass the logged-in user from Angular to a .NET background job?

**Answer:**

I do not pass a token into Hangfire as the only auth forever. I pass user id and tenant id that the API already trusted, store the job in that tenant, and the worker opens that tenant’s database. The job does not trust a raw id from the browser.

**Cross-question:** Run the job as the user?

**Cross-answer:**

I record who asked. The worker uses a service identity plus those ids, with the same permission rules.

---

## Q14. CORS in local Angular (`localhost:4200`) + API (`localhost:5000`)?

**Answer:**

Different origins. The API must allow `localhost:4200` in dev. A proxy in `proxy.conf.json` can hide this in development by using the same origin. Prod is often same domain behind nginx.

**Cross-question:** `AllowAnyOrigin` to save time?

**Cross-answer:**

Not with credentials. I list the real origins.

---

## Q15. What do you put in git vs user secrets vs environment?

**Answer:**

Git: code, non-secret config keys, example values. User secrets: local connection strings. Server env or a vault: production secrets. Angular env: public API URL only.

**Cross-question:** `appsettings.json` with a real password?

**Cross-answer:**

No.

---

## Q16. How would you design a dashboard number like “open incidents”?

**Answer:**

Agree the meaning with the business. Count in SQL/EF with tenant filter. Return one DTO. Angular shows the number and a loading/error state. I do not count in the UI from a partial list.

**Cross-question:** Cache the number?

**Cross-answer:**

Maybe for a few seconds, keyed by tenant. Stale cards confuse people. I keep TTL short.

---

## Q17. XSS from a field the user typed, shown in Angular, stored in .NET?

**Answer:**

I store the raw text (or sanitized if the product needs HTML). I encode on output. Angular interpolation encodes. I never `innerHTML` that field. The API does not return HTML unless it is a very clear, sanitized feature.

**Cross-question:** Markdown?

**Cross-answer:**

Render with a sanitizer. No raw HTML from the user.

---

## Q18. How do you handle timezones?

**Answer:**

Store UTC. API sends ISO-8601 with Z or offset. Angular shows it in the user’s locale with `DatePipe`. I do not store “server local time”.

**Cross-question:** Birthday?

**Cross-answer:**

That is a date without time. `DateOnly` on the server. Do not convert it through UTC or the day can shift.

---

## Q19. Feature flag: who owns it?

**Answer:**

Config or a small table. API can hide endpoints. Angular can hide routes. Both. A hidden button is not enough if the API still works.

**Cross-question:** Flag in Angular environment only?

**Cross-answer:**

Users can flip it in DevTools. The API must refuse.

---

## Q20. You have one day to ship MVP. What do you cut?

**Answer:**

I keep: auth, tenant check, one list, one create, empty and error states. I cut: pretty charts, extra filters, perfect animations, extra roles. I write the risk down.

**Cross-question:** Skip tests?

**Cross-answer:**

I keep one isolation test and the main happy path if I can. I do not skip auth checks.

---

## Q21. Explain your stack to a hiring manager who is not a coder.

**Answer:**

C# and .NET are the kitchen. Angular is the dining room. JavaScript is the language the dining room speaks. The database is the pantry. I make sure an order from table 5 never gets table 6’s food.

**Cross-question:** What is a DTO in that picture?

**Cross-answer:**

The plate we send out. Not the whole pantry shelf.

---

## Q22. What is the biggest bug you have seen between UI and API?

**Answer:**

Pick a real one if you have it. A safe example:

> The UI sent dates as local strings. The API parsed them as UTC. Filters were off by one day. We agreed on ISO UTC and a `DatePipe` on the screen.

**Cross-question:** How do you prevent it?

**Cross-answer:**

One date format in the contract, a shared example in the API docs, and a test that a known instant round-trips.

---

## Q23. How do you secure a multi-tenant Angular + .NET app? Five bullets.

**Answer:**

1. HTTPS.
2. Auth on every API.
3. Tenant from server context, not from the body.
4. DTOs, no extra fields.
5. Angular hides UI; API still checks.

**Cross-question:** Sixth?

**Cross-answer:**

Do not log tokens. Do not put secrets in the Angular bundle.

---

## Q24. RxJS `switchMap` vs a .NET cancel token — same idea?

**Answer:**

Yes in spirit. `switchMap` drops the old HTTP call. `CancellationToken` tells EF/HTTP to stop. If the user types fast, I want both: Angular cancels the old request, Kestrel notices the abort, EF stops.

**Cross-question:** Does Angular cancel reach EF automatically?

**Cross-answer:**

If the browser aborts, the request can cancel. I still pass the token in the API action. I do not ignore it.

---

## Q25. Why might `ng serve` work and production `ng build` fail at runtime?

**Answer:**

Wrong API URL in environment.prod. Hash routing vs server rewrite. Base href. Minifier exposing a bug. Lazy chunk 404 because the path is wrong.

**Cross-question:** First place you look?

**Cross-answer:**

Browser console and the failed network request. Then `environment.prod` and the server’s fallback to `index.html`.

---

## Q26. Walk me through NriCare login from the Angular button.

**Answer:**

User types phone and password. Angular posts `POST /api/Auth/login`. API finds the user, checks BCrypt, writes `UserLoginActivity`, returns JWT + refresh token, plus flags like `isApprovedUser` and `hasActiveSubscription`. Angular stores the access token and sends `Authorization: Bearer` on later calls. Session claims drive the association filter. OTP is a separate signup flow, not this login.

**Cross-question:** Password in the log?

**Cross-answer:**

We log phone number on login, not the password. Never log the JWT or refresh token.

---

## Q27. NR user books a service from the app. What hits the API?

**Answer:**

Web/app booking POST with service provider, service type, dates. Server checks the user is an NR user, prices from `ServiceProviderServices`, adds tax / association / platform / verifier fees, locks the wallet, creates a hold, opens a chat, returns the booking number. Angular then shows Under Discussion and the chat thread.

**Cross-question:** Is the provider paid at that moment?

**Cross-answer:**

No. Only a hold. Payout is `ReleasePayment` when the job is completed (or after verifier approval).

---

## Q28. How do web and mobile share the same backend?

**Answer:**

One ASP.NET API. Web routes `api/...`, mobile `api/app/...`, common `api/Auth`, OTP, files. Two Swagger docs. Same JWT, same database. Angular admin/web is `NriCare.Web.UI`. That is why a booking created on mobile still shows in the web association dashboard, as long as the association filter allows it.

**Cross-question:** Different tokens per client?

**Cross-answer:**

Same token shape. `session_id` is per login, not per “web vs app”. Logout should revoke that session’s refresh token — today logout only records logout time.
