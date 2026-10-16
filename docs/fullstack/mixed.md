# Full-stack mix

These join C#, .NET, Angular, and JavaScript. Interviewers use them to see if you can think across the stack.

Answers are in simple spoken English.

---

## Q1. Walk me through a login from the Angular button to the database.

**Answer:**

The user types an email and password. Angular checks that the fields aren't empty, then `HttpClient` posts to `/api/auth/login` over HTTPS. The API looks up the user in that tenant's database, checks the password hash, and returns a token or sets a cookie. Angular keeps the session the way we agreed (memory, cookie, or storage). Later calls send the token. The API reads the user from the token, not from a user id in the request body.

A few words in plain terms. HTTPS means the data is encrypted on the way, so nobody on the same Wi-Fi can read the password. A password hash is a scrambled, one-way version of the password. The database never keeps the real password, so we scramble what the user typed and compare the two scrambled values. A token is like a wristband at a concert: once the gate checks your ticket, you show the wristband for the rest of the night instead of the ticket.

Why the last point matters: if the API trusted a user id sent in the body, anyone could change that number and act as someone else. The token is signed by the server, so the user can't change it without the server noticing.

**Cross-question:** Where do you *not* put the password?

**Cross-answer:**

Not in the URL, not in logs, not as an extra copy in localStorage, and not in error messages. The server hashes it. And I never log the token either.

URLs end up in browser history, server logs and proxy logs, so a password there gets copied to many places. localStorage can be read by any script running on the page, so a single injected script could steal it. The token gets the same care because, while it's valid, it works just like the password.

---

## Q2. A list is empty in Angular but the API returns 200. How do you debug?

**Answer:**

I open the Network tab and look at the URL, the status, and the JSON. If the body has rows, the bug is in the mapping or an `*ngIf`. If the body is `[]`, I check the tenant, the filters, and the user. I don't start by rewriting the UI.

