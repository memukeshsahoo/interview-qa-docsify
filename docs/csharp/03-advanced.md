# C# — Advanced

These separate “I used the syntax” from “I know what the runtime is doing”. Still keep the words simple.

---

## Q1. How does garbage collection work?

**Answer:**

The GC (garbage collector) frees heap objects that nothing points to anymore. New, short-lived objects start in Gen 0. If they survive a collection, they move to Gen 1, and then to Gen 2. Big objects go to a separate large object heap. A collection can pause your threads while it runs.

**Cross-question:** How can a GC language still “leak”?

**Cross-answer:**

You still hold a reference somewhere. Think of a static list, an event handler you never unsubscribed, or a cache with no limit. The GC cannot collect what you still point at. Unmanaged resources also leak if you skip `Dispose`.

---

## Q2. Dispose vs async dispose vs finalizer?

**Answer:**

`Dispose` means “clean up now”. `DisposeAsync` is for cleanup that needs an await, like flushing a network buffer. A finalizer is the last chance. It is slow, and you cannot await inside it. I prefer `SafeHandle` over writing my own finalizer.

**Cross-question:** Why is a finalizer a last resort?

**Cross-answer:**

The object has to stay alive longer so the finalizer thread can run it. And finalizers are hard to write correctly.

---

## Q3. `Span<T>` and `stackalloc`?

**Answer:**

`Span<T>` is a window over existing memory, so you don’t create an extra array when you don’t need one. `stackalloc` puts bytes on the stack. That is fast, but the space is limited. One rule to remember: a `Span` cannot be a class field.

**Cross-question:** When do I need `Memory<T>`?

**Cross-answer:**

In async methods, in fields, and in queues — places where `Span` is not allowed. I take `.Span` only at the point where I actually read or write.

---

## Q4. Covariance and contravariance?

**Answer:**

Covariance (`out T`): an IEnumerable of string can be used as an IEnumerable of object. It works because the type only produces T.

Contravariance (`in T`): an Action of object can be used as an Action of string. It works because the type only consumes T.

`List<T>` is neither, because you both add to it and read from it.

**Cross-question:** Why arrays are a trap?

**Cross-answer:**

A `string[]` can be used as an `object[]`. Then `o[0] = 123` compiles but blows up at runtime. Generics do not allow that hole.

---

## Q5. How is async compiled? What is a sync context?

**Answer:**

The compiler turns an async method into a state machine. Local variables that live across an `await` become fields on that state machine. A synchronization context (old ASP.NET, WinForms) says “continue on this same thread” after the await. ASP.NET Core does not install one by default.

**Cross-question:** When do I use `ConfigureAwait(false)`?

**Cross-answer:**

In libraries. That way I do not force the caller’s UI context, and I lower the deadlock risk. In ASP.NET Core app code it matters less. Don’t use it if the next line has to run on the UI thread.

---

## Q6. Channels vs `Task.WhenAll`?

**Answer:**

`WhenAll` waits for a known set of tasks. A `Channel` is a pipeline: producers put items in, consumers take them out, and it may run forever. You can give it a size limit so you don’t flood memory.

**Cross-question:** `IAsyncEnumerable`?

**Cross-answer:**

You use it with `await foreach` and pull the next item when you are ready. It works nicely for paging through an API.

---

## Q7. Interlocked vs lock vs ConcurrentDictionary?

**Answer:**

`Interlocked` is for simple atomic operations on numbers. `lock` is the usual tool. ConcurrentDictionary is for many threads hitting the same map. But it is not always faster. If there is hardly any contention, a simple lock can win. So I measure.

**Cross-question:** Is `volatile` a lock?

**Cross-answer:**

No. It only limits some reordering and stale reads. I don’t use it for business rules.

---

## Q8. Expression trees vs `Func`?

**Answer:**

An `Expression<Func<...>>` is data — a tree that describes the code. EF reads that tree and writes SQL from it. A `Func` is already compiled code, so EF cannot see inside it.

**Cross-question:** Can I compile an expression?

**Cross-answer:**

