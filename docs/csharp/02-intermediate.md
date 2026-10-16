# C# — Intermediate

OOP, generics, LINQ, async. Talk like you use these at work.

---

## Q1. Encapsulation, inheritance, polymorphism, abstraction?

**Answer:**

These are the four main ideas of object-oriented programming. They are all about keeping code organised so it is easier to change later.

**Encapsulation:** hide the inner data. Use private fields and public methods. Think of a bank account: you can't reach in and change the balance directly. You go through `Deposit` and `Withdraw`, and those methods can check the rules (no negative amounts, no overdraft). Without this, any part of the app could put the object into a broken state.

**Inheritance:** a child type reuses a parent. For example, `SavingsAccount` and `CurrentAccount` can both inherit from `Account`, so the shared code (balance, account number) is written once instead of copied into each class.

**Polymorphism:** I call the same method, and the real type decides what runs. If I have a list of shapes and call `Area()` on each one, a circle uses its own formula and a square uses its own. The calling code does not need an `if` for every type, and I can add a new shape without touching it.

**Abstraction:** I depend on `IRepository`, not `PostgresRepository`. My service only knows "there is something that can save and load data". It doesn't care how. That way I can swap the database, or use a fake one in tests, without rewriting the service. It's like driving a car: you use the steering wheel and pedals, and you don't need to know how the engine works.

**Cross-question:** Inheritance or composition?

**Cross-answer:**

I go with composition first — “has a”. I use inheritance only when it is a real “is a”. Deep class trees get painful. Interfaces plus injected helpers scale better.

Composition means a class gets its behaviour by holding other objects, for example an `OrderService` that *has a* payment helper and *has a* email helper. Inheritance means a class *is a* more specific version of another. The problem with deep inheritance is that a change in a parent class quietly affects every child under it, and you end up stuck with behaviour you don't want. With composition I can swap one helper without touching the rest.

---

## Q2. Abstract class vs interface?

**Answer:**

An **interface** is a promise. A type can implement many.

An **abstract class** can hold shared code and fields. A class can have only one parent class.

In plain words: an interface is like a job description. It lists what you must be able to do (`Save`, `Load`), but says nothing about how. A class can sign up to many job descriptions at once, like `IComparable` and `IDisposable` together.

An abstract class is more like a half-built template. It can already contain working code and data that every child shares, and it leaves some parts for the child to fill in. You can't create an abstract class directly, only its children. Because C# allows only one parent class, I use an abstract class when several types really share code, and an interface when I just want different types to offer the same set of methods.

**Cross-question:** Fields on an interface? Constructor on an abstract class?

**Cross-answer:**

Interfaces do not have instance fields. Abstract classes can have constructors and real state.

So an interface can't store per-object data like a `_name` field. An abstract class can, and its constructor runs when a child object is created, which is a good place to set up that shared data.

---

## Q3. Why generics? Why `List<T>` not `ArrayList`?

**Answer:**

Generics keep types safe at compile time, so mistakes show up when I build. `List<int>` stores ints. Old `ArrayList` stores `object` and boxes ints. I get fewer surprises and better speed.

Generics let me write one piece of code that works with many types, while still telling the compiler the exact type each time. `T` is just a placeholder: `List<string>` is a list of strings, `List<Order>` is a list of orders.

With `ArrayList`, everything goes in as `object`, so I could put a string into a list that was meant for numbers and only find out when the app crashes at run time. Also, putting an `int` into an `object` means "boxing": the number gets wrapped in a small extra object on the heap, and unwrapped again when read. That costs memory and time. `List<int>` stores the numbers directly, so there is no wrapping.

**Cross-question:** What is a constraint?

**Cross-answer:**

`where T : class, IEntity, new()` — T must be a class, must implement `IEntity`, must have an empty constructor. Then I can call those members on T.

A constraint is a rule about which types are allowed in place of `T`. Without it, the compiler only knows `T` is "some type", so I can't call anything special on it. With the constraint, the compiler knows every `T` has what `IEntity` promises and can be created with `new T()`, so I can use those safely.

---

## Q4. `IEnumerable<T>` vs `IQueryable<T>`?

**Answer:**

`IEnumerable` runs in memory.

