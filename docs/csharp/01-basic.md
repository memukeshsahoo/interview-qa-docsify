# C# — Basic

If these feel weak, the interview often stops here. Speak in plain words, then add the name of the idea.

---

## Q1. What is C#? How is it different from .NET?

**Answer:**

C# is the **programming language** I write. .NET is the **platform** under it. It gives me the runtime and the libraries I use for things like collections, files, networking, and web APIs. When I compile C# code, it turns into **Intermediate Language (IL)**. When the app runs, the **Common Language Runtime (CLR)** uses the **Just-In-Time (JIT) compiler** to turn that IL into machine code the computer can run. So, simply: **C# is the language, and .NET is the platform that runs it and provides the libraries.**

IL is a middle step: it is not yet code for one specific kind of computer. The CLR is the engine that runs my app. It loads the code, manages memory, and cleans up objects I no longer use. "Just-In-Time" means the IL is turned into real machine code at the moment it is needed, on the machine where the app is running.

An everyday comparison: C# is like the English I write a recipe in, and .NET is the kitchen with the stove, the tools and the ingredients already stocked. The language alone does nothing. The platform is what actually cooks the meal. This matters in an interview because people often mix the two up, and saying it clearly shows I know which part does what.

**Cross-question:** .NET Framework vs .NET Core vs just “.NET”?

**Cross-answer:**

The old .NET Framework is Windows only. .NET Core was the cross-platform rewrite. From .NET 5 onward it is just called .NET. Today we target things like `net8.0` or `net10.0`.

"Cross-platform" means the same app can run on Windows, Linux and macOS. That is why modern .NET apps can run in Linux containers and on cheaper Linux servers. The `net8.0` part is the "target framework" written in the project file. It tells the build which version of .NET the app is made for. .NET Framework still exists for old Windows apps, but new work goes on modern .NET.

---

## Q2. Value type vs reference type?

**Answer:**

A **value type** (`int`, `bool`, `struct`, `DateTime`) holds the data itself. Copy the variable, you copy the data.

A **reference type** (`class`, `string`, arrays) holds a pointer to the object. Copy the variable, both names point at the same object.

An everyday comparison: a value type is like giving someone a photocopy of a page. If they write on their copy, my page does not change. A reference type is like giving someone the address of my house. We both have the address, but there is only one house. If they paint the door, I see the new color too.

This matters because it decides whether a change in one place shows up somewhere else. If I pass a class object into a method and the method changes a property, the caller sees that change. If I pass a struct, the method works on its own copy, and the caller's value stays the same.

I don't say “structs always live on the stack”, because that is not always true. A struct field on a class lives with that object on the heap.

**Cross-question:** Is `string` a value type because it cannot change?

**Cross-answer:**

No. `string` is a reference type. It is immutable, meaning each change makes a new string. Copying a string variable still copies the reference.

It feels like a value type because nobody can change a string in place. When I write `s = s + "!"`, I am not changing the old string. I am making a new one and pointing `s` at it. Anyone else still holding the old string sees the old text. So it behaves safely like a value, but under the hood it is still a reference.

---

## Q3. Stack vs heap?

**Answer:**

The **stack** is for short-lived method data. When the method ends, it is gone.

Think of the stack as a pile of plates. Each time a method is called, a new plate goes on top with that method's local values. When the method finishes, its plate comes off. This is very fast, and nothing needs to clean it up later.

The **heap** is for objects. The garbage collector cleans them when nothing points at them.

The heap is more like a big storage room. Objects can stay there as long as something still refers to them, even after the method that made them has ended. The **garbage collector** is the part of .NET that walks through that room from time to time and throws away objects nobody is using any more. That saves me from freeing memory by hand, but creating lots of short-lived objects gives it more work, which can slow the app down.

**Cross-question:** Where is a `struct` inside a class?

**Cross-answer:**

On the heap, as part of that object.

The class object lives on the heap, and the struct is stored inside it, like a box packed inside a bigger box. Wherever the outer object goes, its struct fields go with it. That is why "structs live on the stack" is not a safe rule.

---

## Q4. Boxing and unboxing?

**Answer:**

Boxing is putting a value like `int` into an `object`. That makes a small heap object. Unboxing pulls it back out.

