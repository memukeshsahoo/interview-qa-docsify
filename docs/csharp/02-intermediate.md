# C# — Intermediate

OOP, generics, LINQ, async. Talk like you use these at work.

---

## Q1. Encapsulation, inheritance, polymorphism, abstraction?

**Answer:**

**Encapsulation:** hide the inner data. Use private fields and public methods.

**Inheritance:** a child type reuses a parent.

**Polymorphism:** I call the same method, the real type decides what runs.

**Abstraction:** I depend on `IRepository`, not `PostgresRepository`.

**Cross-question:** Inheritance or composition?

**Cross-answer:**

Composition first — “has a”. Inheritance only when it is a real “is a”. Deep class trees get painful. Interfaces plus injected helpers scale better.

---

## Q2. Abstract class vs interface?

**Answer:**

An **interface** is a promise. A type can have many.

An **abstract class** can hold shared code and fields. A class can have only one parent class.

**Cross-question:** Fields on an interface? Constructor on an abstract class?

**Cross-answer:**

Interfaces do not have instance fields. Abstract classes can have constructors and real state.

---

## Q3. Why generics? Why `List<T>` not `ArrayList`?

**Answer:**

Generics keep types safe at compile time. `List<int>` stores ints. Old `ArrayList` stores `object` and boxes ints. I get fewer surprises and better speed.

**Cross-question:** What is a constraint?

**Cross-answer:**

`where T : class, IEntity, new()` — T must be a class, must implement `IEntity`, must have an empty constructor. Then I can call those members on T.

---

## Q4. `IEnumerable<T>` vs `IQueryable<T>`?

**Answer:**

`IEnumerable` runs in memory.

`IQueryable` is an expression tree. EF can turn it into SQL.

If I `ToList()` first, then filter, that filter is in memory. If I filter before `ToList()` on a query, it can become a SQL `WHERE`.

**Cross-question:** I put my own C# method inside `Where` on EF. What happens?

**Cross-answer:**

EF often cannot turn it into SQL and throws. Keep the filter simple, or filter in memory after you already narrowed the rows.

---

## Q5. Deferred execution?

**Answer:**

`Where` and `Select` do not run at once. They run when you loop, or `ToList`, or `Count`. If I change the source before that, results change. If I loop twice, I may hit the database twice.

**Cross-question:** Which LINQ methods run now?

**Cross-answer:**

`ToList`, `ToArray`, `First`, `Any`, `Count`, `Sum`. On EF, they send SQL now.

---

## Q6. What does async / await actually do?

**Answer:**

`await` means “pause this method until that work finishes, and free the thread for other work”. The compiler builds a state machine for you. I/O like HTTP or SQL does not need a thread sitting and waiting.

**Cross-question:** Why is `async void` bad?

**Cross-answer:**

You cannot await it. Errors can crash the app or vanish. Only event handlers use `async void`. Everything else: `async Task`.

---

## Q7. Deadlock with `.Result` or `.Wait()`?

**Answer:**

You block a thread while the rest of the async method needs a thread to finish. In old ASP.NET / UI that can deadlock. In ASP.NET Core it is less common, but you still waste a thread-pool thread. Under load the app can stall.

**Cross-question:** Is `.GetAwaiter().GetResult()` OK in `Main`?

**Cross-answer:**

Console can use `async Task Main`. In libraries I keep async all the way. Blocking is a last resort.

---

## Q8. `Task` vs `ValueTask` vs `Thread`?

**Answer:**

A **thread** is an OS worker. Expensive.

A **Task** is a piece of work. Waiting on I/O may use **no** thread.

**ValueTask** avoids extra Task objects when the result is often already there. Do not await a ValueTask twice.

**Cross-question:** Does `await Task.Delay(1000)` block a thread for one second?

**Cross-answer:**

No. It is a timer. `Thread.Sleep` does block.

---

## Q9. Delegates, Action, Func, events?

**Answer:**

A delegate is a type-safe “pointer to a method”. `Action` returns nothing. `Func` returns a value. An **event** wraps a delegate so outsiders can only subscribe, not fire it or wipe the list.

**Cross-question:** Why not a public delegate field?

**Cross-answer:**

Anyone could set it to null or invoke it. Also: if the publisher lives long, remember to unsubscribe or you leak memory.

---

## Q10. `IDisposable` and `using`?

**Answer:**

If a type holds files, connections, or other cleanup, it implements `IDisposable`. `using` calls `Dispose` even if an error happens.

**Cross-question:** Should I dispose `HttpClient` every call?

**Cross-answer:**

No. That burns sockets. Use `IHttpClientFactory` and inject `HttpClient`. Do not `new HttpClient()` in a loop.

---

## Q11. `init` and `required`?

**Answer:**

`init` lets you set the property in an object initializer, then it is frozen. `required` means you must set it when you create the object. Nice for DTOs.

**Cross-question:** Auto-property vs backing field?

**Cross-answer:**

Auto is fine until you need extra logic.

---

## Q12. Extension methods?

**Answer:**

A static method that looks like an instance method. First parameter has `this`. LINQ is built this way.

**Cross-question:** Can they touch private fields?

**Cross-answer:**

No. They are just static methods with nicer syntax. If the type already has the same method, that one wins.

---

## Q13. `var` vs `dynamic`?

**Answer:**

`var` is decided at compile time. Same speed as writing the type.