`IQueryable` is an expression tree, which is a description of the query. EF can turn it into SQL.

If I `ToList()` first, then filter, that filter is in memory. If I filter before `ToList()` on a query, it can become a SQL `WHERE`.

Why it matters: imagine a table with a million customers and I only want ten of them. If I call `ToList()` first, the database sends all million rows to my app, and then my app throws most of them away. That is slow and uses a lot of memory. If I keep it as `IQueryable` and add the `Where` first, EF builds SQL with a `WHERE`, and the database sends back only the ten rows I need.

It's like ordering from a shop: `IQueryable` lets me tell the shop exactly what I want before they ship it. Calling `ToList()` too early is like asking them to ship the whole warehouse and sorting it at home.

**Cross-question:** I put my own C# method inside `Where` on EF. What happens?

**Cross-answer:**

EF often cannot turn it into SQL and throws. Keep the filter simple, or filter in memory after you already narrowed the rows.

The reason is that EF has to translate the filter into SQL, and the database knows nothing about my custom C# method. So EF has no SQL to write for it. The fix is to use simple conditions EF understands, or first narrow the rows with SQL and then run my own method on the small result in memory.

---

## Q5. Deferred execution?

**Answer:**

`Where` and `Select` do not run right away. They run when you loop, call `ToList`, or call `Count`. If I change the source before that, results change. If I loop twice, I may hit the database twice.

"Deferred" just means "delayed". When I write `var active = users.Where(u => u.IsActive);`, nothing is filtered yet. I have only written down a recipe. The work happens later, when something actually asks for the results.

This is useful because I can build up a query in steps and only pay for it once. But it can surprise people: if the list changes between writing the query and running it, I get the new data, not the old. And if the source is a database, looping over the same query twice sends the SQL twice.

**Cross-question:** Which LINQ methods run now?

**Cross-answer:**

`ToList`, `ToArray`, `First`, `Any`, `Count`, `Sum`. On EF, they send the SQL right away.

These methods need a real answer (a list, a single item, a number, true or false), so they can't wait. They run the recipe immediately.

---

## Q6. What does async / await actually do?

**Answer:**

`await` means “pause this method until that work finishes, and free the thread for other work”. The compiler builds a state machine for you, which is the code that remembers where to pick up again. I/O like HTTP or SQL does not need a thread sitting and waiting.

A thread is like a worker in the app. When my code asks the database for data, the answer takes some time to come back. Without async, the worker just stands there waiting. With `await`, the worker is sent off to help other requests, and when the database replies, a worker picks up my method right where it left off.

It's like ordering food at a counter and getting a buzzer. You don't stand at the counter blocking the line. You sit down, and the buzzer tells you when it's ready. On a web server, this means the same number of threads can handle many more requests at once.

**Cross-question:** Why is `async void` bad?

**Cross-answer:**

You cannot await it. Errors can crash the app or just vanish. Only event handlers should use `async void`. Everything else gets `async Task`.

The `Task` is what lets the caller know when the work is done and whether it failed. With `async void` there is no `Task`, so the caller can't wait for it and can't catch its exceptions with a normal `try/catch`. Event handlers are the exception only because their signature has to return `void`.

---

## Q7. Deadlock with `.Result` or `.Wait()`?

**Answer:**

It happens when you block a thread while the rest of the async method needs a thread to finish. In old ASP.NET / UI that can deadlock. In ASP.NET Core it is less common, but you still waste a thread-pool thread. Under load the app can stall.

A deadlock is when two things wait for each other forever. In a UI app or old ASP.NET, the async method wants to continue on the same thread that started it. But that thread is stuck on `.Result`, waiting for the async method to finish. Each one waits for the other, and nothing moves. It's like two people in a narrow doorway, each politely waiting for the other to go first.

ASP.NET Core doesn't have that "come back to the same thread" rule, so the classic deadlock is rare. But blocking still holds a thread doing nothing. If many requests do it at once, the thread pool runs out of free threads and the whole app slows down or stops responding.

**Cross-question:** Is `.GetAwaiter().GetResult()` OK in `Main`?

**Cross-answer:**

A console app can use `async Task Main` instead. In libraries I keep async all the way. Blocking is a last resort.