Yes. `.Compile()` turns it into a delegate. But then it is no longer a tree, so don’t pass that to EF and expect SQL.

---

## Q9. Source generators — why do they exist?

**Answer:**

They write C# for you at compile time — for example for JSON, logging, and DI. You pay the cost at build time instead of with slow reflection at startup. They are also good for AOT.

**Cross-question:** Generator vs reflection?

**Cross-answer:**

Reflection is flexible but slower. Generators are rigid but fast.

---

## Q10. Nullable reference types — are they runtime safe?

**Answer:**

No. They are only compiler warnings. At runtime, `string` and `string?` are the same type. You can still get null reference errors if you use `!` or turn the warnings off.

**Cross-question:** Then why bother?

**Cross-answer:**

They catch a lot of bugs before production. On new code I treat those warnings as errors.

---

## Q11. Pattern matching that is more than `is int`?

**Answer:**

There are property patterns, relational patterns like `>`, `and` / `or`, and list patterns like `[1, .., var last]`. They make checks shorter and safer than a pile of ifs.

**Cross-question:** Switch vs visitor?

**Cross-answer:**

A switch is fine for a closed set of types. A visitor is for when you keep adding operations over a stable set of types. Most apps are fine with switch plus records.

---

## Q12. Primary constructors — any catch?

**Answer:**

`class Foo(IClock clock)` is great for DI. The catch is that the parameter stays in scope for the whole class. So don’t treat it like a casual local you can change without thinking. For domain objects with rules, a normal constructor that validates can be clearer.

**Cross-question:** Collection expressions `[1,2,3]`?

**Cross-answer:**

Nice syntax for lists and spans. Same idea as collection initializers, just shorter.

---

## Q13. Assembly vs module vs JIT?

**Answer:**

An assembly is the DLL you ship. The CLR (the .NET runtime) loads it, and the JIT compiles each method to machine code the first time it is called — unless you used AOT or ReadyToRun.

**Cross-question:** Does a strong name make the app secure?

**Cross-answer:**

It proves the DLL’s identity and that nobody edited it. It is not encryption, and it is not authentication for users.

---

## Q14. Generic math (`INumber<T>`)?

**Answer:**

Now you can write one method that adds `int` or `double`, because number types share interfaces with static operators. Old C# had no way to say “T must support `+`”.

**Cross-question:** Boxing in generic math?

**Cross-answer:**

Avoiding boxing is the whole point. The constraints use those interfaces, so the JIT can call the right operators directly.

---

## Q15. `unsafe` and pointers — when?

**Answer:**

Interop, image buffers, and very hot parsers. Never in normal business code. Also, pinning objects for too long is bad for the GC.

**Cross-question:** `fixed` vs `GCHandle`?

**Cross-answer:**

`fixed` is scoped to a block. A pinned `GCHandle` is for longer native calls, and you must call `Free` yourself.

---

## Q16. How would you make an immutable model?

**Answer:**

I use records or `init` properties, checks in the constructor, `with` for copies, and read-only lists. Value objects compare by their data. Entities usually compare by id.

**Cross-question:** EF and immutability?

**Cross-answer:**

EF likes to set properties. Fully frozen entities fight the ORM. A common pattern is a mutable entity for storage, with immutable events or DTOs around it.

---

## Q17. Locking on a boxed int or a string?

**Answer:**

Boxing makes a new object every time, so the lock does nothing useful. Strings can be interned, so you might end up locking the same string as some other library. I always use `private readonly object _gate = new()`.

**Cross-question:** `System.Threading.Lock` in newer .NET?

**Cross-answer:**

It is a real lock type, so you don’t lock on the wrong thing by accident. You should still know the classic `lock`.

---

## Q18. What is a memory barrier in simple words?

**Answer:**

It is a “do not reorder this” fence for the CPU and the compiler. You almost never write one by hand. `lock` and `Interlocked` already include the fences you need.

**Cross-question:** Should I use `Thread.MemoryBarrier` in a web app?

**Cross-answer:**

No. Use the higher-level tools.

