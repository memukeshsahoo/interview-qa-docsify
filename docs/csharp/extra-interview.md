# C# — Extra Interview Questions

## Q1. First vs FirstOrDefault vs Single?

**Answer:** `First` returns the first match and throws if there is none. `FirstOrDefault` returns the default when there is none. `Single` expects exactly one match and throws if there are zero or more than one.

The "default" is `null` for classes and things like `0` for an `int`. So the choice is really about what "no result" means in my code. If a missing item is a bug, `First` fails loudly and tells me straight away. If a missing item is normal, like searching for a user who may not exist, `FirstOrDefault` lets me check for `null` and handle it.

`Single` is the strictest. It's like asking "who has the key to this room?" and expecting exactly one person. If nobody has it, or two people do, something is wrong and I want to know.

**Cross-question:** When would you use `Single`?

**Cross-answer:** When the business rule says the result must be unique, like a lookup by a unique email or ID.

If that query ever returns two rows, it means bad data got in, for example a duplicate email. `Single` throws in that case instead of quietly picking one, so the problem gets noticed. There is also `SingleOrDefault`, which returns the default when there are zero matches but still throws when there is more than one.

## Q2. What is SelectMany?

**Answer:** `Select` maps one item to one result. `SelectMany` flattens nested collections into one sequence.

Say I have a list of orders, and each order has a list of items. `orders.Select(o => o.Items)` gives me a list of lists: one bundle of items per order. `orders.SelectMany(o => o.Items)` opens every bundle and pours all the items into one flat list. It's like emptying several shopping bags onto one table.

That's handy when I want to work with all the inner items together, for example to total every item across all orders.

**Cross-question:** Is it the same as Select?

**Cross-answer:** No. Select can produce a sequence of collections; SelectMany produces one flat sequence.

With `Select` I'd need a second loop to reach the inner items. `SelectMany` does that inner loop for me.

## Q3. Why avoid multiple enumeration?

**Answer:** If the source is expensive, every enumeration can repeat the work. With EF Core, that can mean multiple database queries. So I materialize once, meaning I load it into a list, when I actually need to reuse the results.

"Enumeration" means going through the items, for example with `foreach`, `Count()` or `Any()`. A LINQ query is like a recipe, not the finished dish. Each time I go through it, the recipe is cooked again from scratch. If the source is a database, that means sending the same SQL again. If it's a slow calculation, it runs again.

For example, if I call `query.Count()` and then `foreach (var x in query)`, that can be two trips to the database, and the data could even change between the two. Calling `ToList()` once runs the query a single time and keeps the results in memory, so every later use reads from that list.