"Async all the way" means if a method I call is async, my method is async too, up to the top. With `async Task Main`, even the entry point can `await`, so there is no need to block at all.

---

## Q8. `Task` vs `ValueTask` vs `Thread`?

**Answer:**

A **thread** is an OS worker, and it is expensive.

A **Task** is a piece of work. While it waits on I/O it may use **no** thread.

**ValueTask** avoids extra Task objects when the result is often already there. Do not await a ValueTask twice.

To put it simply: a thread is a real worker that the operating system has to create and manage, and each one takes memory. Creating lots of them by hand is wasteful. A `Task` is more like a ticket that says "this work will finish at some point". The runtime decides which worker, if any, handles it, so I don't manage threads myself.

`ValueTask` is a small speed-up for hot code. If a method usually has the answer ready straight away (for example, from a cache), creating a new `Task` object every call is extra garbage for the GC to clean. `ValueTask` can skip that. The catch is that it may be reused behind the scenes, so it should be awaited only once.

**Cross-question:** Does `await Task.Delay(1000)` block a thread for one second?

**Cross-answer:**

No. It uses a timer. `Thread.Sleep` is the one that blocks.

`Task.Delay` sets a timer and lets the thread go do other work. When the second is up, the method continues. `Thread.Sleep` keeps the thread stuck doing nothing for the whole second, so no one else can use it.

---

## Q9. Delegates, Action, Func, events?

**Answer:**

A delegate is a type-safe “pointer to a method”. `Action` returns nothing. `Func` returns a value. An **event** wraps a delegate so outsiders can only subscribe, not fire it or wipe the list.

In everyday words, a delegate is a variable that holds a method instead of a value, so I can pass behaviour around. "Type-safe" means the compiler checks that the method has the right parameters and return type. For example, `Func<int, int, int>` holds a method that takes two ints and returns an int, and `Action<string>` holds a method that takes a string and returns nothing.

An event is like a newsletter. Other classes can subscribe (`+=`) and unsubscribe (`-=`), but only the class that owns the event can send it out. That stops other code from triggering it by mistake or removing everyone else's subscriptions.

**Cross-question:** Why not a public delegate field?

**Cross-answer:**

Anyone could set it to null or invoke it. Also, if the publisher lives a long time, remember to unsubscribe, or you leak memory.

With a plain public field, one careless `= null` or `= MyHandler` wipes out every other subscriber. The `event` keyword only allows `+=` and `-=` from outside, which prevents that. On the memory side: when I subscribe, the publisher keeps a reference to my object. If the publisher lives for the whole app and I never unsubscribe, my object can never be cleaned up by the GC.

---

## Q10. `IDisposable` and `using`?

**Answer:**

If a type holds files, connections, or other cleanup, it implements `IDisposable`. `using` calls `Dispose` even if an error happens.

Some things are limited and are not just memory: open files, database connections, network sockets. The garbage collector cleans up memory on its own schedule, and it doesn't know that a connection should go back to the pool *now*. `Dispose` is the method that releases these things right away.

`using` is a safe way to make sure `Dispose` always runs, like a door that closes itself behind you. Without it, if an exception happens halfway through, the cleanup line could be skipped, and the file stays locked or the connection stays open. Do that enough times and the app runs out of connections.

**Cross-question:** Should I dispose `HttpClient` every call?

**Cross-answer:**

No. That burns through sockets. Use `IHttpClientFactory` and inject `HttpClient`. Do not `new HttpClient()` in a loop.

Even after you dispose an `HttpClient`, the operating system keeps its network connection around for a while before fully closing it. Create and dispose one per call under load, and the machine can run out of available sockets. `IHttpClientFactory` reuses connections in a managed way, so I get fresh clients without that problem.

---

## Q11. `init` and `required`?

**Answer:**

`init` lets you set the property in an object initializer, then it is frozen. `required` means you must set it when you create the object. Nice for DTOs.

For example, with `public string Email { get; init; }` I can write `new User { Email = "a@b.com" }`, but after that line nobody can change `Email`. That makes the object safer to pass around, because no other code can quietly change it later.

`required` solves a different problem: forgetting a value. If `Email` is marked `required` and someone writes `new User { }` without it, the code doesn't compile. A DTO (a simple object used to carry data, like a request or response body) often needs both: values that must be filled in, and that shouldn't change afterwards.

