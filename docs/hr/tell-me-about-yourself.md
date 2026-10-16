# Tell me about yourself

This is usually the first question. They want to hear if you can talk clearly and stay on topic.

Speak for **60–90 seconds**. Do not read your whole CV.

---

## Q1. Tell me something about yourself.

**Answer:**

You can fill in the blanks. Keep it short.

The interviewer is not testing your memory of your CV. They want to see three things: who you are as a developer today, what you work on right now, and what you want next. A simple order that works is: present, then one real project, then what you are looking for. If you ramble about school or every job you ever had, they lose interest before you reach the good part.

> I am a full-stack developer. I work with C#, ASP.NET Core, and Angular. Day to day I build APIs and the screens that use them — login, lists, dashboards, that kind of work.
>
> Right now I work on NriCare, a care-service product. NR users book service providers through a main association. I have worked on the ASP.NET Core API: JWT login, wallets and holds, bookings and quotations, Razorpay top-up, SignalR chat, and TickerQ background jobs. The UI is Angular. The database is PostgreSQL with EF Core.
>
> I like taking a feature from the API all the way to the UI. I also like writing code that the next person can read. I am looking for a role where I can go deeper on the backend and still stay close to Angular.

Notice that the middle part names real features, not just "I know .NET". Each one (JWT login, wallet holds, Razorpay, SignalR, TickerQ) is a hook the interviewer can pick up and ask about. Only name things you are happy to explain in detail.

**Cross-question:** That is a bit general. Tell me **one** project in 30 seconds.

**Cross-answer:**

Here they want depth instead of breadth. Pick one flow and say what made it hard, not just what the screen looked like.

> I built the booking and wallet flow. When an NR user books, we lock the wallet, hold the amount, and open a chat with the provider. The screen was not the hard part. The hard part was avoiding double-spending, splitting the payment when the job is completed, and making sure one association never sees another’s rows.

"Double-spending" means the same money being used twice, for example if two requests hit the wallet at the same moment. Naming the hard part like this shows you understand the risk behind the feature, which is what they are really listening for.

---

## Q2. Walk me through your resume.

**Answer:**

Pick three points, newest first. Skip school unless they ask.

Starting with your latest work matters because that is the work closest to the job you are applying for. Think of it like a trailer for a film: show the best recent scenes, not the whole story from the beginning. Keep each point to one or two sentences and link it to the role you want.

> I will keep this to recent work. Right now I am a full-stack engineer on NriCare — .NET APIs and Angular. I owned pieces of booking, wallet holds, and chat: API, EF queries, and the screens that call them. Before that I built business apps with the same stack, including auth and background jobs.

**Cross-question:** Why are you looking / why did you leave?

**Cross-answer:**

Stay kind. Do not blame people.

Even if your current job has problems, complaining makes the interviewer wonder what you will say about them later. Talk about what you are moving toward, not what you are running from.

> I want a place where I can own bigger backend work — design, performance, helping others — and still ship Angular screens. I am not running away from a person. I want a better fit for growth.

---

## Q3. What are your strengths?

**Answer:**

Give two, with a small proof.

Anyone can say "I am hard-working" or "I am a good problem solver". It means nothing without proof. Pick two strengths and show how you actually use them in your work. That turns a claim into something the interviewer can believe.

> First, I can turn a fuzzy request into a small slice that ships: table, API, screen, and who is allowed to use it. Second, I debug in a calm order. I reproduce the problem, I find which layer is wrong — UI, API, EF, or database — then I fix the real cause, not a patch on top.

A "small slice that ships" means a thin piece of a feature that works from top to bottom, instead of a half-built big feature. The debugging order is like a doctor checking symptoms before giving medicine: first confirm the problem, then find where it really comes from, then treat that.

**Cross-question:** When did one of those strengths go wrong?

**Cross-answer:**

This checks if you are self-aware. Every strength has a downside when you push it too far. Admit it and say what you learned.

> I once built a big abstraction before the need was clear. “Clean architecture” slowed us down. Now I ship the simple version first. I split it out only when a second real use case shows up.

An "abstraction" here means extra layers of code built to handle cases that might come later. If those cases never come, the extra layers just make the code slower to change.

---

## Q4. What is your biggest weakness?

**Answer:**

Use a real one, not “I work too hard”. Then say what you changed.

Interviewers hear "I am a perfectionist" all the time and stop listening. A real weakness, with a clear fix, shows honesty and that you can improve yourself without being told. The fix is the most important part of the answer.

> I used to start coding before I wrote what “done” means. I missed empty lists and “user has no permission”. Now I write three paths first: happy path, empty path, and not allowed. Then I open the editor.

The "happy path" is when everything goes right. The "empty path" is when there is no data to show. "Not allowed" is when the user does not have permission. Thinking of all three before coding is like checking the weather, traffic and fuel before a trip, not just the route.