`dynamic` is decided at run time. Typos blow up later. I almost never use `dynamic`.

**Cross-question:** Is `var` slower?

**Cross-answer:**

No.

---

## Q14. Equals and GetHashCode for Dictionary keys?

**Answer:**

If two objects are equal, they must have the same hash. Override both. Do not change those fields after the object is in a dictionary.

**Cross-question:** Why do records help?

**Cross-answer:**

Records generate value equality and a matching hash from the main data. Still: do not mutate a record used as a key.

---

## Q15. `lock` basics?

**Answer:**

`lock (obj)` lets only one thread inside. Lock a **private** object. Never lock `this` or a string.

**Cross-question:** Can I `await` inside `lock`?

**Cross-answer:**

No. Use `SemaphoreSlim.WaitAsync()` for async.

---

## Q16. What are attributes?

**Answer:**

Stickers on code: `[HttpGet]`, `[Required]`, `[Authorize]`. They do nothing alone. Some framework must read them.

**Cross-question:** `[Obsolete]` vs deleting a method?

**Cross-answer:**

Obsolete warns callers first. Deleting is a breaking change.

---

## Q17. Partial classes?

**Answer:**

One class split across files. Designers and source generators use this so your file is not overwritten.

**Cross-question:** Why does EF like partial?

**Cross-answer:**

Generated file can be replaced. Your extra code stays in the other file.

---

## Q18. `nameof` and `$"..."`?

**Answer:**

`nameof(user.Email)` stays correct after rename. `$"Hello {name}"` is interpolation.

**Cross-question:** Can I store a `Span<T>` on a class?

**Cross-answer:**

No. `Span<T>` is a `ref struct` and lives on the stack. Use `Memory<T>` for fields and async.

---

## Q19. What is a lambda?

**Answer:**

A short function: `x => x.IsActive`. Used a lot in LINQ and tests.

**Cross-question:** Closure in a loop?

**Cross-answer:**

The lambda can capture a variable. If you capture the loop index the old `for` way, you can see the last value. Prefer `foreach` and be careful what you capture.

---

## Q20. `IEnumerable` vs `ICollection` vs `IList`?

**Answer:**

`IEnumerable`: can loop. `ICollection`: also Count and Add. `IList`: also index `[0]`. I return the smallest one the caller needs.

**Cross-question:** Why not always return `List<T>`?

**Cross-answer:**

Then callers depend on List methods and can mutate my data. If I later change to an array, I break them.

---

## Q21. What is covariance in simple words? (`IEnumerable<string>` as `IEnumerable<object>`)

**Answer:**

I can treat a list of strings as a sequence of objects **if I only read**. I cannot do that with `List<string>` as `List<object>` because someone could add a non-string.

**Cross-question:** `in` and `out` on interfaces?

**Cross-answer:**

`out T` = producer (covariance). `in T` = consumer (contravariance).

---

## Q22. How do you copy a list without sharing it?

**Answer:**

`list.ToList()` or `[.. list]`. That is a shallow copy. Nested objects are still the same instances.

**Cross-question:** Deep copy?

**Cross-answer:**

I rarely deep-copy whole graphs. I map to a new DTO instead.

---

## Q23. `async Task` vs `Task.Run` for a database call?

**Answer:**

A real async EF call already frees the thread. Wrapping a sync EF call in `Task.Run` just moves blocking to the thread pool. Prefer true async APIs.

**Cross-question:** When is `Task.Run` OK?

**Cross-answer:**

CPU-heavy work you do not want on the request thread, and you have no async API. Not for every SQL call.

---

## Q24. What is a finalizer? Do you write one?

**Answer:**

`~MyClass()` runs some time after GC. I almost never write one. I use `Dispose` and `using`. Finalizers are easy to get wrong and slow GC.

**Cross-question:** What if I forget Dispose?

**Cross-answer:**

The resource can leak until GC, or forever if something still holds it. For files and connections, always `using`.

---

## Q25. `string` intern pool — do you care day to day?

**Answer:**

The runtime can reuse some string literals. I do not intern my own strings in app code. I just know two literals `"ok"` may be the same instance.

**Cross-question:** Why not lock on a string?

**Cross-answer:**

Interned strings are shared. Two parts of the app could lock the same string by accident and deadlock.


---

## Additional Questions

### Q26. What is `IAsyncEnumerable<T>`?
**Answer:** It lets me receive async results one by one with `await foreach`, instead of loading everything first. It is useful for large result sets or paged APIs.

**Cross-question:** When would you prefer it over `Task<List<T>>`?

**Cross-answer:** When the data can be consumed gradually and I do not need the complete list in memory.

### Q27. Why use a `CancellationToken`?
**Answer:** It lets the caller say, “stop this work.” I pass the request token to EF Core, HttpClient, and long-running loops instead of doing work after the client has gone away.

**Cross-question:** Is cancellation automatic?

**Cross-answer:** No. The called method has to observe the token or pass it to an API that supports it.

### Q28. `Dictionary` vs `ConcurrentDictionary`?
**Answer:** A normal dictionary is fine for normal single-threaded access. If multiple threads need to read and update it concurrently, I use `ConcurrentDictionary` or protect the dictionary with a lock.

**Cross-question:** Is ConcurrentDictionary automatically safe for multi-step business logic?

**Cross-answer:** No. A sequence like “check then update” may still need atomic APIs or a lock.