**Cross-question:** Auto-property vs backing field?

**Cross-answer:**

An auto-property is fine until you need extra logic.

An auto-property like `public string Name { get; set; }` makes the compiler create the hidden field for me. If I later need to do something when the value is set, like trim spaces or check it isn't empty, I switch to my own private backing field and write the `get` and `set` by hand.

---

## Q12. Extension methods?

**Answer:**

It is a static method that looks like an instance method. The first parameter has `this`. LINQ is built this way.

It lets me "add" a method to a type I don't own, without changing it or inheriting from it. For example, I can write `public static bool IsBlank(this string s)` in a static class, and then call `name.IsBlank()` as if `string` had that method all along. Behind the scenes, the compiler turns it into a normal static call.

That's exactly how LINQ works: `Where`, `Select` and `OrderBy` are extension methods on `IEnumerable<T>`, which is why every list and array seems to have them.

**Cross-question:** Can they touch private fields?

**Cross-answer:**

No. They are just static methods with nicer syntax. If the type already has the same method, that one wins.

Because an extension method lives outside the class, it only sees what any outside code can see: the public members. And if the class has its own method with the same name and parameters, the compiler picks the class's method and ignores the extension.

---

## Q13. `var` vs `dynamic`?

**Answer:**

`var` is decided at compile time. Same speed as writing the type.

`dynamic` is decided at run time. Typos only blow up later, when the code runs. I almost never use `dynamic`.

With `var x = 5;`, the compiler sees the 5 and fixes `x` as an `int` forever. It's just a shortcut so I don't have to type the type name. All the normal checks still happen, so a typo in a method name is caught when I build.

With `dynamic`, the compiler stops checking. It says "trust me, I'll figure it out when the code runs". If I misspell a property, I only find out when that line runs, maybe in production. It's useful in rare cases like some COM or loosely typed data, but for normal code I want the compiler catching my mistakes early.

**Cross-question:** Is `var` slower?

**Cross-answer:**

No.

The compiler swaps `var` for the real type while building, so the final program is exactly the same as if I had written the type out.

---

## Q14. Equals and GetHashCode for Dictionary keys?

**Answer:**

If two objects are equal, they must have the same hash. Override both. Do not change those fields after the object is in a dictionary.

A dictionary finds things fast by using the hash code like a shelf number. It first goes to the shelf for that hash, and then uses `Equals` to find the exact item on that shelf. It's like a library: the shelf number gets you close, and then you read the titles to find the right book.

If I override `Equals` but not `GetHashCode`, two "equal" objects can get different shelf numbers, so the dictionary looks on the wrong shelf and says the key isn't there. And if I change a field that the hash is based on after adding the object, its shelf number changes but the dictionary still has it on the old shelf, so I can't find it anymore.

**Cross-question:** Why do records help?

**Cross-answer:**

Records generate value equality and a matching hash from the main data. Still: do not mutate a record used as a key.

With a record, two objects with the same values count as equal, and the compiler writes `Equals` and `GetHashCode` for me so they always match. That removes the "forgot to override one of them" mistake. But the shelf-number problem from above still applies if I change the values after using it as a key.

---

## Q15. `lock` basics?

**Answer:**

`lock (obj)` lets only one thread inside. Lock a **private** object. Never lock `this` or a string.

When two threads change the same data at the same time, they can step on each other. For example, both read a counter as 5, both add 1, and both write 6, so one update is lost. A `lock` is like a single key to a room: whoever has the key goes in, and everyone else waits outside until it comes back.

The object you lock on is that key. If it's private, only my class can use it. If I lock on `this` or a string, other code outside my class could lock on the same object without knowing, and then the two pieces of code block each other. That can cause slowdowns or even a deadlock.

**Cross-question:** Can I `await` inside `lock`?

**Cross-answer:**

No. Use `SemaphoreSlim.WaitAsync()` for async.

A `lock` belongs to the thread that took it, but after an `await` the method may continue on a different thread. Because of that, the compiler doesn't allow `await` inside a `lock` at all. `SemaphoreSlim` with a count of 1 gives the same "one at a time" behaviour, and `WaitAsync()` lets me wait for my turn without blocking a thread.

---

