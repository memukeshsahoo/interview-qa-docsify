# .NET — Real-world Backend Interview

Practical questions for a backend developer. Keep the answer simple, then explain the trade-off.

---

## Q1. An API is slow. How do you debug it?

**Answer:**

First I reproduce it and measure where the time goes: application code, database, external API, or response serialization. Then I check logs, traces, SQL count, query time, and payload size. I fix the actual bottleneck instead of adding cache blindly.

In plain words, I find out which step is slow before I touch anything. A request is a chain of steps: my code runs, it asks the database for data, it may call another company's API, and then it turns the result into JSON (that last step is serialization). A trace shows how long each step took, a bit like a receipt that lists every item and its price. The SQL count tells me how many database queries one request made, and the payload size tells me how much data I'm sending back.

This matters because guessing wastes time and can make things worse. If the real problem is one query running 200 times, adding a cache only hides it for a while, and now I also have stale data to worry about. It's like a car that's slow because the handbrake is on: a bigger engine won't fix it, releasing the handbrake will.

**Cross-question:** What if the database query is slow?

**Cross-answer:**

I check the generated SQL, the execution plan, indexes, filters, joins, and how much data is loaded. I also look for N+1 queries.

The generated SQL is the actual query that EF (Entity Framework, the library that turns my C# code into SQL) sends to the database. The execution plan is the database's own explanation of how it will run that query, for example whether it reads the whole table or jumps straight to the right rows using an index. An index works like the index at the back of a book: it lets the database find rows without reading every page. N+1 means I run one query to get a list and then one more query for every item in that list, so 100 items turn into 101 trips to the database.

---

## Q2. An endpoint returns 10,000 rows. What would you change?

**Answer:**

I would not return everything by default. I would add pagination, filtering and sorting. I would select only the fields the UI needs, and use AsNoTracking for read-only queries.

Pagination means sending the data in pages, like 20 rows at a time, the way a search engine shows results page by page instead of all at once. Filtering and sorting let the user ask for just what they want, such as "only open orders, newest first". Selecting only the needed fields means if the screen shows a name and a date, I don't also send the address, notes and history. AsNoTracking tells EF that I'm only reading the data, so it doesn't need to keep a copy of every row to watch for changes. That saves memory and time.

Without these changes, every call pulls 10,000 rows from the database, holds them in server memory, sends a big response over the network, and makes the user's browser or phone process all of it. Nobody reads 10,000 rows on one screen anyway.

**Cross-question:** Why not just increase the server timeout?

**Cross-answer:**

That just hides the problem. Large responses still use up database, memory and network resources.

A timeout is how long the server waits before giving up. Making it longer means the request fails less often, but it's still slow and still expensive. When many users hit that endpoint at the same time, the load adds up and can slow down the whole system for everyone, not just this one screen.

---

## Q3. Two users update the same record at the same time. How do you handle it?

**Answer:**

I use optimistic concurrency. That means I don't lock the row; I just check that nobody changed it before I save. For example, I use a PostgreSQL xmin-based concurrency token or another version column. If the version changed before my update, EF throws a concurrency exception. I return a clear conflict response and ask the client to reload.

Here's the problem it solves. Two people open the same record. Person A changes the phone number and saves. Person B, still looking at the old version, changes the address and saves. Without a check, B's save can quietly overwrite A's change, and nobody notices. This is called a "lost update".

A version column is like a version number on a shared document. When I load the record, I remember its version. When I save, the database only accepts my change if the version is still the same. In PostgreSQL, xmin is a hidden system column that changes every time a row is updated, so it can act as that version number without me adding a new column. If someone else saved first, the version no longer matches, EF throws an exception, and I tell the user "this record was changed by someone else, please reload". It's called "optimistic" because I assume conflicts are rare and only check at save time, instead of locking the record for everyone while one person edits it.

**Cross-question:** Is last-write-wins always okay?

**Cross-answer:**

No. It can be acceptable for some non-critical fields, but it is risky for money, inventory and important status transitions.

Last-write-wins means whoever saves last simply overwrites the earlier change. For something like a user's display preference, that's usually fine. But if two people both sell the last item in stock, or both change a balance, or one person cancels an order while another marks it as shipped, silently losing one of those changes leads to wrong money, wrong stock counts or an order in an impossible state.

---

## Q4. A payment request times out. The client retries. How do you prevent double payment?

**Answer:**

I make the operation idempotent. The client sends an idempotency key, and the database has a unique constraint for that operation. A retry with the same key returns the existing result instead of creating another payment.

Idempotent means doing the same thing twice has the same effect as doing it once. A lift button is a good example: pressing it five times still calls the lift once. The idempotency key is a unique ID the client creates for one payment attempt and sends with every retry of that same attempt. A unique constraint is a rule in the database that says "this value can only appear once in this table".

Why it matters: a timeout doesn't tell the client whether the payment went through. Maybe the server charged the card and only the reply got lost. If the client retries without a key, the customer can be charged twice. With the key, the server sees "I've already handled this one" and just returns the first result.

**Cross-question:** Is checking AnyAsync alone enough?

**Cross-answer:**

No. Two requests can both pass the check. The database unique constraint is the final protection.

AnyAsync asks the database "does a payment with this key already exist?". The problem is timing. If two identical requests arrive at almost the same moment, both ask the question, both hear "no", and both go ahead and insert a payment. This is called a race condition. The unique constraint is checked by the database at the moment of inserting, so the second insert fails no matter how close together the requests were. The AnyAsync check is still useful for giving a friendly response in the normal case, but the constraint is what actually guarantees it.

---

## Q5. Your API updates the database and then sends an email. The email fails. What should happen?

**Answer:**

Normally I don't make the HTTP request depend on the email going out. Instead, I save an outbox record in the same transaction as the business change. A background worker sends the email and retries failures.

A transaction is a group of database changes that either all succeed or all fail together. The outbox is a table where I write "an email needs to be sent" as a row. Because that row is saved in the same transaction as the real change (say, a booking), I can't end up with a booking and no email request, or an email request with no booking. A background worker is a separate piece of code that keeps checking the outbox, sends the emails, and tries again if the email service is down.

It works like a shop's "letters to post" tray. The clerk finishes the sale and drops the letter in the tray. If the post office is closed, the letter waits in the tray and goes out later. The customer doesn't have to stand at the counter waiting, and the sale isn't cancelled just because the post office was closed.

**Cross-question:** Why not send the email before SaveChanges?

**Cross-answer:**

Then the email can be sent even if the database transaction later rolls back.

SaveChanges is the point where EF actually writes to the database. If I send "Your booking is confirmed" first and then the save fails, for example because of a validation error or a conflict, the database undoes the change (that's a rollback). Now the customer has a confirmation email for a booking that doesn't exist, and I can't take the email back.

---

## Q6. How do you prevent a user from accessing another user's resource by changing the ID?

**Answer:**

I never rely on the ID being hard to guess. I load the resource and check ownership, tenant and permission on the server.

Imagine a URL like /orders/105. A curious user changes it to /orders/106 and sees someone else's order. This is a very common security hole. The fix is that on every request, the server checks: does this record belong to the logged-in user? Is it in the same tenant (in a system shared by many companies or hospitals, a tenant is one of those customers, and their data must stay separate)? Does this user's role allow this action? If any answer is no, I refuse the request.

The check has to happen on the server because anything in the browser can be changed by the user. Hiding a button in the UI doesn't stop someone from calling the API directly.

**Cross-question:** Are GUIDs enough?

**Cross-answer:**

No. A random ID is not an authorization check.

A GUID is a long random ID that's very hard to guess. But IDs leak all the time: in shared links, screenshots, browser history, logs or emails. Once someone has the ID, a random-looking value doesn't stop them. It's like having a hard-to-guess house number but no lock on the door. The server still has to check that this person is allowed to see this record.

---

## Q7. Your background job uses a DbContext from an HTTP request. What is wrong?

**Answer:**

The request scope may already be disposed when the job runs. A background job should create its own DI scope and resolve a fresh DbContext from that scope.

A DbContext is EF's object for talking to the database. In ASP.NET Core it's usually given to me by DI (dependency injection, the built-in system that creates objects and hands them to the code that needs them). Objects in a "scope" live only as long as that scope. For a web request, the scope ends when the response is sent, and the DbContext is disposed, meaning it's closed and cleaned up.

So if I start a background job and hand it the request's DbContext, the request may finish first. When the job then tries to use the DbContext, it's already gone and I get an error, or worse, odd behaviour that's hard to track down. It's like giving someone your visitor badge to use later: once your visit is over, the badge stops working. The background job should make its own scope (using IServiceScopeFactory), get its own DbContext from it, and dispose that scope when it's done.

**Cross-question:** Can I make DbContext singleton?

**Cross-answer:**

No. DbContext is normally scoped because it represents a unit of work and is not designed for concurrent use.

A singleton is one shared object for the whole application. A unit of work means one DbContext tracks the changes for one piece of work, such as one request, and saves them together. If every request shared a single DbContext, many requests would use it at the same time, which it doesn't support, and you'd get errors. It would also keep tracking more and more entities over time, using memory and mixing one user's changes with another's.

---

## Q8. How do you handle cancellation in a REST API?

**Answer:**

I accept a CancellationToken and pass it down to EF and to external async calls. That way, if the client disconnects, the server can stop work nobody is waiting for.

A CancellationToken is a signal that says "stop, nobody needs this anymore". In ASP.NET Core, the framework gives me one that fires when the client goes away, for example when the user closes the page or navigates elsewhere. If I pass it on to methods like ToListAsync(ct) or an HttpClient call, those methods can stop early instead of finishing a slow query whose result will just be thrown away.

It's like a restaurant where a customer leaves before the food is ready. If the waiter tells the kitchen, the cook can stop and work on the next order. Without cancellation, the server keeps doing expensive work for people who have already left, and that slows things down for the people still waiting.

**Cross-question:** Should I catch OperationCanceledException and return 500?

**Cross-answer:**

No. Cancellation is normal, expected control flow. I don't treat it like an application failure.

OperationCanceledException is the exception .NET throws when work stops because of a cancellation token. A 500 status means "the server has a bug or crashed". If I turn every cancelled request into a 500, my error logs and alerts fill up with false alarms, and real errors get harder to spot. The client has already gone, so nobody will read the response anyway.

---

## Q9. How do you safely retry an external API call?

**Answer:**

I only retry operations that are safe to retry, or that are protected by idempotency. I use a limited number of retries, with backoff and timeouts. I don't retry every exception forever.

Some calls are safe to repeat, like reading data: asking twice does no harm. Others aren't, like "create an order", where a repeat could create two. Those I retry only if they have an idempotency key, so the other side knows it's the same request. Backoff means waiting a bit longer between each try, for example 1 second, then 2, then 4. A timeout means I don't wait forever for one try. I also only retry errors that are likely to be temporary, like a network blip or a "service busy" response, not errors like "invalid data" that will fail the same way every time.

Why it matters: if a service is struggling and every client retries instantly and endlessly, they pile on more load and keep it down for longer. It's like redialling a busy phone line every second; spacing the calls out gives the other side room to recover.

**Cross-question:** Should a payment capture always be retried automatically?

**Cross-answer:**

Only when the provider's contract says the operation is safe to retry, ideally with an idempotency mechanism.

Capturing a payment means actually taking money that was earlier approved. If the first try timed out, I don't know whether the money was taken. A blind retry could charge the customer twice. So I follow what the payment provider's documentation says. Many providers accept an idempotency key so a repeat is recognised as the same request. If that isn't available, I check the payment status with the provider before trying again.

---

## Q10. A database query works in development but is slow in production. Why?

**Answer:**

Production has different data volume, indexes, statistics, hardware and concurrency. I compare the generated SQL and execution plan in production and check actual row counts.

On my laptop the table might have 500 rows. In production it might have millions, and a query that reads the whole table goes from instant to painfully slow. Indexes may differ between environments. Statistics are the database's notes about how the data is spread out, and the database uses them to choose how to run a query, so different statistics can lead to a different (and worse) plan. Production also has many users running queries at the same time (that's concurrency), competing for the same resources.

So I don't guess from what I see locally. I look at the real execution plan in production and compare how many rows the database expected with how many it actually read. A big gap there often points to the problem.

**Cross-question:** Would adding an index always fix it?

**Cross-answer:**

No. An index helps specific access patterns, and it also adds write and storage cost.

An index on a column only helps queries that filter or sort by that column in a way the index can use. It won't help if the slow part is something else, like loading too many rows or a bad join. And every index has a cost: each insert, update or delete also has to update the index, and it takes up disk space. Adding indexes everywhere can make writes slower without fixing the original read problem.

---

## Q11. How would you design a common notification system for multiple modules?

**Answer:**

I would keep one shared notification entity with UserId, Module, Type, Title, Message, ReferenceId, IsRead and timestamps. The module and reference tell the app where to go when the user opens the notification. Module-specific business data stays outside the shared notification table.

In everyday terms, every part of the app (bookings, wallet, chat and so on) drops its notifications into the same inbox. Each notification says who it's for, which module it came from, what kind it is, the text to show, whether it's been read, and when it was created. ReferenceId is the ID of the thing it's about, like a booking ID. So when the user taps it, the app knows from Module and ReferenceId to open that exact booking.

The details of the booking itself stay in the bookings table, not in the notification. That keeps the notification table simple and the same shape for every module, and it means I don't copy business data that could go out of date.

**Cross-question:** Should every module have its own notification table?

**Cross-answer:**

Not if the lifecycle and delivery model are the same. One shared entity is simpler for unread counts, listing, read state and push notification handling.

If all notifications are created, shown, marked as read and pushed to the phone the same way, separate tables just mean doing the same work several times. To show the bell icon's unread count, I'd have to query every table and add the numbers up. With one table, it's one query, one "mark as read" endpoint, and one place that sends push notifications.

---

## Q12. How would you design wallet transactions?

**Answer:**

I would treat every money change as a ledger entry, not just an edit to a balance. The balance update and transaction records should be atomic. For holds and releases, I would use an idempotency key and database constraints to prevent duplicates.

A ledger is like a bank statement: a list of every credit and debit, each with an amount, a reason and a time. Instead of just changing the balance from 500 to 300, I also record "200 debited for booking X". Atomic means the balance change and the ledger entry are saved together in one transaction, so I never end up with one saved and the other missing.

A hold is when money is set aside for something but not taken yet, and a release is giving it back. These often happen through retries or background jobs, so the same hold could be attempted twice. An idempotency key plus a unique constraint in the database makes sure the second attempt is rejected instead of holding the money twice.

**Cross-question:** Why not only store the current balance?

**Cross-answer:**

A balance alone does not give a reliable audit trail. A ledger lets us understand where the money came from and where it went.

If a user says "my balance is wrong", a single number can't tell me what happened. With a ledger I can go through every entry and see each top-up, payment, hold and refund. I can also add up the entries and check that they match the stored balance, which helps catch bugs. For anything involving money, being able to explain every change is essential.

---

## Q13. How do you prevent two requests from spending the same wallet balance?

**Answer:**

The database has to enforce the concurrency rule. Depending on the operation, I can use PostgreSQL FOR UPDATE, an atomic conditional update, or optimistic concurrency. Application-level locks alone are not enough when multiple API servers are running.

The risk: a wallet has 100. Two payments of 80 arrive at the same moment. Both read the balance, both see 100, both decide "that's enough", and both go through. Now the wallet is at minus 60.

There are a few ways to stop this in the database:
- FOR UPDATE locks the wallet row when I read it, so the second request has to wait until the first one finishes, and then it sees the new balance.
- An atomic conditional update does the check and the change in one statement, like "reduce the balance by 80 only if the balance is at least 80". The database runs it as one step, so two requests can't both pass.
- Optimistic concurrency (from Q3) lets both try, but only the first save succeeds; the second sees the version changed and has to retry with fresh data.

I pick one based on the operation. The point is that the database is the one place every server shares, so that's where the rule has to live.

**Cross-question:** Why is SemaphoreSlim not enough?

**Cross-answer:**

It only coordinates threads inside one application process. It cannot coordinate two separate servers.

SemaphoreSlim is a .NET lock that lets only a set number of threads into a piece of code at a time. But it lives in the memory of one running app. If my API runs on two servers behind a load balancer, each server has its own separate semaphore and they know nothing about each other. Request 1 on server A and request 2 on server B both get in. It's like two shops each having their own "one customer at a time" rule; it doesn't stop one customer from being in each shop at once.

---

## Q14. What should an API return when a business rule fails?

**Answer:**

I return a meaningful HTTP status and an error response with a stable shape. For example, 409 Conflict fits a state conflict, like trying to accept a quotation that is already accepted. I avoid returning raw exception messages or database errors.

An HTTP status is the number that says how the request went: 200 means OK, 400 means bad request, 409 means conflict, and so on. A stable shape means every error response has the same fields, like a code and a message, so the frontend can handle all errors the same way and show a clear message to the user.

I don't send raw exception or database messages for two reasons. They mean nothing to the user ("violates foreign key constraint..."), and they can reveal details about my database and code that an attacker could use.

**Cross-question:** 400 or 409?

**Cross-answer:**

I use 400 when the request itself is invalid. I use 409 when the request is fine but conflicts with the current state of the resource.

400 Bad Request is for when the request itself is wrong, like a missing required field or a negative quantity. The client has to fix what it sent. 409 Conflict is for when the request is well formed but can't be applied right now because of the record's current state, like accepting a quotation that's already accepted. The same request might have worked a minute earlier; the data changed, not the request.

---

## Q15. How do you review an EF Core query before putting it into production?

**Answer:**

I check a few things. Do filtering and projection happen in SQL? Does the query load navigation properties it doesn't need? Is pagination applied before ToListAsync? Do indexes support the filters and ordering? For important queries, I also look at the generated SQL and the execution plan.

Projection means picking only the columns I need (with Select) instead of whole objects. Navigation properties are the links from one entity to related ones, like an order's customer or its items. Loading them when I don't need them pulls extra tables and extra rows. ToListAsync is the point where EF actually runs the query and brings the results into memory, so anything that comes before it runs in the database, and anything after runs in my app.

The aim is simple: the database should do the narrowing down, and only the small result should travel to my app. Checking the generated SQL confirms EF is doing what I think it's doing, and the execution plan shows whether the database can use an index or has to scan the whole table.

**Cross-question:** What is the common mistake?

**Cross-answer:**

Calling ToListAsync too early and then doing filtering, sorting or pagination in memory.

For example, loading all orders with ToListAsync() and then calling Where and Take(20) on the list. The code looks fine and gives the right result, but the database sends every row to the app first, and the app throws most of them away. It's like asking a library to post you every book they have so you can pick the one you want, instead of asking for that one book. On a small test database you won't notice; in production with millions of rows, it's very slow.