---

## Q19. `ref` returns and `ref` fields — why do they exist?

**Answer:**

So hot code can pass around a location in memory without copying a big struct. They make it easy to create bugs, like dangling refs. I don’t use them in normal API code.

**Cross-question:** Can a `ref` return point at a local that is gone?

**Cross-answer:**

The compiler tries to stop that. That is why `ref struct` and the ref safety rules exist.

---

## Q20. How do you think about allocation in a hot path?

**Answer:**

I look for boxing, extra strings, LINQ in a tight loop, lambdas that allocate, and `ToList` calls for no reason. I only move to `Span` after measuring. Clear code comes first.

**Cross-question:** Is LINQ always slow?

**Cross-answer:**

No. It adds extra delegates and sometimes extra collections. On a 20-item list nobody cares. In a 10-million-item loop inside a game tick, maybe it matters.

---

## Q21. `ConfigureAwait` on ASP.NET Core — still needed?

**Answer:**

In app code, usually not, because there is no sync context. In library code it is still a good habit. Either way, never block with `.Result`.

**Cross-question:** Can `await` resume on another thread?

**Cross-answer:**

Yes. After I/O, a different thread-pool thread may continue the work. So be careful with thread-local data. And don’t use `Thread.CurrentPrincipal` as your only place to store the user.

---

## Q22. What is IL and why would you look at it?

**Answer:**

IL is the bytecode that C# compiles to. I look at it (with SharpLab or ILSpy) when I need to know if something boxes or allocates. I don’t read IL every day.

**Cross-question:** Does `async` make huge IL?

**Cross-answer:**

The state machine is bigger than a sync method. That is normal. I still use async for I/O.

---

## Q23. Records and `with` — is it a deep copy?

**Answer:**

No. It copies only the top level. Nested objects are still shared unless you replace them too.

**Cross-question:** Record equality with a list inside?

**Cross-answer:**

The generated equality calls the list’s own Equals, and for `List<T>` that does not mean “same items”. People get surprised by this. For lists that should act like values, think it through or write the comparison yourself.

---

## Q24. How would you cancel async work?

**Answer:**

I pass a `CancellationToken` from the HTTP request or from a timeout. I check it inside loops. I pass it on to EF and HttpClient. I don’t ignore it.

**Cross-question:** What if I swallow `OperationCanceledException`?

**Cross-answer:**

Then the caller thinks the work finished. For HTTP, cancellation is normal — the user just navigated away. I let it bubble up as a cancel, not as a 500.

---

## Q25. Difference between parallelism and concurrency?

**Answer:**

Concurrency means many tasks are in progress at once, like async I/O. Parallelism means many CPUs are running at the same time, like `Parallel.For` or `Task.Run` on CPU work. A web API is mostly concurrent I/O, not “use all cores to compute one request”.

**Cross-question:** Should an API `Parallel.ForEach` a database save?

**Cross-answer:**

Usually no. `DbContext` is not thread-safe. And you can hammer the database and make things worse.


---

## Additional Questions

### Q26. What is `IAsyncEnumerable<T>` useful for?
**Answer:** It gives me a stream of async results. I can handle each item with `await foreach` instead of waiting for the whole collection.

**Cross-question:** Does it automatically make database queries cheap?

**Cross-answer:** No. I still need sensible filtering, indexes, and paging. Streaming only changes how I consume the results.

### Q27. What happens when `Task.WhenAll` has multiple failures?
**Answer:** The combined task becomes faulted. If I need all the failures, I look at the individual tasks. I don’t assume only the first operation failed.

**Cross-question:** Would you use it for 10,000 database writes?

**Cross-answer:** No. I would control how many run at once so I don’t overwhelm the database.

### Q28. When is `ValueTask` actually worth using?
**Answer:** When a method very often completes synchronously and the allocation matters. Otherwise I prefer `Task`, because it is simpler and easier to compose.

**Cross-question:** Would you use it everywhere for performance?

**Cross-answer:** No. I measure first. `ValueTask` has usage rules and can make code harder to reason about.