In plain words: an `int` is a simple value. When code needs to treat it as an `object`, .NET wraps it in a little box on the heap so it can be passed around like any other object. Unboxing opens the box and copies the number back out. Unboxing also checks the type, so trying to unbox it as the wrong type throws an error.

It is extra work. In a loop, prefer `List<int>` over an old `ArrayList` of ints.

`ArrayList` stores everything as `object`, so every int you add gets boxed and every int you read gets unboxed. Doing that thousands of times means thousands of extra small objects for the garbage collector to clean up. `List<int>` is generic, so it stores the ints directly with no boxing.

**Cross-question:** If a method takes `object` and I pass an `int`, is that boxing?

**Cross-answer:**

Yes. The same thing happens with some interface calls on structs. Generics like `where T : IComparable<T>` help you skip boxing.

With a generic constraint like that, the method works with the real type `T` instead of `object`, so the value does not need to be wrapped first. It is the same reason `List<int>` is better than `ArrayList`: the code knows the exact type, so there is no box to make.

---

## Q5. `==` vs `Equals` vs `ReferenceEquals`?

**Answer:**

`ReferenceEquals` asks: same object in memory?

`==` on classes is usually the same, unless the type overloads it. `string` does.

`Equals` can be overridden to mean “same data”.

An everyday comparison: two people can each own the same model of phone. `ReferenceEquals` asks, "Is this the exact same phone?" A well-written `Equals` asks, "Are these the same model, color and storage?" Knowing which question you are asking stops bugs where two objects look the same on screen but the check says they are different, for example when looking for an item in a list.

**Cross-question:** Two objects with the same fields: why is `==` false?

**Cross-answer:**

Because your class did not overload `==`, so it compares references, and they are two different objects. Strings look equal because `string` special-cases `==`.

Each `new` makes a separate object, so by default `==` sees two different objects even if every field matches. If I want "same data means equal", I override `Equals` (and `GetHashCode`) or use a `record`, which does that for me.

---

## Q6. Access modifiers?

**Answer:**

`public` — anyone. `private` — only this type. `protected` — this type and children. `internal` — this project (assembly). There are also `protected internal` and `private protected` for mixed cases.

Access modifiers decide who is allowed to see and use a class or a member. It is like rooms in a house: the living room is open to guests (`public`), the bedroom is only for me (`private`), and some rooms are open to family (`protected`). An "assembly" is the DLL or EXE a project builds into.

This matters because it lets me hide the inner details. If other code can only use my public methods, I can change the private parts later without breaking anyone. It also stops other code from putting my object into a bad state by changing a field directly.

**Cross-question:** Default for a class? For a member?

**Cross-answer:**

A class is `internal` if you write nothing. A member is `private`.

So by default C# keeps things as closed as makes sense: a top-level class is visible only inside its own project, and a field or method is visible only inside its own class. I have to choose to open things up, which is the safer starting point.

---

## Q7. `class` vs `struct` vs `record`?

**Answer:**

A **class** is a reference type. Use it for things with identity, like a user in the database.

"Identity" means the thing is still the same thing even when its details change. A user who changes their email is still the same user.

A **struct** is a value type. Keep it small.

Because a struct is copied every time it is passed around, a big struct means a lot of copying. Small value-like data, such as a point with X and Y, is a good fit.

A **record** is usually a class with “same data means equal”, and a nice `with` copy. Good for DTOs.

A DTO (data transfer object) is a simple object that just carries data, for example the body of an API request. With `with`, I can make a copy that changes one property and keeps the rest, such as `order with { Status = "Paid" }`, without touching the original.

**Cross-question:** Record or class for an EF entity?

**Cross-answer:**

I use a class. EF likes mutable objects with an id. Records are nicer for request/response models.

EF (Entity Framework) tracks an entity and watches its properties change so it knows what to save. That fits a normal class you can change in place. A record's "equal if the data is equal" rule also clashes with the idea that each row is identified by its id, not by its values.

---

## Q8. What is a constructor? Are they inherited?

**Answer:**

A constructor runs when you `new` an object. Its job is to set the object up. Child classes do **not** inherit constructors. If the parent has no empty constructor, the child must call `: base(...)`.

Think of it as the setup step when you open a new account: the bank will not hand you the account until the name and starting details are filled in. A constructor makes sure an object starts in a valid state, so the rest of the code does not have to check for half-built objects.

Since constructors are not inherited, each child class writes its own. With `: base(...)`, the child passes the values the parent needs, so the parent part of the object is set up first.