## Q16. What are attributes?

**Answer:**

They are like stickers on code: `[HttpGet]`, `[Required]`, `[Authorize]`. They do nothing on their own. Some framework has to read them.

An attribute is extra information I attach to a class, method or property. The attribute itself doesn't run any logic. Something else, usually a framework, looks for the sticker at run time and acts on it. For example, ASP.NET Core sees `[HttpGet]` and routes GET requests to that method. It sees `[Authorize]` and checks the user is logged in first. And it sees `[Required]` on a model property and reports a validation error if the value is missing.

This keeps code short. Instead of writing the same "is the user logged in?" check in every method, I put one sticker on it and the framework handles it.

**Cross-question:** `[Obsolete]` vs deleting a method?

**Cross-answer:**

`[Obsolete]` warns callers first. Deleting is a breaking change.

If I delete a method that other code still calls, that code suddenly stops compiling. With `[Obsolete("Use NewMethod instead")]`, everything still works, but callers get a warning telling them to move. Once they have, I can delete it safely.

---

## Q17. Partial classes?

**Answer:**

It is one class split across several files. Designers and source generators use this so your file is not overwritten.

The `partial` keyword tells the compiler: "there are more pieces of this class in other files, join them together when building". The final result is one normal class.

This matters with generated code. Tools like a designer or a source generator write some of the code for you, and they may rewrite their file every time. If my own code were in that same file, it would be wiped out. With partial classes, the tool owns one file and I own the other, and neither touches the other's file.

**Cross-question:** Why does EF like partial?

**Cross-answer:**

The generated file can be replaced, and your extra code stays safe in the other file.

For example, when EF generates entity classes from an existing database, it can regenerate them after the schema changes. If I added helper methods in a separate partial file, they survive that regeneration.

---

## Q18. `nameof` and `$"..."`?

**Answer:**

`nameof(user.Email)` stays correct after rename. `$"Hello {name}"` is interpolation.

`nameof` gives me the name of a variable, property or method as a string, here `"Email"`. If I just typed the string `"Email"` by hand and later renamed the property to `EmailAddress`, the string would silently become wrong. With `nameof`, the rename tool updates it, and if it doesn't match anything the code won't compile. It's handy in error messages, logging and argument checks.

`$"Hello {name}"` is string interpolation: the `$` lets me drop values straight into a string inside `{ }`. It's easier to read than gluing pieces together with `+`.

**Cross-question:** Can I store a `Span<T>` on a class?

**Cross-answer:**

No. `Span<T>` is a `ref struct` and lives on the stack. Use `Memory<T>` for fields and async.

`Span<T>` is a lightweight view over a block of memory, for example part of an array, without copying it. Because it can point to memory on the stack, which goes away when the method returns, C# only lets it live on the stack too. A class lives on the heap and can outlive the method, so it isn't allowed to hold a `Span<T>`. The same rule is why you can't keep one across an `await`. `Memory<T>` is the heap-friendly version that can be stored in fields and used in async code.

---

## Q19. What is a lambda?

**Answer:**

A short function: `x => x.IsActive`. Used a lot in LINQ and tests.

A lambda is a small function with no name, written right where I need it. `x => x.IsActive` reads as "given an `x`, return `x.IsActive`". So `users.Where(x => x.IsActive)` means "keep the users who are active".

It saves me from writing a whole separate method for a one-line check. In tests, I use them to tell a mock what to return or to check what was passed in.

**Cross-question:** Closure in a loop?

**Cross-answer:**

The lambda can capture a variable. If you capture the loop index the old `for` way, you can see the last value. Prefer `foreach` and be careful what you capture.

A closure is when a lambda uses a variable from outside itself. It doesn't take a copy of the value. It keeps a link to the variable. In a `for` loop there is only one `i` that keeps changing, so if I build lambdas inside the loop and run them later, they all see `i` as it is at the end. `foreach` creates a fresh variable on each pass, so each lambda keeps its own value. In a `for` loop, copying `i` into a new local variable inside the loop body fixes it.

---

## Q20. `IEnumerable` vs `ICollection` vs `IList`?

**Answer:**

`IEnumerable`: can loop. `ICollection`: also Count and Add. `IList`: also index `[0]`. I return the smallest one the caller needs.