Do not say you are bad at the main skill of the job.

**Cross-question:** How would your last lead describe you in one line?

**Cross-answer:**

Give a fair line with one small honest minus. Only praise sounds fake.

> Delivers what he promises, asks questions early, and is sometimes too picky in code review. I am working on keeping my comments in line with the real risk.

"In line with the real risk" means a security problem gets a strong comment, while a small naming choice gets a light one or none at all.

---

## Q5. Why should we hire you?

**Answer:**

Match **their** job, not a generic speech.

Read the job post before the interview and pick the two or three things they clearly need. Then show you already do those things. A general "I am passionate and a fast learner" could be said by anyone. A matched answer makes it easy for them to picture you in the role.

> You need someone who can ship a .NET API and an Angular screen in the same sprint, and who does not treat security as extra homework. That is how I already work: check the tenant, use `[Authorize]`, send DTOs not full database rows. I can start helping in week one because this is my stack.

In plain words: "check the tenant" means making sure each customer group only sees its own data. `[Authorize]` is the .NET attribute that blocks a request if the user is not logged in or not allowed. A DTO is a small object that carries only the fields the screen needs, so private columns from the database are not sent out by mistake.

**Cross-question:** What would your first 30 days look like?

**Cross-answer:**

They want to see that you have a calm plan: learn first, then deliver something small, then improve things.

> Week 1: run the app, read the docs, fix a small bug. Week 2 and 3: own one small feature end to end. Week 4: suggest one real improvement based on what I saw, not a random blog idea.

---

## Q6. Describe a conflict at work.

**Answer:**

Use a story: situation, what you did, result.

This is often called the STAR way (Situation, Task, Action, Result). It keeps the story short and easy to follow. The interviewer is checking whether you can disagree politely, protect what matters, and still get the work done.

> A product manager wanted a “quick export” with every column, including internal ids. I wanted to keep private fields out. I offered a safe file with allowed columns, and a follow-up ticket for extra fields. We shipped on time. The extra columns came later when we had a real need.

The good part of this story is that nobody "won" by force. The product manager got the export on time, and private data stayed safe. Offering a middle option is usually better than just saying no.

**Cross-question:** What if they still say “just do it”?

**Cross-answer:**

> I write down the risk in plain words — “this field is personal data” — and I offer a cheaper option. I will not quietly ship a security hole to keep the peace.

Writing it down matters because a spoken warning is easy to forget. A short written note makes the risk clear and lets the right person make the final call with full information.

---

## Q7. Tell me about a production bug you fixed.

**Answer:**

> After login, an association user saw an empty booking list. The screen was fine. The API returned 200 with no rows. The JWT had no `main_association_id`, so the global filter did not match any rows. I logged the association id (not the token), fixed the claim on login, and made SuperAdmin the only role that turns the filter off. Missing association on a normal user means empty or 403, never “all rows”.

In everyday words: a JWT is the login pass the app gives the user after sign-in. It carries small facts about the user, called claims, such as which association they belong to. The "global filter" is a rule on the server that automatically shows each user only their own association's rows. Here, the login pass was missing the association, so the filter found nothing to show. A "200" means the server said "all fine", which is why the bug was hard to spot at first: nothing looked broken, the list was just empty.

The story also shows good habits. Logging the association id instead of the token keeps secrets out of the logs. And the rule "never all rows" means that if something is missing, the system fails safe by showing less, not more.

**Cross-question:** How did you stop it happening again?

**Cross-answer:**

> A check that association A cannot read association B’s booking id, and a rule: SuperAdmin can disable the filter, nobody else.

Fixing the bug once is not enough. The check catches the problem if it ever comes back, and the clear rule about who can turn the filter off stops someone from opening it up by accident.

---

## Q8. Where do you see yourself in 3 to 5 years?

**Answer:**

They want to know if your goals fit what this job can give you, and if you plan to grow instead of standing still. Be honest, and keep it linked to the kind of work this role offers.

> I want to be the person the team trusts with the hard parts: performance, security, and design reviews. I still want to write code, not only slides. Being a lead is fine if it means helping the team, not leaving the keyboard.

**Cross-question:** Do you want to be a manager?

**Cross-answer:**

> I am open to tech lead. People management is a different job. I would take it only if I am ready to be judged on other people’s success.

A tech lead guides the technical side: design, code quality, hard problems. A people manager looks after hiring, growth and team well-being. Showing that you know the difference makes the answer sound thought through.

---

## Q9. Do you have any questions for us?

**Answer:**

Always yes. Try three:

Saying "no questions" can sound like you are not really interested. Good questions also help you decide if you want the job. An interview goes both ways.

1. What does success look like in six months for this role?
2. How do you handle production issues? Who is on call?
3. How do frontend and backend share the API contract?

