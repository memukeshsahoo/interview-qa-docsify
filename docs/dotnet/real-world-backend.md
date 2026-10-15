# .NET — Real-world Backend Interview

Practical questions for a backend developer. Keep the answer simple, then explain the trade-off.

---

## Q1. An API is slow. How do you debug it?

**Answer:**

First I reproduce it and measure where the time goes: application code, database, external API, or response serialization. Then I check logs, traces, SQL count, query time, and payload size. I fix the actual bottleneck instead of adding cache blindly.

**Cross-question:** What if the database query is slow?

**Cross-answer:**

I check the generated SQL, the execution plan, indexes, filters, joins, and how much data is loaded. I also look for N+1 queries.

---

## Q2. An endpoint returns 10,000 rows. What would you change?

**Answer:**

I would not return everything by default. I would add pagination, filtering and sorting. I would select only the fields the UI needs, and use AsNoTracking for read-only queries.

**Cross-question:** Why not just increase the server timeout?

**Cross-answer:**

That just hides the problem. Large responses still use up database, memory and network resources.

---

## Q3. Two users update the same record at the same time. How do you handle it?

**Answer:**

I use optimistic concurrency. That means I don't lock the row; I just check that nobody changed it before I save. For example, I use a PostgreSQL xmin-based concurrency token or another version column. If the version changed before my update, EF throws a concurrency exception. I return a clear conflict response and ask the client to reload.

**Cross-question:** Is last-write-wins always okay?

**Cross-answer:**

No. It can be acceptable for some non-critical fields, but it is risky for money, inventory and important status transitions.

---

## Q4. A payment request times out. The client retries. How do you prevent double payment?

**Answer:**

I make the operation idempotent. The client sends an idempotency key, and the database has a unique constraint for that operation. A retry with the same key returns the existing result instead of creating another payment.

**Cross-question:** Is checking AnyAsync alone enough?

**Cross-answer:**

No. Two requests can both pass the check. The database unique constraint is the final protection.

---

## Q5. Your API updates the database and then sends an email. The email fails. What should happen?

**Answer:**

Normally I don't make the HTTP request depend on the email going out. Instead, I save an outbox record in the same transaction as the business change. A background worker sends the email and retries failures.

**Cross-question:** Why not send the email before SaveChanges?

**Cross-answer:**

Then the email can be sent even if the database transaction later rolls back.

---

## Q6. How do you prevent a user from accessing another user's resource by changing the ID?

**Answer:**

I never rely on the ID being hard to guess. I load the resource and check ownership, tenant and permission on the server.

**Cross-question:** Are GUIDs enough?

**Cross-answer:**

No. A random ID is not an authorization check.

---

## Q7. Your background job uses a DbContext from an HTTP request. What is wrong?

**Answer:**

The request scope may already be disposed when the job runs. A background job should create its own DI scope and resolve a fresh DbContext from that scope.

**Cross-question:** Can I make DbContext singleton?

**Cross-answer:**

No. DbContext is normally scoped because it represents a unit of work and is not designed for concurrent use.

---

## Q8. How do you handle cancellation in a REST API?

**Answer:**

I accept a CancellationToken and pass it down to EF and to external async calls. That way, if the client disconnects, the server can stop work nobody is waiting for.

**Cross-question:** Should I catch OperationCanceledException and return 500?

**Cross-answer:**

No. Cancellation is normal, expected control flow. I don't treat it like an application failure.

---

## Q9. How do you safely retry an external API call?

**Answer:**

I only retry operations that are safe to retry, or that are protected by idempotency. I use a limited number of retries, with backoff and timeouts. I don't retry every exception forever.

**Cross-question:** Should a payment capture always be retried automatically?

**Cross-answer:**

Only when the provider's contract says the operation is safe to retry, ideally with an idempotency mechanism.

---

## Q10. A database query works in development but is slow in production. Why?

**Answer:**

Production has different data volume, indexes, statistics, hardware and concurrency. I compare the generated SQL and execution plan in production and check actual row counts.

**Cross-question:** Would adding an index always fix it?

**Cross-answer:**

No. An index helps specific access patterns, and it also adds write and storage cost.

---

## Q11. How would you design a common notification system for multiple modules?

**Answer:**

I would keep one shared notification entity with UserId, Module, Type, Title, Message, ReferenceId, IsRead and timestamps. The module and reference tell the app where to go when the user opens the notification. Module-specific business data stays outside the shared notification table.

**Cross-question:** Should every module have its own notification table?

**Cross-answer:**

Not if the lifecycle and delivery model are the same. One shared entity is simpler for unread counts, listing, read state and push notification handling.

---

## Q12. How would you design wallet transactions?

**Answer:**

I would treat every money change as a ledger entry, not just an edit to a balance. The balance update and transaction records should be atomic. For holds and releases, I would use an idempotency key and database constraints to prevent duplicates.

**Cross-question:** Why not only store the current balance?

**Cross-answer:**

A balance alone does not give a reliable audit trail. A ledger lets us understand where the money came from and where it went.

---

## Q13. How do you prevent two requests from spending the same wallet balance?

**Answer:**

The database has to enforce the concurrency rule. Depending on the operation, I can use PostgreSQL FOR UPDATE, an atomic conditional update, or optimistic concurrency. Application-level locks alone are not enough when multiple API servers are running.

**Cross-question:** Why is SemaphoreSlim not enough?

**Cross-answer:**

It only coordinates threads inside one application process. It cannot coordinate two separate servers.

---

## Q14. What should an API return when a business rule fails?

**Answer:**

I return a meaningful HTTP status and an error response with a stable shape. For example, 409 Conflict fits a state conflict, like trying to accept a quotation that is already accepted. I avoid returning raw exception messages or database errors.

**Cross-question:** 400 or 409?

**Cross-answer:**

I use 400 when the request itself is invalid. I use 409 when the request is fine but conflicts with the current state of the resource.

---

## Q15. How do you review an EF Core query before putting it into production?

**Answer:**

I check a few things. Do filtering and projection happen in SQL? Does the query load navigation properties it doesn't need? Is pagination applied before ToListAsync? Do indexes support the filters and ordering? For important queries, I also look at the generated SQL and the execution plan.

**Cross-question:** What is the common mistake?

**Cross-answer:**

Calling ToListAsync too early and then doing filtering, sorting or pagination in memory.
