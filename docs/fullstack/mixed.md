# Full-stack mix

These join C#, .NET, Angular, and JavaScript. Interviewers use them to see if you can think across the stack.

Answers are in simple spoken English.

---

## Q1. Walk me through a login from the Angular button to the database.

**Answer:**

The user types an email and password. Angular checks that the fields aren't empty, then `HttpClient` posts to `/api/auth/login` over HTTPS. The API looks up the user in that tenant's database, checks the password hash, and returns a token or sets a cookie. Angular keeps the session the way we agreed (memory, cookie, or storage). Later calls send the token. The API reads the user from the token, not from a user id in the request body.

**Cross-question:** Where do you *not* put the password?

**Cross-answer:**

Not in the URL, not in logs, not as an extra copy in localStorage, and not in error messages. The server hashes it. And I never log the token either.

---

## Q2. A list is empty in Angular but the API returns 200. How do you debug?

**Answer:**

I open the Network tab and look at the URL, the status, and the JSON. If the body has rows, the bug is in the mapping or an `*ngIf`. If the body is `[]`, I check the tenant, the filters, and the user. I don't start by rewriting the UI.

**Cross-question:** CORS error vs empty list?

**Cross-answer:**

A CORS error shows up in the console, and the call looks blocked. An empty list is a successful call that just has no rows. They're different problems.

---

## Q3. How do you keep Angular types in sync with C# DTOs?

**Answer:**

I treat the JSON as a contract. Either the names match, or there's one mapper. If the team uses OpenAPI, it can generate the types. I don't type everything as `any`. When the API changes a field and I own both sides, both change in the same PR.

**Cross-question:** C# `long` id in Angular?

**Cross-answer:**

JSON numbers are read as doubles in JS, so big ids can get rounded. If ids are 64-bit, I send them as strings.

---

## Q4. Validation: Angular vs FluentValidation vs both?

**Answer:**

Angular validation is for quick feedback to the user. The API is the real gate. I repeat the basic rules, like required and email, on both sides. I never skip server validation just because “the button is disabled”.

**Cross-question:** Disabled button as security?

**Cross-answer:**

No. Anyone can call the API directly with Postman.

---

## Q5. How do you upload a file from Angular to .NET?

**Answer:**

I use `FormData` with `HttpClient`. The API checks the size and type, saves the file under a generated name, and checks that the user is allowed to upload to that record. I don't trust the file name or the MIME type the browser sent.

**Cross-question:** Put the file in git? In `wwwroot`?

**Cross-answer:**

Not in git. `wwwroot` only if the file is meant to be public. Private files go through a download action that checks permission.

---

## Q6. Real-time: SignalR vs polling?

**Answer:**

Polling is simple but adds extra load. SignalR is live but has more moving parts, and it needs a backplane if you run more than one server. If the screen can be 30 seconds out of date, I poll. For chat or live status, I use SignalR.

**Cross-question:** Auth on the socket?

**Cross-answer:**

It's the same user as on HTTP. And the server must never push another tenant's events.

---

## Q7. Where should filtering happen — Angular or API?

**Answer:**

If the data is large or sensitive, filtering happens in the API. Angular can filter a small list it already downloaded, as a nice extra for the user. I don't download 50,000 rows just to hide them in the UI.

**Cross-question:** Tenant filter in Angular only?

**Cross-answer:**

Never. That's not security.

---

## Q8. Error handling end to end?

**Answer:**

On the API, I return ProblemDetails and no stack traces. In Angular, an interceptor shows a toast for 500, redirects on 401, and shows a clear message on 400. I log a correlation id on both sides so support can find the matching server log.

**Cross-question:** Show the SQL error in the toast?

**Cross-answer:**

No.

---

## Q9. How do you version an API the Angular app uses?

**Answer:**

Additive changes are the safest. For breaking changes, I add `/v2`, or add a new field next to the old one for a while. Angular ships after the API, or at the same time if we own both.

**Cross-question:** Two Angular versions in the wild?

**Cross-answer:**

Then the API has to keep supporting the old contract until clients upgrade. I don't break mobile users on a Friday.

---

## Q10. State: Angular signal vs .NET session vs database?

**Answer:**

The database is the source of truth. Angular signals hold UI state. The server session or cookie answers “who is logged in”. If I have more than one server, I don't keep something like the shopping cart only in an in-memory singleton on the API.

**Cross-question:** In-memory cache as source of truth?