**Cross-question:** Static constructor?

**Cross-answer:**

It runs once for the type, before first use. It takes no parameters and has no `public`. I use it to set static data.

.NET calls it for me automatically, so I never call it myself. That is why it has no access modifier and no parameters. A typical use is filling a static lookup table once, before anything reads it.

---

## Q9. `const` vs `readonly`?

**Answer:**

`const` is baked in at compile time. It is always static, and it only works with simple values like numbers and strings.

A `const` is fixed forever, like the number of days in a week. The compiler copies the value straight into every place that uses it.

`readonly` can be set in the constructor, so its value can come from runtime.

A `readonly` field can only be set when the object is created, and after that it cannot change. That makes it good for things known only when the app runs, such as a setting read from a config file or a service passed into the constructor. It protects the value from being changed by mistake later.

**Cross-question:** If I change a `const` in one DLL, does the other DLL need a rebuild?

**Cross-answer:**

Usually yes. The value was copied into the caller when it was compiled. `static readonly` is looked up at runtime, so the other DLL can see the new value after you replace the first DLL.

It is like printing a phone number on a flyer. If the number changes, the old flyers still show the old number until you print them again. So I only use `const` for public values that will truly never change, and use `static readonly` for anything that might.

---

## Q10. Array vs `List<T>` vs `IEnumerable<T>`?

**Answer:**

An array has a fixed size. `List<T>` can grow. `IEnumerable<T>` just means “I can foreach this”. With LINQ, it might not have run yet. If it hits the database, I don't loop it twice.

An array is like an egg tray: it has a set number of slots. A `List<T>` is like a shopping bag that stretches as I add items. `IEnumerable<T>` says nothing about storage at all. It only promises I can go through the items one by one.

The "might not have run yet" part is called **deferred execution**. A LINQ query is like a recipe, not the meal. It runs only when I loop over it, and it runs again every time I loop. If that query goes to the database, looping twice means two database calls. When I need the results more than once, I call `ToList()` once and reuse the list.

**Cross-question:** Why return `IReadOnlyList<T>`?

**Cross-answer:**

So the caller cannot `Add` or `Clear` my inner list. It is a smaller promise.

The caller can still read items and use an index, but cannot change my data behind my back. A smaller promise also gives me freedom: I can change how I store the data inside my class later without breaking callers.

---

## Q11. `string` vs `StringBuilder`?

**Answer:**

Strings never change, so `s = s + x` in a loop makes many extra strings. `StringBuilder` edits a buffer instead.

Each `+` makes a brand-new string and copies all the old text into it. In a loop of thousands of items, that is thousands of copies that get longer every time, plus lots of garbage to clean up. A `StringBuilder` keeps one growing piece of memory (a buffer) and adds to the end of it. It is like writing on one long notepad instead of rewriting the whole page every time I add a word.

**Cross-question:** Always use `StringBuilder`?

**Cross-answer:**

No. For a few pieces, `$"{a}-{b}"` is fine and easier to read. I use `StringBuilder` in loops.

For a handful of pieces, the extra cost is tiny, and clear code matters more. `StringBuilder` pays off when the number of pieces is large or unknown, like building a report line by line.

---

## Q12. What can be null? What is `int?`?

**Answer:**

Classes can be null. `int` cannot. `int?` is a wrapper: either a number or “no value”.

Null means "nothing here". A class variable can point at nothing. A plain `int` always holds a number, and its default is `0`. The trouble is that `0` can be a real answer. If a form has an optional "age" field, `0` and "not filled in" mean different things. `int?` lets me tell them apart: I can check `HasValue` before I use the number.

**Cross-question:** `int?` vs `string?`?

**Cross-answer:**

`int?` is a real wrapper at runtime. `string?` is mostly a compiler warning. At runtime it is still a string and can still be null if you ignore warnings.

`int?` is short for `Nullable<int>`, a real type that carries the number plus a "has value" flag. `string?` is part of the nullable reference types feature: the `?` is a note to the compiler that says "this may be null", so it can warn me when I forget to check. The warnings help catch bugs early, but they do not stop a null at runtime.

---

## Q13. `if` vs `switch`?

**Answer:**

I use `if` for conditions. `switch` is nice when I have a list of cases. A switch **expression** returns a value in one go.

