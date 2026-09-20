# Tell me about yourself

This is usually the first question. They want to hear if you can talk clearly and stay on topic.

Speak for **60–90 seconds**. Do not read your whole CV.

---

## Q1. Tell me something about yourself.

**Answer:**

You can fill in the blanks. Keep it short.

> I am a full-stack developer. I work with C#, ASP.NET Core, and Angular. In my day job I build APIs and screens — login, lists, dashboards, that kind of work.
>
> Right now I work on NriCare, a care-service product. NR users book service providers through a main association. I have worked on the ASP.NET Core API: JWT login, wallets and holds, bookings and quotations, Razorpay top-up, SignalR chat, and TickerQ background jobs. The UI is Angular. The database is PostgreSQL with EF Core.
>
> I like taking a feature from the API all the way to the UI. I also like writing code that the next person can read. I am looking for a role where I can go deeper on the backend and still stay close to Angular.

**Cross-question:** That is a bit general. Tell me **one** project in 30 seconds.

**Cross-answer:**

> I built the booking and wallet flow. When an NR user books, we lock the wallet, hold the amount, and open a chat with the provider. The hard part was not the screen. The hard part was not double-spending, splitting payment on complete, and keeping one association from seeing another’s rows.

---

## Q2. Walk me through your resume.

**Answer:**

Pick three points, newest first. Skip school unless they ask.

> I will keep this to recent work. Right now I am a full-stack engineer on NriCare — .NET APIs and Angular. I owned pieces of booking, wallet holds, and chat: API, EF queries, and the screens that call them. Before that I built business apps with the same stack, including auth and background jobs.

**Cross-question:** Why are you looking / why did you leave?

**Cross-answer:**

Stay kind. Do not blame people.

> I want a place where I can own bigger backend work — design, speed, helping others — and still ship Angular screens. I am not running away from a person. I want a better fit for growth.

---

## Q3. What are your strengths?

**Answer:**

Give two, with a small proof.

> First, I can turn a fuzzy request into a small slice that ships: table, API, screen, and who is allowed to use it. Second, I debug in a calm order. I reproduce it, I find which layer is wrong — UI, API, EF, or database — then I fix the real cause, not a patch on top.

**Cross-question:** When did one of those strengths go wrong?

**Cross-answer:**

> I once built a big abstraction before the need was clear. “Clean architecture” slowed us down. Now I ship the simple version first. I split it out only when a second real use case shows up.

---

## Q4. What is your biggest weakness?

**Answer:**

Use a real one, not “I work too hard”. Then say what you changed.

> I used to start coding before I wrote what “done” means. I missed empty lists and “user has no permission”. Now I write three paths first: happy path, empty path, and not allowed. Then I open the editor.

Do not say you are bad at the main skill of the job.

**Cross-question:** How would your last lead describe you in one line?

**Cross-answer:**

> Delivers what he promises, asks questions early, sometimes too picky in code review. I am trying to keep comments in line with real risk.

---

## Q5. Why should we hire you?

**Answer:**

Match **their** job, not a generic speech.

> You need someone who can ship a .NET API and an Angular screen in the same sprint, and who does not treat security as extra homework. That is how I already work: check the tenant, use `[Authorize]`, send DTOs not full database rows. I can start helping in week one because this is my stack.

**Cross-question:** What would your first 30 days look like?

**Cross-answer:**

> Week 1: run the app, read the docs, fix a small bug. Week 2 and 3: own one small feature end to end. Week 4: suggest one real improvement based on what I saw, not a random blog idea.

---

## Q6. Describe a conflict at work.

**Answer:**

Use a story: situation, what you did, result.

> A product manager wanted a “quick export” with every column, including internal ids. I wanted to keep private fields out. I offered a safe file with allowed columns, and a follow-up ticket for extra fields. We shipped on time. The extra columns came later when we had a real need.

**Cross-question:** What if they still say “just do it”?

**Cross-answer:**

> I write down the risk in plain words — “this field is personal data” — and I offer a cheaper option. I will not quietly ship a security hole to keep the peace.

---

## Q7. Tell me about a production bug you fixed.

**Answer:**

> After login, an association user saw an empty booking list. The screen was fine. The API returned 200 with no rows. The JWT had no `main_association_id`, so the global filter did not match any rows. I logged the association id (not the token), fixed the claim on login, and made SuperAdmin the only role that turns the filter off. Missing association on a normal user means empty or 403, never “all rows”.

**Cross-question:** How did you stop it happening again?

**Cross-answer:**

> A check that association A cannot read association B’s booking id, and a rule: SuperAdmin can disable the filter, nobody else.

---

## Q8. Where do you see yourself in 3 to 5 years?

**Answer:**

> I want to be the person the team trusts with the hard path: speed, security, and design reviews. I still want to write code, not only slides. Lead is fine if it means helping the team, not leaving the keyboard.

**Cross-question:** Do you want to be a manager?

**Cross-answer:**

> I am open to tech lead. People management is a different job. I would take it only if I am ready to be judged on other people’s success.

---

## Q9. Do you have any questions for us?

**Answer:**

Always yes. Try three:

1. What does success look like in six months for this role?
2. How do you handle production issues? Who is on call?
3. How do frontend and backend share the API contract?

**Cross-question:** Why those?

**Cross-answer:**

> They tell me if I will be blocked, if quality is real, and if the API is treated like a product.

---

## Q10. How do you explain a technical thing to a non-tech person?

**Answer:**

> I drop the jargon. I say what the user will see, what could go wrong, and how long it takes. Example: instead of “we need a distributed cache”, I say “the second server does not see the first server’s memory, so users may get logged out. Redis is shared memory both servers can use.”

**Cross-question:** Give a 20-second version of multi-tenant.

**Cross-answer:**

> Each main association’s data is filtered on the server. A user from association A should never see association B, even if they guess a booking id. SuperAdmin is the exception, on purpose.

---

## Q11. What are you most proud of?

**Answer:**

Pick one story with a result.

> I am proud of the wallet hold on booking. Users used to worry money would leave the wallet before the job was done. We hold the amount, chat and quotation can still change, and we only split payment when the work is completed. Support calls about “I was charged twice” went down because release uses an idempotency key.

**Cross-question:** What would you do differently?

**Cross-answer:**

> I would agree “available balance” with the product first. Gross `Balance` vs `Balance minus holds` confused people. The code was doing both in different screens. One definition would have saved support time.

---

## Q12. How do you handle a tight deadline?

**Answer:**

> I cut scope, not quality of the risky parts. We ship the smallest useful version: one screen, one API, auth on. Nice-to-haves go to the next sprint. I say the trade-off out loud so nobody is surprised.

**Cross-question:** Have you ever missed a date?

**Cross-answer:**

> Yes. I flagged it as soon as I knew, with a new date and what we could still ship. Silent delay is worse than an honest slip.

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
### Q14. Why are you looking for a change now?
**Answer:** “I have learned a lot in my current role. I am now looking for broader backend ownership, stronger engineering challenges and a team where I can keep growing while contributing with the stack I already know.”
### Q15. What do you want from your next role?
**Answer:** “I want real product ownership, code reviews, good engineering practices and enough backend depth to work on performance, architecture and reliability, not only CRUD screens.”