The Network tab (in the browser's developer tools) shows every call the page makes and exactly what came back. A 200 status only means "the call worked", not "there was data". So the response body is the first fact I need, because it tells me which half of the app to look at.

It's like a parcel that arrives looking empty. First I check if the box itself was empty (the server sent nothing) or if the item fell out on the way to the shelf (the Angular code lost it). That one check cuts the search in half.

**Cross-question:** CORS error vs empty list?

**Cross-answer:**

A CORS error shows up in the console, and the call looks blocked. An empty list is a successful call that just has no rows. They're different problems.

CORS is the browser's rule about which websites may read answers from an API on another address. With a CORS error, the browser refuses to hand the answer to our code at all, so the fix is in the API's settings. With an empty list, everything worked, and the question is why the query found nothing.

---

## Q3. How do you keep Angular types in sync with C# DTOs?

**Answer:**

I treat the JSON as a contract. Either the names match, or there's one mapper. If the team uses OpenAPI, it can generate the types. I don't type everything as `any`. When the API changes a field and I own both sides, both change in the same PR.

A DTO (data transfer object) is a simple class that describes the shape of the data the API sends or receives. OpenAPI is a standard description of an API, and tools can read it and write the matching TypeScript types for us, so nobody copies them by hand.

Why it matters: if C# renames `fullName` to `name` and Angular still reads `fullName`, the screen quietly shows blanks, with no error. Using `any` hides this because TypeScript stops checking. With real types, the compiler points at every place that broke. It's like both sides of a deal signing the same written agreement instead of each remembering their own version.

**Cross-question:** C# `long` id in Angular?

**Cross-answer:**

JSON numbers are read as doubles in JS, so big ids can get rounded. If ids are 64-bit, I send them as strings.

JavaScript can only hold whole numbers exactly up to about 9 quadrillion (2^53 − 1). A C# `long` can go much higher. Past that limit, JS silently changes the last digits, so the app could ask for record ...808 when the real id was ...807. Sending the id as a string keeps every digit exactly as it is.

---

## Q4. Validation: Angular vs FluentValidation vs both?

**Answer:**

Angular validation is for quick feedback to the user. The API is the real gate. I repeat the basic rules, like required and email, on both sides. I never skip server validation just because “the button is disabled”.

FluentValidation is a .NET library where we write the rules for incoming data in one clear place on the server. Angular checks are there for comfort: the user sees "email is required" right away, without waiting for a round trip.

But the browser belongs to the user, and they can change anything in it. So the server must check again before saving. Think of a shop: the label on the shelf tells you the price (helpful), but the till is what actually decides what you pay.

**Cross-question:** Disabled button as security?

**Cross-answer:**

No. Anyone can call the API directly with Postman.

Postman is a tool that sends requests to an API without any web page in between. Someone can also just re-enable the button in DevTools. A disabled button only guides honest users. It doesn't stop anyone.

---

## Q5. How do you upload a file from Angular to .NET?

**Answer:**

I use `FormData` with `HttpClient`. The API checks the size and type, saves the file under a generated name, and checks that the user is allowed to upload to that record. I don't trust the file name or the MIME type the browser sent.

`FormData` is the browser's way of packing a file together with other form fields into one request. The MIME type is the label that says what kind of file it is, like "image/png".

Why the caution: the name and the label both come from the user's machine, so they can be fake. A file called `photo.png` might really be a script, and a name like `../../web.config` could try to write outside the upload folder. Saving under a name we generate ourselves avoids that, and the size limit stops someone filling the disk with one huge upload.

**Cross-question:** Put the file in git? In `wwwroot`?

**Cross-answer:**

Not in git. `wwwroot` only if the file is meant to be public. Private files go through a download action that checks permission.

Git is for source code, not user data. Anything in `wwwroot` is served to anyone who knows the address, with no login check. So a private document there is like leaving it on a public notice board. A download action lets the API check who is asking before it hands the file over.

---

## Q6. Real-time: SignalR vs polling?

**Answer:**

Polling is simple but adds extra load. SignalR is live but has more moving parts, and it needs a backplane if you run more than one server. If the screen can be 30 seconds out of date, I poll. For chat or live status, I use SignalR.

Polling means the app asks "anything new?" every few seconds, like a child asking "are we there yet?". Most answers are "no", which is wasted work. SignalR is a .NET library that keeps a connection open, so the server can push news the moment it happens, like getting a phone call instead of checking your inbox again and again.

A backplane is a shared channel (often Redis) between servers. Without it, a user connected to server A won't hear a message that was raised on server B.

**Cross-question:** Auth on the socket?

**Cross-answer:**

It's the same user as on HTTP. And the server must never push another tenant's events.

The live connection has to be logged in and checked just like a normal API call. A tenant is one customer organisation sharing the app. When the server sends a live update, it should only go to that tenant's users, for example by putting each tenant in its own group, so one company never sees another company's data.

---

## Q7. Where should filtering happen — Angular or API?

**Answer:**

If the data is large or sensitive, filtering happens in the API. Angular can filter a small list it already downloaded, as a nice extra for the user. I don't download 50,000 rows just to hide them in the UI.

Filtering on the server means the database does the work and only the matching rows travel over the network. That keeps the page fast, especially on a phone. It also means data the user isn't allowed to see never leaves the server.

Hiding rows in the browser is like posting someone every file in the office and asking them not to read the ones that aren't theirs. The data has still arrived, and anyone can look in the Network tab.

**Cross-question:** Tenant filter in Angular only?

**Cross-answer:**

Never. That's not security.

If the API returns every tenant's rows and Angular only hides the others, all of that data is still in the response. Anyone can read it in DevTools or call the API directly. The tenant filter has to be in the API query.

---

## Q8. Error handling end to end?

**Answer:**

On the API, I return ProblemDetails and no stack traces. In Angular, an interceptor shows a toast for 500, redirects on 401, and shows a clear message on 400. I log a correlation id on both sides so support can find the matching server log.

ProblemDetails is a standard JSON shape for errors (a title, a status, some detail), so every error looks the same. A stack trace is the list of code lines that failed. It helps developers but tells attackers how the app is built. An interceptor in Angular is code that sees every HTTP response in one place, so we don't repeat error handling on every screen.

The status codes: 500 means the server broke, 401 means "you're not logged in", 400 means "what you sent is wrong". A correlation id is like a ticket number. The user or support quotes it, and we find the exact matching entry in the server logs.

**Cross-question:** Show the SQL error in the toast?

**Cross-answer:**

No.

A SQL error can show table names, column names and sometimes data. That helps an attacker and confuses the user. I show a short, friendly message with the correlation id, and keep the real error in the server log.

---

## Q9. How do you version an API the Angular app uses?

**Answer:**

Additive changes are the safest. For breaking changes, I add `/v2`, or add a new field next to the old one for a while. Angular ships after the API, or at the same time if we own both.

An additive change means only adding things, like a new field. Old clients just ignore what they don't know, so nothing breaks. A breaking change is renaming or removing something, or changing what it means. Old clients still expect the old shape.

So the API goes first. If Angular went first, it would ask for things the server can't give yet. It's like a restaurant adding a dish to the menu: the kitchen has to be ready to cook it before the waiters start offering it.

**Cross-question:** Two Angular versions in the wild?

**Cross-answer:**

Then the API has to keep supporting the old contract until clients upgrade. I don't break mobile users on a Friday.

Some users keep an old tab open for days, and mobile users may not update their app for weeks. So the old shape has to keep working until almost nobody uses it. And I avoid risky releases right before the weekend, when fewer people are around to fix things.

---

## Q10. State: Angular signal vs .NET session vs database?

**Answer:**

The database is the source of truth. Angular signals hold UI state. The server session or cookie answers “who is logged in”. If I have more than one server, I don't keep something like the shopping cart only in an in-memory singleton on the API.

"Source of truth" means the one place whose answer wins when things disagree. A signal in Angular is a value that the screen watches and redraws when it changes, like which tab is open or what's in a search box. That's short-lived and fine to lose on refresh.

An in-memory singleton is one object living in one server's memory. With two servers behind a load balancer, the user's next request may land on the other server, which has never seen their cart. It's like keeping notes on a sticky pad at one desk when the customer may be served at another desk next time.

**Cross-question:** In-memory cache as source of truth?

**Cross-answer:**

It disappears on restart and it's different on each server. It's fine as a cache, but not as the real record.

A cache is like keeping your most-used files on your desk instead of in the cupboard: faster to reach, but the cupboard (the database) still holds the real copy. If the desk is cleared overnight, nothing is lost.

---

## Q11. How do you test this stack without a full browser?

**Answer:**

For the API, I use WebApplicationFactory with a test database. For Angular, TestBed and HttpTestingController. Then a few e2e tests for login and one main flow. I don't write e2e tests for every field.

WebApplicationFactory starts the real API in memory inside the test, so I can send real HTTP requests to it without deploying anything. TestBed builds an Angular component in a test, and HttpTestingController catches its HTTP calls so I can check what it asked for and give it a fake reply.

E2E (end-to-end) tests drive a real browser through the whole app. They catch the most, but they're slow and break easily. So I keep many fast small tests and only a few e2e tests for the paths that matter most, like login. It's like checking each part of a car on the bench and then doing one short test drive, not driving across the country for every bolt.

**Cross-question:** Mock the API in e2e only?

**Cross-answer:**

Then I never catch real contract bugs. I want at least one test running against a real local API.

A mock is a fake API that returns whatever I tell it to. If the real API renames a field, the mock still sends the old name, so every test passes while production is broken. One test against the real API catches that kind of mismatch.

---

## Q12. Performance: slow page. Frontend or backend?

**Answer:**

I start in the Network tab. If TTFB (time to first byte) is high, it's the server or SQL. If the download is huge, it's the payload or the bundle. If the Performance panel shows heavy CPU, it's the Angular list. I measure before I guess.

TTFB is how long we wait before the server sends back even the first bit of the answer. A long wait there means the server is busy thinking, often in the database. The payload is the data in the response, and the bundle is the JavaScript the browser downloads to run the app. If those are big, the time goes into downloading. If the download is fast but the page still freezes, the browser is busy drawing too much on screen.

Measuring first stops me from spending a day speeding up the wrong part. It's like a doctor running a test before choosing the medicine.

**Cross-question:** First fix?

**Cross-answer:**

The biggest number. Often that's N+1 SQL or a list without trackBy. It's not a new framework.

N+1 SQL means the code runs one query for a list and then one more query for every row, so 100 rows become 101 trips to the database instead of one or two. `trackBy` tells Angular how to recognise each row, so when the list refreshes it only redraws rows that changed instead of rebuilding all of them. Both are small fixes with a big effect.

---

## Q13. How do you pass the logged-in user from Angular to a .NET background job?

**Answer:**

I don't pass a token into Hangfire and treat it as the only auth forever. I pass the user id and tenant id that the API already trusted, store the job under that tenant, and the worker opens that tenant's database. The job never trusts a raw id that came from the browser.

Hangfire is a .NET library that runs work in the background, like sending a big report later instead of making the user wait. The job may run minutes or hours later, when the user's token has already expired. So the token isn't a good thing to rely on there.

Instead, at the moment the request comes in, the API has already checked who the user is from the token. It saves those checked ids with the job. It's like a receptionist who checks your ID once, then writes your name on the work order. The worker trusts the work order, not whatever someone shouts across the room.

**Cross-question:** Run the job as the user?

**Cross-answer:**

I record who asked for it. The worker runs with a service identity plus those ids, and applies the same permission rules.

A service identity is the background worker's own account, not a person's login. Recording who asked keeps the audit trail clear. And because the worker applies the same permission rules, a user can't get more access through a background job than they'd have on the screen.

---

## Q14. CORS in local Angular (`localhost:4200`) + API (`localhost:5000`)?

**Answer:**

Those are different origins, so the API has to allow `localhost:4200` in dev. A proxy in `proxy.conf.json` can hide this during development by making calls look same-origin. In production it's often the same domain behind nginx.

An origin is the scheme, host and port together. Even on the same machine, a different port counts as a different site. By default the browser won't let a page read answers from another origin, unless that API says it's allowed (that's CORS).

The proxy works like a receptionist. Angular calls its own address, `localhost:4200/api/...`, and the dev server quietly forwards the call to `localhost:5000`. The browser only sees one origin, so CORS never comes up. In production, nginx (a web server) often does the same job: the app and the API sit under one domain.

**Cross-question:** `AllowAnyOrigin` to save time?

**Cross-answer:**

Not with credentials. I list the real origins.

`AllowAnyOrigin` lets every website on the internet call the API from a user's browser. ASP.NET Core doesn't even allow it together with credentials (cookies or auth headers), because then any site could act as the logged-in user. Listing the exact origins we trust is safer and only takes a minute.

---

## Q15. What do you put in git vs user secrets vs environment?

**Answer:**

Git gets the code, non-secret config keys, and example values. User secrets hold local connection strings. Production secrets go in server environment variables or a vault. The Angular environment file holds only the public API URL.

User secrets is a .NET feature that keeps private settings in a file on the developer's own machine, outside the project folder, so they never get committed by accident. A vault (like Azure Key Vault) is a secure store for passwords and keys that only the app and a few people can read.

Why it matters: anything that goes into git stays in its history, even after you delete it, and everyone with access to the repo can see it. The Angular environment file ends up inside the JavaScript sent to every browser, so anything in it is public. It's like house keys: you don't tape them to the front door, even if the door is yours.

**Cross-question:** `appsettings.json` with a real password?

**Cross-answer:**

No.

`appsettings.json` is committed to git, so a real password there would be visible to everyone with repo access and stay in the history. I keep a placeholder or empty value there, and the real one comes from user secrets locally or from environment variables or a vault in production.

---

## Q16. How would you design a dashboard number like “open incidents”?

**Answer:**

First I agree with the business on what the number means. Then I count it in SQL/EF with the tenant filter and return one DTO. Angular shows the number with a loading and an error state. I don't count in the UI from a partial list.

"Open" sounds simple, but it can mean different things: does "on hold" count? Do closed-then-reopened ones count? If we don't agree first, the number will look wrong to someone, even if the code is perfect.

Counting in the database (through EF, Entity Framework, the .NET tool that turns C# into SQL) gives the true total. If Angular counted the rows it had, it might only see the first page of 20 and show "20" when the real answer is 340. The loading and error states matter too: showing "0" while data is still loading or after a failure would mislead people.

**Cross-question:** Cache the number?

**Cross-answer:**

Maybe for a few seconds, keyed by tenant. Stale cards confuse people, so I keep the TTL short.

Caching means remembering the answer for a short time so we don't count again on every page load. "Keyed by tenant" means each customer has their own saved number, so one company never sees another's count. TTL (time to live) is how long the saved number is kept before we count again.

---

## Q17. XSS from a field the user typed, shown in Angular, stored in .NET?

**Answer:**

I store the raw text (or sanitized text if the product needs HTML), and I encode it on output. Angular interpolation already encodes. I never use `innerHTML` for that field. The API doesn't return HTML unless it's a very clear, sanitized feature.

XSS (cross-site scripting) is when someone types code, like a `<script>` tag, into a field, and the app later shows it to other users as real code. The browser runs it, and it can steal their session or act as them.

Encoding means turning special characters into safe text, so `<script>` is shown as those letters on screen instead of being run. Angular's `{{ }}` interpolation does this for us. `innerHTML` skips that safety, so the browser treats the text as real HTML. Sanitizing means removing the dangerous parts and keeping only safe tags.

**Cross-question:** Markdown?

**Cross-answer:**

I render it with a sanitizer. No raw HTML from the user.

Markdown turns into HTML, and most Markdown tools also let people mix in raw HTML. So the output goes through a sanitizer that keeps only safe tags like bold, lists and links, and strips scripts and event handlers.

---

## Q18. How do you handle timezones?

**Answer:**

I store UTC. The API sends ISO-8601 with a Z or an offset. Angular shows it in the user's locale with `DatePipe`. I don't store “server local time”.

UTC is one world clock that doesn't change with the seasons or the country. ISO-8601 is the standard way to write a date and time, like `2024-03-10T09:30:00Z`. The Z means "this is UTC", and an offset like `+05:30` says how far from UTC it is. With that, every client knows the exact moment.

Storing the server's local time causes trouble when the server moves to another region or the clocks change for daylight saving. It's like writing "meet at 9" in a note to friends in different countries without saying whose 9. With UTC we store the exact moment once, and each user sees it in their own local time.

**Cross-question:** Birthday?

**Cross-answer:**

That's a date with no time, so I use `DateOnly` on the server. Don't convert it through UTC, or the day can shift.

A birthday is the same calendar day everywhere. If we stored 15 March as midnight in India and then converted it to UTC, it would become 14 March at 18:30, and a user in another timezone could see the wrong day. `DateOnly` holds just the date, with no time and no timezone, so nothing can shift it.

---

## Q19. Feature flag: who owns it?

**Answer:**

The flag lives in config or a small table. The API can hide endpoints and Angular can hide routes, and I do both. A hidden button isn't enough if the API still works.

A feature flag is an on/off switch for a feature that we can flip without a new release. It's handy for testing something with a few customers first, or turning off a broken feature fast.

Angular hiding the route keeps the screen clean for users. The API refusing the call is what actually keeps the feature off. If only the button is hidden, someone who knows the URL can still use it, like taking the sign off a door but leaving it unlocked.

**Cross-question:** Flag in Angular environment only?

**Cross-answer:**

Users can flip it in DevTools. The API has to refuse the call.

Everything in the Angular app runs in the user's browser, so they can change values there or call the API directly. The flag check that counts is the one on the server.

---

## Q20. You have one day to ship MVP. What do you cut?

**Answer:**

I keep auth, the tenant check, one list, one create, and the empty and error states. I cut pretty charts, extra filters, perfect animations, and extra roles. And I write the risks down.

An MVP (minimum viable product) is the smallest version that does the main job and can be shown to real users. What I keep is what makes it safe and usable: people log in, they only see their own company's data, they can see and add records, and they get a clear message when there's nothing to show or something fails.

What I cut is nice to have and easy to add later. A missing chart annoys people, but a data leak between customers can't be undone. Writing the risks down means everyone knows what was skipped, so nothing is forgotten after launch.

**Cross-question:** Skip tests?

**Cross-answer:**

If I can, I keep one isolation test and the main happy path. I don't skip auth checks.

An isolation test checks that one tenant can't see another tenant's data. The happy path is the normal flow where everything goes right, like logging in and creating a record. Those two tests cover the worst risk and the main use, so they're worth the time even on a tight day.

---

## Q21. Explain your stack to a hiring manager who is not a coder.

**Answer:**

C# and .NET are the kitchen. Angular is the dining room. JavaScript is the language the dining room speaks. The database is the pantry. My job is to make sure an order from table 5 never gets table 6's food.

So guests (users) only see the dining room. They order there, the kitchen does the real work behind the door, and the pantry stores everything. The "table 5 vs table 6" part is about keeping each customer's data separate and correct, which is the thing that matters most to a business.

**Cross-question:** What is a DTO in that picture?

**Cross-answer:**

It's the plate we send out, not the whole pantry shelf.

The kitchen picks only what this order needs and puts it on a plate. In the same way, a DTO carries only the fields the screen needs, not the whole database record with things like internal ids or password hashes.

---

## Q22. What is the biggest bug you have seen between UI and API?

**Answer:**

Pick a real one if you have it. Here's a safe example:

> The UI sent dates as local strings, but the API parsed them as UTC. So the filters were off by one day. We agreed on ISO UTC in the API and used a `DatePipe` on the screen.

Why this is a good example: it's a bug that lives in the gap between two sides that each look correct on their own. A user in India picks "10 March" as a start date, the browser sends it with no timezone, and the server reads it as UTC midnight. Converted back, the time is now on a different day, so records near midnight fall into the wrong day. Interviewers like it because it shows you understand the contract between UI and API, not just one side.

**Cross-question:** How do you prevent it?

**Cross-answer:**

One date format in the contract, a shared example in the API docs, and a test that checks a known instant round-trips correctly.

"Round-trips" means we send a known date and time to the API, read it back, and check it's still exactly the same moment. If anyone changes how dates are handled on either side, that test fails before users notice.

---

## Q23. How do you secure a multi-tenant Angular + .NET app? Five bullets.

**Answer:**

1. HTTPS.
2. Auth on every API.
3. Tenant comes from the server context, not from the body.
4. DTOs, with no extra fields.
5. Angular hides UI, but the API still checks.

Multi-tenant means many customer companies share one app, so the main danger is one company seeing another's data. In plain words: encrypt all traffic so nobody can read it on the way, check the login on every call (not just some), and work out which company the user belongs to from their verified token, never from a value they send, which they could change.

Returning DTOs means we send only the fields the screen needs, so nothing private leaks by accident. And the last point sums it up: hiding a button in Angular helps the user, but only the server's check actually protects the data.

**Cross-question:** Sixth?

**Cross-answer:**

Don't log tokens, and don't put secrets in the Angular bundle.

Logs are read by many people and tools, and a token in a log can be copied and used to act as that user. The Angular bundle is downloaded to every browser, so anything inside it, like an API key, is public.

---

## Q24. RxJS `switchMap` vs a .NET cancel token — same idea?

**Answer:**

Yes, in spirit. `switchMap` drops the old HTTP call. `CancellationToken` tells EF or HTTP work to stop. If the user types fast, I want both: Angular cancels the old request, Kestrel notices the abort, and EF stops.

Picture a search box. The user types "a", "ap", "app". Each letter could start a request. `switchMap` (an RxJS operator) says "a newer search came in, so forget the old one", and it cancels the old HTTP call. That way an old, slow answer can't arrive last and overwrite the newest results.

On the server, Kestrel (the web server inside ASP.NET Core) sees that the browser gave up on the request. A `CancellationToken` is the signal that carries that news into our code, so EF can stop the database query instead of finishing work nobody will read. It's like a customer cancelling an order: the waiter tells the kitchen so they stop cooking it.

**Cross-question:** Does Angular cancel reach EF automatically?

**Cross-answer:**

If the browser aborts, the request can be cancelled. But I still have to pass the token through the API action. I don't ignore it.

ASP.NET Core can give the action a `CancellationToken` that fires when the client disconnects. But EF only stops if I pass that token on, for example into `ToListAsync(cancellationToken)`. If I don't pass it, the query keeps running to the end even though nobody is waiting.

---

## Q25. Why might `ng serve` work and production `ng build` fail at runtime?

**Answer:**

Common reasons: a wrong API URL in environment.prod, hash routing vs server rewrite rules, the base href, the minifier exposing a bug, or a lazy chunk 404 because the path is wrong.

`ng serve` is the development server, and it's forgiving. A production build is packed differently and served by a real web server, so small setup differences show up only there. In plain words:

- **API URL:** the production settings file points at the wrong server.
- **Routing and rewrites:** if the user refreshes on `/orders/5`, the web server looks for a file with that path, finds none, and returns 404. It has to be told to send `index.html` instead (or the app uses hash routing like `/#/orders/5`).
- **Base href:** tells the app which folder it lives in. If it's wrong, the browser looks for the script files in the wrong place.
- **Minifier:** the build shrinks the code, which can expose code that only worked by luck.
- **Lazy chunk 404:** parts of the app load on demand, and if their path is wrong, that page fails to open.

**Cross-question:** First place you look?

**Cross-answer:**

The browser console and the failed network request. Then `environment.prod` and the server's fallback to `index.html`.

The console shows the actual error, and the failed request shows exactly which URL broke and with what status. That usually points straight at one of the causes above, so I don't have to guess.

---

## Q26. Walk me through NriCare login from the Angular button.

**Answer:**

The user types their phone number and password. Angular posts to `POST /api/Auth/login`. The API finds the user, checks the password with BCrypt, writes a `UserLoginActivity` row, and returns a JWT and a refresh token, plus flags like `isApprovedUser` and `hasActiveSubscription`. Angular stores the access token and sends `Authorization: Bearer` on later calls. The session claims drive the association filter. OTP is part of a separate signup flow, not this login.

A few terms in plain words. BCrypt is a slow, one-way way to scramble passwords. It's slow on purpose, so guessing millions of passwords takes too long. A JWT (JSON Web Token) is a signed note from the server that says who the user is. It works like a stamped visitor pass that can't be changed without the stamp breaking. The refresh token is used to get a new JWT when the old one expires, so the user doesn't have to log in again.

`Authorization: Bearer` is the header that carries the JWT on each call, meaning "whoever bears this pass". Claims are the facts written inside the token, like the user's id. The API reads them to decide which association's data this user may see.

**Cross-question:** Password in the log?

**Cross-answer:**

On login we log the phone number, not the password. We never log the JWT or the refresh token.

The phone number helps us trace who tried to log in. The password, JWT and refresh token are all keys. Anyone who reads them in a log could use them to get into the account.

---

## Q27. NR user books a service from the app. What hits the API?

**Answer:**

The web or app sends a booking POST with the service provider, service type, and dates. The server checks that the user is an NR user, gets the price from `ServiceProviderServices`, and adds tax, association, platform, and verifier fees. Then it locks the wallet, creates a hold, opens a chat, and returns the booking number. Angular then shows the booking as Under Discussion along with the chat thread.

The key idea is that the server works out the price, not the app. The app only says what is being booked. If the app sent the price, a user could change it to 1 rupee. Reading it from `ServiceProviderServices` keeps the price honest.

Locking the wallet stops two bookings from spending the same balance at the same moment. A hold is money set aside but not paid yet, like when a hotel blocks an amount on your card at check-in and only charges it at check-out.

**Cross-question:** Is the provider paid at that moment?

**Cross-answer:**

No, it's only a hold. The payout happens in `ReleasePayment` when the job is completed (or after verifier approval).

This protects the user: the money is kept safe until the work is done, and only then does it move to the provider.

---

## Q28. How do web and mobile share the same backend?

**Answer:**

It's one ASP.NET API. Web routes are under `api/...`, mobile under `api/app/...`, and common ones like `api/Auth`, OTP, and files are shared. There are two Swagger docs. Both use the same JWT and the same database. The Angular admin/web app is `NriCare.Web.UI`. That's why a booking created on mobile still shows up in the web association dashboard, as long as the association filter allows it.

So there's one kitchen serving two dining rooms. The routes are split so each client gets endpoints shaped for its own screens, while shared things like login aren't written twice. Swagger is the auto-made API documentation page, and having two keeps the web and mobile lists easy to read.

Because it's one database, there's only one copy of every booking. Nothing has to be synced between web and mobile, so they can't drift apart.

**Cross-question:** Different tokens per client?

**Cross-answer:**

The token shape is the same. `session_id` is per login, not per “web vs app”. Logout should revoke that session's refresh token, but today logout only records the logout time.

Each time someone logs in, on any device, they get their own `session_id`. Revoking the refresh token on logout would mean that session can't get new access tokens anymore. Since today logout only records the time, a refresh token from that session would still work until it expires. That's a known gap to be honest about.


---
## Additional Questions
### Q29. How do you handle a file upload end to end?
**Answer:** Angular sends FormData. The API checks authentication, authorization, size and type, stores the file in private or object storage, and returns a safe file id or URL. Downloads go through permission checks.

Authentication is "who are you?" and authorization is "are you allowed to do this?". Both are checked before the file is accepted. Object storage (like Azure Blob or Amazon S3) is a service built for keeping lots of files, separate from the web server, so the files aren't open to the public by default.

Returning a safe id instead of the real file path means users can't guess other people's files. When someone wants the file back, the API checks their permission first, like a library desk that checks your card before handing over a book from the back room.
### Q30. How do you keep frontend and backend contracts aligned?
**Answer:** I define DTOs and API examples clearly, use generated clients or a shared OpenAPI spec where it makes sense, and test the important contracts. A field rename on the backend should not quietly break the Angular app.

The "contract" is the agreed shape of the data: field names, types and what's required. A generated client is TypeScript code made automatically from the API's OpenAPI description, so Angular's types always match what the API really sends.

Why it matters: a mismatch usually doesn't crash anything. The screen just shows a blank where a value should be, and nobody notices until a user complains. Generated types turn that into a compile error, and contract tests catch it before release.
### Q31. How would you debug a slow full-stack page?
**Answer:** I start with the browser's Network waterfall. Then I separate API time, SQL time, payload size and rendering time. I fix the biggest measured bottleneck first.

The waterfall is the chart in the Network tab that shows each request as a bar: when it started, how long it waited, and how long it took to download. It quickly shows whether time goes into waiting for the server or into the browser itself.

Splitting the time into those four parts tells me where to look: slow API code, a slow database query, too much data sent, or too much work drawing the screen. A bottleneck is the narrowest point, like the neck of a bottle, that limits everything else. Fixing a small delay somewhere else won't help while the biggest one is still there.