If I am checking one value against many options, like an order status of "Pending", "Paid" or "Shipped", a long `if / else if` chain gets hard to read. A `switch` lays the cases out in a clean list. A switch expression goes further: it picks a result and hands it back, so I can write something like "the label for this status" in a few short lines. The compiler can also warn me when I have not handled every case.

**Cross-question:** Give a simple pattern match.

**Cross-answer:**

`if (user is { IsActive: true, Role: "Admin" })` — “if this user is active and admin”.

Pattern matching means checking the shape of an object in one go. This line also checks that `user` is not null, so I don't need a separate null check before reading its properties.

---

## Q14. try / catch / finally?

**Answer:**

`try` holds the risky code. `catch` handles a **specific** error if you can. `finally` always runs, so it is for cleanup. I don't swallow errors, and I don't use exceptions as normal flow.

An exception is .NET's way of saying "something went wrong and I can't carry on", like a file that is missing or a network call that fails. Catching a specific type, such as `FileNotFoundException`, means I only handle the problems I actually know how to fix. Everything else goes up to code that can deal with it or log it.

"Swallowing" an error means catching it and doing nothing. The app looks fine, but the problem is hidden and much harder to find later. Exceptions are also slow compared to a normal `if`, so for expected cases, like a user typing letters into a number box, I check first instead of waiting for a crash.

**Cross-question:** `throw ex;` vs `throw;`?

**Cross-answer:**

`throw;` keeps the original stack. `throw ex;` hides where it first failed. I use `throw;` or wrap it: `throw new AppException("...", ex)`.

The stack trace is the list of method calls that led to the error, like a trail of footprints back to where it started. `throw ex;` starts the trail again from the `catch` block, so the real starting point is lost. Wrapping with `ex` as the inner exception adds my own message and still keeps the original error and its trail inside.

---

## Q15. Namespaces and `using`?

**Answer:**

Namespaces group types so names do not clash. `using System.Linq;` brings names in. `using var x = ...` is different — that **disposes** `x` at the end of the block.

A namespace is like a folder for class names. Two libraries can both have a class called `Logger`, and the namespace tells them apart, like two files with the same name in different folders. The `using` line at the top just saves me from typing the full name every time.

The other `using` is about cleanup. Some objects hold things outside .NET, such as an open file or a database connection. "Dispose" means close and release them. `using var` makes sure that happens at the end of the block, even if an error is thrown, so files and connections are not left open.

**Cross-question:** File-scoped namespace?

**Cross-answer:**

You write `namespace MyApp.Services;` at the top. Same meaning, less nesting. It needs C# 10+.

With the old style, the whole file sits inside `{ }` braces, which pushes every line one level to the right. The file-scoped style applies the namespace to the whole file without the braces, so the code is flatter and easier to read.

---

## Q16. `ref`, `out`, `in`?

**Answer:**

They pass a variable by reference.

Normally C# passes a copy of the value into a method. Passing by reference means the method gets access to the caller's own variable, so a change inside the method can be seen outside it.

- `ref`: already has a value; method can change it
- `out`: method **must** set it
- `in`: read-only reference, useful for big structs

A common example of `out` is `int.TryParse("42", out var number)`. The method returns `true` or `false` to say if it worked, and puts the number into `number`. `in` helps when a struct is large: it passes a reference instead of copying the whole thing, and promises the method will not change it.

**Cross-question:** `out` or a tuple?

**Cross-answer:**

`TryParse`-style `out` is fine. For two results I prefer a small type or `(bool ok, User user)`. Many `out` parameters are hard to read.

A tuple returns several values together, and naming them (`ok`, `user`) makes the call site clear. With many `out` parameters, the reader has to jump between the call and the method to work out what each one means.

---

## Q17. Overload vs override vs `new`?

**Answer:**

**Overload:** same name, different parameters.

For example, `Print(int x)` and `Print(string s)`. The compiler picks the right one based on what I pass in.

**Override:** a child replaces a `virtual` method. Calls use the real type of the object.

The parent marks a method as `virtual` to say "children may change this". If an `Animal` has a virtual `Speak()` and `Dog` overrides it, calling `Speak()` on a dog always gives the dog's version, even when the variable is typed as `Animal`.

**`new`:** hides the parent method. It is easy to get wrong, so I avoid it.

`new` does not replace the parent method. It adds a second method with the same name. Which one runs depends on the type of the variable, not the real object, and that is what catches people out.