Think of them as levels of access. `IEnumerable<T>` only says "you can go through the items one by one". `ICollection<T>` adds "you can ask how many there are, and add or remove items". `IList<T>` adds "you can jump straight to item number 5".

Returning the smallest one is about not promising more than I need to. If the caller only needs to loop, `IEnumerable<T>` tells them exactly that, and I stay free to change what's behind it later.

**Cross-question:** Why not always return `List<T>`?

**Cross-answer:**

Then callers depend on List methods and can change my data. If I later change to an array, I break them.

If I hand out my actual `List<T>`, anyone can call `Add` or `Clear` on it and change data my class thinks it controls. And once callers use list-only methods, I'm stuck with `List<T>` forever, because switching to anything else would break their code.

---

## Q21. What is covariance in simple words? (`IEnumerable<string>` as `IEnumerable<object>`)

**Answer:**

I can treat a list of strings as a sequence of objects **if I only read**. I cannot do that with `List<string>` as `List<object>` because someone could add a non-string.

Every string is an object, so reading strings out and treating them as objects is always safe. That's why `IEnumerable<string>` can be passed where `IEnumerable<object>` is expected: `IEnumerable` only lets you read.

`List<T>` lets you write too. If C# let me treat a `List<string>` as a `List<object>`, I could then add a number to it, and the original list of strings would suddenly hold a number. So the compiler blocks it. It's like a box of apples: anyone can take out a fruit, but you can't let someone put an orange into a box labelled "apples only".

**Cross-question:** `in` and `out` on interfaces?

**Cross-answer:**

`out T` = producer (covariance). `in T` = consumer (contravariance).

`out T` means the interface only gives `T` back (like `IEnumerable<out T>`), so a more specific type can be used where a more general one is expected. `in T` means the interface only takes `T` in (like `IComparer<in T>`), so it works the other way: something that can compare any `object` can be used where I need something that compares strings.

---

## Q22. How do you copy a list without sharing it?

**Answer:**

`list.ToList()` or `[.. list]`. That is a shallow copy. Nested objects are still the same instances.

A shallow copy gives me a new list, so adding or removing items in the copy doesn't affect the original. But the items inside are not copied. Both lists point to the same objects. If I change `copy[0].Name`, the original list sees that change too, because it's the same object.

It's like photocopying a list of phone numbers: you now have two pieces of paper, but they still ring the same people.

**Cross-question:** Deep copy?

**Cross-answer:**

I rarely deep-copy whole graphs. I map to a new DTO instead.

A deep copy means copying every nested object too, which is slow and easy to get wrong with objects that point to each other. Mapping to a new DTO gives me fresh objects with just the data I need, so nothing is shared by accident.

---

## Q23. `async Task` vs `Task.Run` for a database call?

**Answer:**

A real async EF call already frees the thread. Wrapping a sync EF call in `Task.Run` just moves the blocking to the thread pool. I prefer true async APIs.

With `await db.Users.ToListAsync()`, while the database is working, no thread is waiting. That's the whole point of async.

With `await Task.Run(() => db.Users.ToList())`, the request thread is freed, but another thread from the same pool now sits blocked waiting for the database. Nothing is saved, and there is a little extra overhead. It's like hiring someone to stand in a queue for you: you're free, but someone is still standing in the queue.

**Cross-question:** When is `Task.Run` OK?

**Cross-answer:**

For CPU-heavy work you do not want on the request thread, when there is no async API. Not for every SQL call.

CPU-heavy means the work is actual calculation, like resizing a big image or crunching numbers, not waiting on something outside. In a desktop app, `Task.Run` keeps that work off the UI thread so the window doesn't freeze.

---

## Q24. What is a finalizer? Do you write one?

**Answer:**

`~MyClass()` runs some time after GC. I almost never write one. I use `Dispose` and `using`. Finalizers are easy to get wrong and they slow down GC.

A finalizer is a last-chance cleanup method. The garbage collector (GC), the part of .NET that frees memory no one uses, calls it before throwing the object away. But I don't know *when* that will be. It could be seconds or minutes later, so it's a bad way to close a file or connection that someone else needs now.