**Cross-answer:**

It disappears on restart and it's different on each server. It's fine as a cache, but not as the real record.

---

## Q11. How do you test this stack without a full browser?

**Answer:**

For the API, I use WebApplicationFactory with a test database. For Angular, TestBed and HttpTestingController. Then a few e2e tests for login and one main flow. I don't write e2e tests for every field.

**Cross-question:** Mock the API in e2e only?

**Cross-answer:**

Then I never catch real contract bugs. I want at least one test running against a real local API.

---

## Q12. Performance: slow page. Frontend or backend?

**Answer:**

I start in the Network tab. If TTFB (time to first byte) is high, it's the server or SQL. If the download is huge, it's the payload or the bundle. If the Performance panel shows heavy CPU, it's the Angular list. I measure before I guess.

**Cross-question:** First fix?

**Cross-answer:**

The biggest number. Often that's N+1 SQL or a list without trackBy. It's not a new framework.

---

## Q13. How do you pass the logged-in user from Angular to a .NET background job?

**Answer:**

I don't pass a token into Hangfire and treat it as the only auth forever. I pass the user id and tenant id that the API already trusted, store the job under that tenant, and the worker opens that tenant's database. The job never trusts a raw id that came from the browser.

**Cross-question:** Run the job as the user?

**Cross-answer:**

I record who asked for it. The worker runs with a service identity plus those ids, and applies the same permission rules.

---

## Q14. CORS in local Angular (`localhost:4200`) + API (`localhost:5000`)?

**Answer:**

Those are different origins, so the API has to allow `localhost:4200` in dev. A proxy in `proxy.conf.json` can hide this during development by making calls look same-origin. In production it's often the same domain behind nginx.

**Cross-question:** `AllowAnyOrigin` to save time?

**Cross-answer:**

Not with credentials. I list the real origins.

---

## Q15. What do you put in git vs user secrets vs environment?

**Answer:**

Git gets the code, non-secret config keys, and example values. User secrets hold local connection strings. Production secrets go in server environment variables or a vault. The Angular environment file holds only the public API URL.

**Cross-question:** `appsettings.json` with a real password?

**Cross-answer:**

No.

---

## Q16. How would you design a dashboard number like “open incidents”?

**Answer:**

First I agree with the business on what the number means. Then I count it in SQL/EF with the tenant filter and return one DTO. Angular shows the number with a loading and an error state. I don't count in the UI from a partial list.

**Cross-question:** Cache the number?

**Cross-answer:**

Maybe for a few seconds, keyed by tenant. Stale cards confuse people, so I keep the TTL short.

---

## Q17. XSS from a field the user typed, shown in Angular, stored in .NET?

**Answer:**

I store the raw text (or sanitized text if the product needs HTML), and I encode it on output. Angular interpolation already encodes. I never use `innerHTML` for that field. The API doesn't return HTML unless it's a very clear, sanitized feature.

**Cross-question:** Markdown?

**Cross-answer:**

I render it with a sanitizer. No raw HTML from the user.

---

## Q18. How do you handle timezones?

**Answer:**

I store UTC. The API sends ISO-8601 with a Z or an offset. Angular shows it in the user's locale with `DatePipe`. I don't store “server local time”.

**Cross-question:** Birthday?

**Cross-answer:**

That's a date with no time, so I use `DateOnly` on the server. Don't convert it through UTC, or the day can shift.

---

## Q19. Feature flag: who owns it?

**Answer:**

The flag lives in config or a small table. The API can hide endpoints and Angular can hide routes, and I do both. A hidden button isn't enough if the API still works.

**Cross-question:** Flag in Angular environment only?

**Cross-answer:**

Users can flip it in DevTools. The API has to refuse the call.

---

## Q20. You have one day to ship MVP. What do you cut?

**Answer:**

I keep auth, the tenant check, one list, one create, and the empty and error states. I cut pretty charts, extra filters, perfect animations, and extra roles. And I write the risks down.

**Cross-question:** Skip tests?

**Cross-answer:**

If I can, I keep one isolation test and the main happy path. I don't skip auth checks.

---

## Q21. Explain your stack to a hiring manager who is not a coder.

**Answer:**

C# and .NET are the kitchen. Angular is the dining room. JavaScript is the language the dining room speaks. The database is the pantry. My job is to make sure an order from table 5 never gets table 6's food.

**Cross-question:** What is a DTO in that picture?