**Cross-question:** Variable type is `Base`, object is `Derived`. Which method runs?

**Cross-answer:**

`override` → child method. `new` → parent method, because the variable is `Base`. That surprise is why `new` is a smell.

A "smell" is a sign in code that something may be wrong, even if it compiles. Here, the same object gives different results depending on which type of variable holds it, which is very hard to spot when reading the code.

---

## Q18. What does `static` mean?

**Answer:**

It belongs to the type, not to one object, so there is no `this`. A static class cannot be created with `new`.

Think of a class as a blueprint for houses. Normal fields belong to each house, like its own front door color. A static field belongs to the blueprint itself, so there is only one, shared by everyone. `Math.Max` is a good example: I don't need to create a "Math object" to compare two numbers.

This is handy for small helper methods and shared values. The risk is that static data is shared across the whole app, so if many requests change it at the same time, they can step on each other.

**Cross-question:** Why are static classes hard to test?

**Cross-answer:**

I cannot easily fake `DateTime.Now` or `File.ReadAllText`. I wrap them in a small interface and inject it.

In a test I want full control. For example, I want "now" to be a fixed date so the result is the same every run. Code that calls `DateTime.Now` directly always gets the real clock. If the code asks for an interface like `IClock` instead, the real app passes the real clock, and the test passes a fake one that returns whatever date I choose.

---

## Q19. What is a property?

**Answer:**

A property looks like a field (`user.Name`) but it is get/set methods. I can add checks later without changing callers.

From the outside it reads like a simple variable, but behind it there are small methods: "get" runs when someone reads the value and "set" runs when someone changes it. So if later I need a rule, like "the name cannot be empty", I add it inside the setter. Everyone still writes `user.Name = "Asha"`, and none of their code has to change.

**Cross-question:** `{ get; set; }` vs a full backing field?

**Cross-answer:**

I start with auto properties. I add a backing field when I need extra logic.

With `{ get; set; }`, the compiler creates the hidden field for me, so there is less code to write. A backing field is a private field I declare myself, which lets me add steps such as checking or trimming the value before storing it.

---

## Q20. `if (x == null)` vs `is null` vs `??`?

**Answer:**

`is null` is nice because it cannot be overloaded. `??` means “if left is null, use the right side”. `?.` stops if the left side is null.

A class can change what `==` means, so in rare cases `x == null` might not do a plain null check. `is null` always checks for null, no matter what the class does. `??` gives a fallback value, for example `name ?? "Guest"`. `?.` lets me read through something that may be missing: `user?.Address?.City` gives null instead of crashing when `user` or `Address` is null. Together they remove a lot of "object reference not set" errors.

**Cross-question:** `??` vs `||`?

**Cross-answer:**

C# does not use `||` for defaults the way JavaScript does. I use `??` for null. For `int?`, `value ?? 0` is the usual pattern.

In C#, `||` only works with true/false values. It means "or" in a condition. `??` is the operator built for "use this if the value is missing".

---

## Q21. What is an enum?

**Answer:**

It is a named set of numbers, like `Open` and `Closed`. It is easier to read than magic ints. I still store them carefully in the database and do not assume the number will never change.

A "magic int" is a bare number in code, like `if (status == 2)`, where nobody can tell what `2` means without digging. With an enum, the code says `status == TicketStatus.Closed`, which explains itself. The compiler also catches typos in the names.

Behind each name is still a number. If someone adds a new value in the middle of the list or reorders it, the numbers can shift, and the values already saved in the database would then mean something else. So I give the numbers fixed values, or store the name as text.

**Cross-question:** Enum vs a lookup table?

**Cross-answer:**

An enum is for a closed list that rarely changes and is used in code. A table lets the business add values without a new build. Status values that product changes often belong in data.

For example, the days of the week fit an enum well. A list of product categories that the business team adds to every month fits a table, because adding a row needs no code change and no new release.

---

## Q22. What is LINQ in one sentence?

**Answer:**

LINQ is a way to filter and map lists in C#, like `Where` and `Select`. In memory it loops objects. With EF it can become SQL.

`Where` keeps only the items that match a rule, and `Select` turns each item into something else. For example, from a list of orders I can get the ids of only the paid ones in one readable line instead of writing a loop by hand. When the data comes from a database through EF, the same kind of query is turned into SQL, so the filtering happens in the database and only the matching rows come back.