The "API contract" is the agreement on what the API sends and receives: field names, types and errors. If teams do not share it clearly, the screen and the server break each other often.

**Cross-question:** Why those?

**Cross-answer:**

> They tell me if I will be blocked, if quality is real, and if the API is treated like a product.

Each question maps to one worry: the first shows if goals are clear, the second shows how they deal with problems, and the third shows how well the teams work together.

---

## Q10. How do you explain a technical thing to a non-tech person?

**Answer:**

> I drop the jargon. I say what the user will see, what could go wrong, and how long it takes. Example: instead of “we need a distributed cache”, I say “the second server does not see the first server’s memory, so users may get logged out. Redis is shared memory both servers can use.”

A non-tech person does not need to know how the thing works inside. They need to know what it means for them and their users. Think of how a mechanic talks to you: they do not explain the engine, they say "your brakes are worn, it is not safe, it will take two hours". This question checks if you can work well with product, sales and support people, not just developers.

**Cross-question:** Give a 20-second version of multi-tenant.

**Cross-answer:**

> Each main association’s data is filtered on the server. A user from association A should never see association B, even if they guess a booking id. SuperAdmin is the exception, on purpose.

Multi-tenant means one app serves many separate customer groups, but each group only sees its own data. It is like a block of flats: everyone shares the same building, but each family has its own locked door.

---

## Q11. What are you most proud of?

**Answer:**

Pick one story with a result.

The result is what makes the story land. Without it, the interviewer only hears what you built, not why it mattered. Say what the problem was for users, what you did, and what changed after.

> I am proud of the wallet hold on booking. Users used to worry money would leave the wallet before the job was done. We hold the amount, the chat and quotation can still change, and we only split the payment when the work is completed. Support calls about “I was charged twice” went down because release uses an idempotency key.

A "hold" is like a hotel blocking an amount on your card at check-in: the money is set aside but not taken yet. An "idempotency key" is a unique id sent with the payment request, so if the same request arrives twice, the system knows it already handled it and does not charge again.

**Cross-question:** What would you do differently?

**Cross-answer:**

> I would agree on what “available balance” means with the product team first. Gross `Balance` vs `Balance minus holds` confused people. The code was doing both in different screens. One definition would have saved support time.

This is a good kind of "do differently" answer because it is about a real lesson, not a fake one. When the same word means two things in different screens, users get confused and support gets extra calls. Agreeing on one meaning early avoids that.

---

## Q12. How do you handle a tight deadline?

**Answer:**

> I cut scope, not the quality of the risky parts. We ship the smallest useful version: one screen, one API, auth on. Nice-to-haves go to the next sprint. I say the trade-off out loud so nobody is surprised.

"Cut scope" means doing fewer things, not doing things badly. Think of cooking for guests who arrive early: you serve the main dish properly and skip the dessert, rather than serving everything half-cooked. Security and login stay in, because those are the parts that hurt most if they go wrong.

**Cross-question:** Have you ever missed a date?

**Cross-answer:**

> Yes. I flagged it as soon as I knew, with a new date and what we could still ship. Silent delay is worse than an honest slip.

Saying "never" sounds unreal. What they want to hear is how you handled it. Telling people early gives them time to plan around it, while a late surprise breaks trust.

---

## 90-second script (fill this in)

> I am **[name]**. I have **[X years]** as a full-stack developer in **C# / ASP.NET Core / Angular**.
> Right now I **[one line]**.
> A project I am proud of is **[name]**. The problem was **[…]**. I **[what you did]**. The result was **[…]**.
> I am strongest at **[skill]**.
> I want this role because **[why it matches]**.


---
## Additional Questions
### Q13. Give me your introduction in 30 seconds.
**Answer:** “I am a full-stack developer focused on C#, ASP.NET Core, EF Core, PostgreSQL and Angular. Most of my work is backend APIs, authentication, payments, background jobs and real-time features, while I also build the Angular screens that consume them. I am looking for a role where I can take more ownership of backend design and production problems.”

This is the short version of Q1. Thirty seconds is about three sentences: what you are, what you mostly do, and what you want next. Practise it out loud so it sounds natural, not memorised.
### Q14. Why are you looking for a change now?
**Answer:** “I have learned a lot in my current role. I am now looking for broader backend ownership, stronger engineering challenges and a team where I can keep growing while contributing with the stack I already know.”

The answer starts with something positive about the current job, then moves to what you want next. That keeps it about growth, not complaints, which is what interviewers hope to hear.
### Q15. What do you want from your next role?
**Answer:** “I want real product ownership, code reviews, good engineering practices and enough backend depth to work on performance, architecture and reliability, not only CRUD screens.”

CRUD means Create, Read, Update, Delete: the basic add, view, edit and remove screens most apps have. Saying "not only CRUD" tells them you want harder problems too, like making things fast and keeping them running well.