**Cross-answer:**

It's the plate we send out, not the whole pantry shelf.

---

## Q22. What is the biggest bug you have seen between UI and API?

**Answer:**

Pick a real one if you have it. Here's a safe example:

> The UI sent dates as local strings, but the API parsed them as UTC. So the filters were off by one day. We agreed on ISO UTC in the API and used a `DatePipe` on the screen.

**Cross-question:** How do you prevent it?

**Cross-answer:**

One date format in the contract, a shared example in the API docs, and a test that checks a known instant round-trips correctly.

---

## Q23. How do you secure a multi-tenant Angular + .NET app? Five bullets.

**Answer:**

1. HTTPS.
2. Auth on every API.
3. Tenant comes from the server context, not from the body.
4. DTOs, with no extra fields.
5. Angular hides UI, but the API still checks.

**Cross-question:** Sixth?

**Cross-answer:**

Don't log tokens, and don't put secrets in the Angular bundle.

---

## Q24. RxJS `switchMap` vs a .NET cancel token — same idea?

**Answer:**

Yes, in spirit. `switchMap` drops the old HTTP call. `CancellationToken` tells EF or HTTP work to stop. If the user types fast, I want both: Angular cancels the old request, Kestrel notices the abort, and EF stops.

**Cross-question:** Does Angular cancel reach EF automatically?

**Cross-answer:**

If the browser aborts, the request can be cancelled. But I still have to pass the token through the API action. I don't ignore it.

---

## Q25. Why might `ng serve` work and production `ng build` fail at runtime?

**Answer:**

Common reasons: a wrong API URL in environment.prod, hash routing vs server rewrite rules, the base href, the minifier exposing a bug, or a lazy chunk 404 because the path is wrong.

**Cross-question:** First place you look?

**Cross-answer:**

The browser console and the failed network request. Then `environment.prod` and the server's fallback to `index.html`.

---

## Q26. Walk me through NriCare login from the Angular button.

**Answer:**

The user types their phone number and password. Angular posts to `POST /api/Auth/login`. The API finds the user, checks the password with BCrypt, writes a `UserLoginActivity` row, and returns a JWT and a refresh token, plus flags like `isApprovedUser` and `hasActiveSubscription`. Angular stores the access token and sends `Authorization: Bearer` on later calls. The session claims drive the association filter. OTP is part of a separate signup flow, not this login.

**Cross-question:** Password in the log?

**Cross-answer:**

On login we log the phone number, not the password. We never log the JWT or the refresh token.

---

## Q27. NR user books a service from the app. What hits the API?

**Answer:**

The web or app sends a booking POST with the service provider, service type, and dates. The server checks that the user is an NR user, gets the price from `ServiceProviderServices`, and adds tax, association, platform, and verifier fees. Then it locks the wallet, creates a hold, opens a chat, and returns the booking number. Angular then shows the booking as Under Discussion along with the chat thread.

**Cross-question:** Is the provider paid at that moment?

**Cross-answer:**

No, it's only a hold. The payout happens in `ReleasePayment` when the job is completed (or after verifier approval).

---

## Q28. How do web and mobile share the same backend?

**Answer:**

It's one ASP.NET API. Web routes are under `api/...`, mobile under `api/app/...`, and common ones like `api/Auth`, OTP, and files are shared. There are two Swagger docs. Both use the same JWT and the same database. The Angular admin/web app is `NriCare.Web.UI`. That's why a booking created on mobile still shows up in the web association dashboard, as long as the association filter allows it.

**Cross-question:** Different tokens per client?

**Cross-answer:**

The token shape is the same. `session_id` is per login, not per “web vs app”. Logout should revoke that session's refresh token, but today logout only records the logout time.


---
## Additional Questions
### Q29. How do you handle a file upload end to end?
**Answer:** Angular sends FormData. The API checks authentication, authorization, size and type, stores the file in private or object storage, and returns a safe file id or URL. Downloads go through permission checks.
### Q30. How do you keep frontend and backend contracts aligned?
**Answer:** I define DTOs and API examples clearly, use generated clients or a shared OpenAPI spec where it makes sense, and test the important contracts. A field rename on the backend should not quietly break the Angular app.
### Q31. How would you debug a slow full-stack page?
**Answer:** I start with the browser's Network waterfall. Then I separate API time, SQL time, payload size and rendering time. I fix the biggest measured bottleneck first.