They also cost extra: an object with a finalizer survives at least one more GC round before its memory is freed. And the code runs on a special GC thread, where other objects it refers to may already have been finalized. So I clean up on purpose with `Dispose`, and leave finalizers to low-level classes that wrap raw operating system handles.

**Cross-question:** What if I forget Dispose?

**Cross-answer:**

The resource can leak until GC, or forever if something still holds it. For files and connections, always `using`.

"Leak" here means the file stays locked or the connection stays taken. If something still has a reference to the object, the GC never collects it, so it's never released. With `using`, I don't have to remember.

---

## Q25. `string` intern pool — do you care day to day?

**Answer:**

The runtime can reuse some string literals. I do not intern my own strings in app code. I just know that two `"ok"` literals may be the same instance.

The intern pool is a shared store of strings that .NET keeps so the same text isn't stored many times. If my code has `"ok"` written in two places, the runtime can point both at one stored copy. It's a memory saving that happens on its own.

Day to day I don't think about it. I compare strings with `==` or `string.Equals`, which compare the text, not whether they are the same object. The only place it really matters is the locking rule below.

**Cross-question:** Why not lock on a string?

**Cross-answer:**

Interned strings are shared. Two parts of the app could lock the same string by accident and deadlock.

If my class does `lock ("orders")` and some other library also does `lock ("orders")`, both may get the very same string object from the pool. Now two unrelated pieces of code are fighting over one key without knowing it, which can block or deadlock. A private `object` used only for locking avoids that.


---

## Additional Questions

### Q26. What is `IAsyncEnumerable<T>`?
**Answer:** It lets me receive async results one by one with `await foreach`, instead of loading everything first. It is useful for large result sets or paged APIs.

Normally an async method returns everything at once, like `Task<List<T>>`: I wait until the whole list is ready. With `IAsyncEnumerable<T>`, items come in as they become available, and I can start working on the first one while the rest are still on the way. It's like watching a video while it streams instead of waiting for the full download.

That helps with memory, because I don't hold all the rows at once, and it helps with paged APIs, because I can fetch the next page only when I've finished with the current one.

**Cross-question:** When would you prefer it over `Task<List<T>>`?

**Cross-answer:** When the data can be consumed gradually and I do not need the complete list in memory.

For example, exporting a big table to a file row by row. If I need to sort the whole set or count it first, a normal list is simpler.

### Q27. Why use a `CancellationToken`?
**Answer:** It lets the caller say, “stop this work.” I pass the request token to EF Core, HttpClient, and long-running loops instead of doing work after the client has gone away.

Think of someone who orders food and then leaves the restaurant. Without cancellation, the kitchen keeps cooking a meal nobody will eat. In a web app, if a user closes the page or the request times out, ASP.NET Core marks the request's token as cancelled. If I've passed that token down, the database query or HTTP call stops early, and the server's time goes to requests that still matter.

**Cross-question:** Is cancellation automatic?

**Cross-answer:** No. The called method has to observe the token or pass it to an API that supports it.

The token is just a signal, like a flag being raised. My code has to check it, for example with `token.ThrowIfCancellationRequested()` inside a loop, or hand it to methods like `ToListAsync(token)` that check it for me. If I never pass it on, the work keeps going.

### Q28. `Dictionary` vs `ConcurrentDictionary`?
**Answer:** A normal dictionary is fine for normal single-threaded access. If multiple threads need to read and update it concurrently, I use `ConcurrentDictionary` or protect the dictionary with a lock.

A normal `Dictionary` assumes only one thread changes it at a time. If two threads add items at the same moment, its insides can get corrupted, and you get wrong results or strange exceptions. `ConcurrentDictionary` is built to handle many threads reading and writing at once, so each single operation, like adding or updating one key, is safe. A typical use is a shared in-memory cache inside a web app, where many requests touch it at the same time.

**Cross-question:** Is ConcurrentDictionary automatically safe for multi-step business logic?

**Cross-answer:** No. A sequence like “check then update” may still need atomic APIs or a lock.

Each single call is safe, but two calls in a row are not one unit. If I check `ContainsKey` and then call `Add`, another thread can add the same key in between. Methods like `GetOrAdd`, `AddOrUpdate` and `TryUpdate` do the check and the change in one safe step. If my logic spans several keys or steps, I still need a lock.