**Cross-question:** `Count()` vs `.Count` on a list?

**Cross-answer:**

On `List<T>`, `.Count` is a property and is cheap. `.Count()` is LINQ and still fine on a list, but on an `IQueryable` it can run `SELECT COUNT` in the database. Know which type you have.

`.Count` is a number the list already keeps, so reading it is instant. `IQueryable` means a query that has not run yet, usually against a database, so `.Count()` there turns into a database call. That is fine once, but calling it again and again, for example inside a loop, means a trip to the database each time.

---

## Q23. `string.IsNullOrEmpty` vs `IsNullOrWhiteSpace`?

**Answer:**

Empty means `""`. White space means `"   "` too. For user input I usually use `IsNullOrWhiteSpace`.

If a user types only spaces into a "name" box, `IsNullOrEmpty` says the value is fine, because it is not empty. In practice that name is blank. `IsNullOrWhiteSpace` also treats spaces, tabs and new lines as "nothing", so it catches that case.

**Cross-question:** Does it catch `" null "` as text?

**Cross-answer:**

No. That is a real string of characters. Only trimming and extra checks would catch junk like that.

The word "null" typed as text is just four letters to C#, not a missing value. If an app needs to reject it, I trim the input and compare it myself.

---

## Q24. What is an interface in simple words?

**Answer:**

It is a contract that says “you must have these methods”. A class can implement many interfaces. I program against `IEmailSender`, not `SmtpEmailSender`, so tests can pass a fake.

Think of a wall socket. Any appliance with the right plug works, and the socket does not care what is inside the appliance. An interface is the shape of the plug. My code only needs "something that can send an email". In the real app that is the SMTP sender. In a test it is a fake that just records the email instead of sending it.

This also makes changes easier. If we switch to a different email provider, I write a new class that follows the same interface, and the rest of the code does not change.

**Cross-question:** Can an interface have code?

**Cross-answer:**

Yes, default interface methods exist. I keep them thin. Shared heavy code still belongs in a class.

A default method gives a basic version that classes get for free, which is useful for adding a new method to an interface without breaking every class that already uses it. But an interface cannot hold instance fields, so real logic with its own data fits better in a class.

---

## Q25. `DateTime` vs `DateTimeOffset` vs `DateOnly`?

**Answer:**

`DateTime` is a date and time, but the timezone is easy to mess up. `DateTimeOffset` includes the offset from UTC. `DateOnly` is just a date, like a birthday.

UTC is the world's base clock that time zones are measured from. India, for example, is UTC+5:30. A `DateTime` like "10:00" does not always tell you which zone it means, so the same value can be read as two different moments. A `DateTimeOffset` like "10:00 +05:30" points to one exact moment anywhere in the world. A birthday has no time at all, so `DateOnly` stops a time zone shift from accidentally moving it to the day before.

**Cross-question:** What do you store in the database for “when was this created”?

**Cross-answer:**

UTC. I convert it to local time for the screen. Mixing local times from servers in two countries is a classic bug.

If one server saves its local time in India and another saves its local time in Europe, the same moment gets saved as two different clock times, and sorting or comparing them goes wrong. Saving everything in UTC gives one shared clock. Each user then sees it in their own time zone.


---
## Additional Questions
### Q26. What is the difference between a class and a struct?
**Answer:** A class is a reference type and a struct is a value type. I normally use classes for entities and services, and small structs/record structs for small value-like data. When I pass a class around, everyone shares the same object. When I pass a struct, each place gets its own copy. That is why structs suit small things like a point or an amount of money, where copying is cheap and each copy standing on its own is what I want.
### Q27. What is a nullable value type?
**Answer:** `int?` means an int can also be null. It is useful when null has a real meaning, like an optional database column. A plain `int` defaults to `0`, and `0` can be a real value. For example, a discount of `0` is different from "no discount set yet". With `int?`, I can check `HasValue` and handle the "no value" case on purpose.
### Q28. What is an enum?
**Answer:** An enum gives readable names to a fixed set of values, like `BookingStatus.Pending`. I prefer it over magic numbers or strings when the set is controlled by the code. Code like `status == BookingStatus.Pending` explains itself, while `status == 1` makes the reader guess. The compiler also catches a misspelled name, which it cannot do for a plain string like `"Pendng"`.
