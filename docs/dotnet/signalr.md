# SignalR (talk from a previous project)

Use **“in my last project”** language. Keep it honest and simple.

---

## Q1. What is SignalR and where did you use it?

**Answer:**

SignalR is the real-time messaging library in ASP.NET Core. Instead of the browser asking again and again, the server **pushes** updates to the browser, usually over WebSockets (with fallbacks if those are not available). I reach for it when polling every few seconds is too slow or too chatty.

A normal web page works like sending a letter: the browser asks, the server answers, and the conversation ends. If the browser wants news, it has to ask again. A WebSocket is more like an open phone line: once connected, either side can talk at any time. SignalR handles that phone line for you. It picks the best way to connect, reconnects when needed, and lets the server call a named function on the client, like `incidentUpdated`.

Why it matters: without it, a screen only changes when the user refreshes or when the app polls on a timer. Users end up looking at old data and acting on it.

**In my previous project:** we used SignalR so agents saw ticket changes **live** — someone took a ticket, a new incident came in, a chat/comment landed — without refreshing the Angular board.

**Example:**

```csharp
public class IncidentHub : Hub
{
    public override async Task OnConnectedAsync()
    {
        var tenant = Context.User!.FindFirst("tenant")!.Value;
        await Groups.AddToGroupAsync(Context.ConnectionId, tenant);
        await base.OnConnectedAsync();
    }
}

// after SaveChanges in an API
await hub.Clients.Group(tenantKey).SendAsync("incidentUpdated", dto);
```

```typescript
const conn = new signalR.HubConnectionBuilder()
  .withUrl('/hubs/incidents', { accessTokenFactory: () => auth.accessToken() })
  .withAutomaticReconnect()
  .build();

conn.on('incidentUpdated', row => store.apply(row));
await conn.start();
```

**Real-world example:**

Two agents opened the same new incident and both clicked **Take**. Without SignalR, both thought they owned it. With SignalR plus a row version on the API, the second agent’s board moved the card to “Taken by Priya” within a second.

**Cross-question:** Why not `setInterval` HTTP GET?

**Cross-answer:**

Polling is simpler, I agree. But with 50 agents polling every 2 seconds, you hammer the API. That's 25 requests every second, and most of them come back with "nothing changed". It's like a child in the back seat asking "are we there yet?" every few seconds. SignalR sends **only when something changes**. That said, for a KPI that can be 30 seconds stale, polling is fine.

---

## Q2. How do you authenticate SignalR?

**Answer:**

The same way as the REST API, with the same user. The JWT goes on the connection, and I put `[Authorize]` on the hub. `[Authorize]` means only logged-in users with a valid token can connect at all. Anyone else is turned away before they reach any hub code.

Even then, I **never** trust a tenant id sent by the client. I read the tenant from the **token** and put the connection in that tenant’s group. Anything the browser sends can be changed by the user with a few clicks in developer tools. The token is signed by the server, so its contents can't be changed without breaking the signature. That's why the token, not the request, decides which tenant you belong to.

**Example:**

```csharp
[Authorize]
public class IncidentHub : Hub { }
```

**Real-world example:**

A tester passed another tenant in the query string. The hub ignored it and used the claim from the token, so they did not get the other hospital’s events.

**Cross-question:** Token in `?access_token=`?

**Cross-answer:**

Yes, that is common for WebSockets, because browsers can't set an `Authorization` header when they open a WebSocket. I just treat it as sensitive: a short-lived access JWT, no logging of full URLs, and HTTPS only. URLs often end up in server and proxy logs, so a short life means a token found in a log is already useless.

---

## Q3. How do you send to the right users only?

**Answer:**

SignalR gives you a few ways to choose who gets a message. Think of it like a school announcement system: you can talk to one student, one classroom, or the whole school.

- `Clients.Caller` — only me  
- `Clients.Others` — everyone else on the hub (careful in multi-tenant)  
- `Clients.Group(tenantKey)` — this tenant  
- `Clients.User(userId)` — one user  

**Default for a tenant product: Groups named by tenant.** Never `Clients.All` in a multi-tenant app.

A multi-tenant app serves many separate customers (tenants) from the same servers. Each customer must only ever see their own data. `Clients.All` is the whole-school loudspeaker: every connected user from every customer hears it. A group named after the tenant is one classroom, so only that customer's users get the message.

**Example:**

```csharp
await Clients.Group(tenantKey).SendAsync("commentAdded", incidentId, preview);
```

**Real-world example:**

A comment in tenant A showed up on tenant B’s board because the code used `Clients.All`. We switched to `Group(tenant)` and added a test for it.

**Cross-question:** User in two tenants?

**Cross-answer:**

We keep one connection per token/tenant. Switching tenant means getting a new token and reconnecting. That way each connection belongs to exactly one tenant group, and there's no chance of one connection getting events from both.

---

## Q4. How does SignalR scale to two servers?

**Answer:**

Not on its own. Each server keeps its own list of WebSocket connections. So a user connected to server 1 will not get a push sent from server 2 unless you add a **backplane** (Redis is the usual one) or a shared message bus. A backplane just passes each message to every server. Sticky sessions can hide the bug in a small setup, and then it fails later.

Picture two receptionists in two different buildings, each with their own list of visitors. If a message for a visitor arrives at building 2 but that visitor is sitting in building 1, it never reaches them. A backplane is a shared radio channel between the receptionists: every message is announced to both buildings, and whoever has that visitor passes it on.

Sticky sessions mean the load balancer keeps sending the same user to the same server. That can make things look fine in testing, but two users who need to see each other's updates can still land on different servers.

**Example:**

```csharp
builder.Services.AddSignalR().AddStackExchangeRedis(redisConnection);
```

**Real-world example:**

We added a second API node behind Nginx. The live “ticket taken” update worked only if both agents happened to land on the same node. Adding the Redis backplane made it work across nodes.

**Cross-question:** Hangfire vs SignalR?

**Cross-answer:**

They solve two different problems. Hangfire is for background jobs like emails and reports: work that can run a bit later, out of the user's way, and should be retried if it fails. SignalR is for live UI: telling screens that are open right now that something changed. In our flow, after the save we wrote to an **outbox** (or published an event), the worker sent the email, and the request also notified the hub. An outbox is a table where you record "this still needs to be sent" in the same save as the data, so the email isn't lost if something crashes after the save.

---

## Q5. How did Angular consume SignalR in your project?

**Answer:**

We had **one** connection, owned by a **root service**. A root service in Angular is created once and shared by the whole app, so every screen talks to the same object. Screens subscribed to its events (or read a signal). We did not call `start()` in every component. On logout we called `stop()`, and we used `withAutomaticReconnect()` to handle short network blips.

Why one connection: every connection is an open socket on the server. If each screen opened its own, one user could hold several sockets, get the same event more than once, and leave sockets open after leaving a page. One shared connection is like one phone line in the house with an extension in each room.

Unrelated components, like the board and a toast, both listened to the same service — the same pattern as a shared state service.

**Example:**

```typescript
@Injectable({ providedIn: 'root' })
export class IncidentHubService {
  private conn?: signalR.HubConnection;
  readonly lastEvent = signal<IncidentEvent | null>(null);

  async start(token: string) {
    this.conn = new signalR.HubConnectionBuilder()
      .withUrl('/hubs/incidents', { accessTokenFactory: () => token })
      .withAutomaticReconnect()
      .build();
    this.conn.on('incidentUpdated', e => this.lastEvent.set(e));
    await this.conn.start();
  }
}
```

**Real-world example:**

The board and a small “live” dot in the header both used `IncidentHubService`, so we did not open two sockets.

**Cross-question:** Connection in `ngOnInit` of the board only?

**Cross-answer:**

Then as soon as you leave the board, the live toasts on other screens stop. The connection's life would be tied to one page, but the live updates are needed across the whole app. A root service is better, because it lives as long as the app does.

---

## Q6. What can go wrong with SignalR in production?

**Answer:**

Most SignalR problems don't show up on a developer's laptop. They appear once there's a proxy, a load balancer, more servers and real users in between.

- Proxy/Nginx must allow WebSockets (`Upgrade` headers). A WebSocket starts as a normal HTTP request that asks to "upgrade" to a live connection. If the proxy drops those headers, the upgrade never happens.  
- A load balancer with no backplane. Users on different servers don't see each other's updates (see Q4).  
- Broadcasting too much (full entity graphs). Send small DTOs instead (id, status, by whom). A DTO is a small object made just for sending data, holding only the fields the receiver needs.  
- Not handling reconnect, so the UI gets stuck on “connecting”.  
- Using SignalR for large file transfers — don’t; use a REST download.

**Example:**

```csharp
await Clients.Group(tenant).SendAsync("incidentUpdated", new { dto.Id, dto.Status, dto.TakenBy });
```

**Real-world example:**

At first we pushed the full incident with all its comments. The messages were huge and the board stuttered. We switched to a small DTO; the board either already had the rest or re-fetched that one row.

**Cross-question:** SignalR for CRUD instead of REST?

**Cross-answer:**

No. Creates and updates still go through REST, because that is where we get validation, status codes and ProblemDetails. ProblemDetails is the standard error format the API returns, so the client knows exactly what went wrong. SignalR **notifies**. REST **changes** data. It's like a shop: you place the order at the counter (REST), and the buzzer tells you when it's ready (SignalR).

---

## Q7. Mini story you can memorize (30 seconds)

**Answer:**

> In my previous project we used SignalR on the incident board. When an agent took a ticket, the API saved it with a concurrency token, so two agents could not take the same ticket at once. Then it notified the tenant group. Angular had one hub service, shared by every screen. Other agents saw the card move right away, without refreshing. We grouped by tenant so we never broadcast to everyone. When we scaled to two servers we added a Redis backplane so updates reached users on both servers. Auth was the same JWT as the REST API.

**Cross-question:** What did you learn?

**Cross-answer:**

Live UI is easy to demo, and just as easy to leak across tenants if you use `Clients.All`. One wrong line sends one customer's data to every other customer's screen. Groups, small DTOs and REST for writes — that is the safe setup.

---

## Q8. How does SignalR work in NriCare (current project)?

**Answer:**

The hub is `ChatHub`, mapped at `/chat/hub`. A hub is the server-side class that clients connect to and call methods on, a bit like a controller for live messages. The JWT can come in as `?access_token=` because browsers cannot set the auth header on a WebSocket. On connect, I put the connection in a group named with the `user_id`. That way, sending to that group reaches every device the user has open.

`SendMessage` saves the row first, then sends `ReceiveMessage` to the **participant groups**, not `Clients.All`. Saving first means the message is safe in the database even if the live push fails. Sending only to the participant groups means only the people in that chat get it. A chat message can also carry a quotation id. Push notifications for offline users run in a new DI scope inside `Task.Run`, so the sender doesn't have to wait for the notification to go out. A new DI scope is needed because the hub's own services (like the database context) belong to the original call and shouldn't be used after it finishes.

**Example:**

```csharp
app.MapHub<ChatHub>("/chat/hub").RequireCors("SignalRCors");

// JWT events
if (!string.IsNullOrEmpty(accessToken) && path.StartsWithSegments("/chat/hub"))
    context.Token = accessToken;
```

**Real-world example:**

An NR user and a service provider chat about a booking. Quotation messages use `ContentType.Quotation`. Presence (`UserOnline` / `UserOffline`) currently goes out to `Clients.All`, which I would restrict. The connection map is in-memory, so it only works per server. If we ran a second server, each server would only know about its own connections.

**Cross-question:** Is `[Authorize]` on the hub?

**Cross-answer:**

No, the hub is not marked `[Authorize]`. Right now it throws `HubException("Unauthorized")` if `user_id` is missing. I would add `[Authorize]` so unauthenticated sockets never even reach `OnConnectedAsync`. That moves the check to the front door: the framework rejects the connection before any of our code runs, instead of us remembering to check inside the hub.


---
## Additional Questions
### Q9. What is the difference between SignalR and REST?
**Answer:** REST is request-response, and it is good for CRUD. SignalR is for real-time events from the server to the client. In a normal app I use REST to change data and SignalR to tell connected clients that something changed.

With REST, the client always starts the conversation: it asks, the server answers, done. That suits creating, reading, updating and deleting data (CRUD), because each action gets a clear answer and a status code. With SignalR, the server can speak first, the moment something happens. So REST is like phoning a shop to ask if your order is ready, and SignalR is the shop texting you the moment it is.

### Q10. How do you handle reconnects?
**Answer:** I turn on automatic reconnect, show the connection state in the UI, and make sure the client can resync any data it missed after reconnecting. A socket is not a reliable database.

Networks drop all the time: Wi-Fi switches, a laptop sleeps, a server restarts. Automatic reconnect brings the connection back, but any messages sent while it was down are simply gone. Showing the state ("reconnecting...") tells the user the screen might be a bit behind. Reloading the data from the API after reconnecting fills in whatever was missed, like catching up on a group chat after your phone was switched off.

### Q11. Why should SignalR messages be small?
**Answer:** Large messages mean more work for the network and the browser. I usually send just the id and the changed fields, and let the client update or refetch the record it needs.

The same message often goes to many connected users at once, so a big message is multiplied by every receiver. The browser also has to process each one and redraw the screen, which makes the page feel slow. A small message is like a text saying "your parcel has arrived": you don't need the whole parcel in the message, just enough to know what changed.

### Q12. What was the biggest difficulty you faced while implementing SignalR?
**Answer:** The hardest part was making the real-time connection reliable outside local development. The hub worked fine locally, but once the client was deployed we hit issues with the WebSocket connection, CORS and the Nginx proxy configuration.

Locally there's usually nothing between the browser and the API. In a deployed setup, there's a proxy in the middle, HTTPS, and the client runs on a different address from the API. CORS is the browser rule that decides which websites may call your API, so it had to allow the real client address. Nginx had to be told to pass the WebSocket upgrade through instead of treating it as a normal request.

I checked the network logs, verified the hub URL and the JWT, configured the proxy for the WebSocket upgrade, and fixed CORS for the real client origin. We kept REST APIs for chat history and SignalR for real-time events, which made reconnecting easier.

### Q13. Why did you use REST APIs and SignalR together for chat?
**Answer:** I used REST for chat list and message history because it is easier to query, paginate and load again. SignalR is better for new messages and read receipts because they need real-time delivery. If the socket disconnects, the app can reconnect and fetch the latest history through REST.

Each tool does what it's good at. History is a lot of data that you load page by page (paginate) when the user opens a chat or scrolls up, and REST does that well. New messages are small and need to show up instantly, which is what SignalR is for. Together they cover each other: if the live connection drops, REST can always bring the screen back up to date.

### Q14. What problem did you face with SignalR behind Nginx?
**Answer:** Normal HTTP proxying worked, but the WebSocket connection was not reliable. I checked the Nginx location block for the hub and made sure the WebSocket `Upgrade` and `Connection` headers were forwarded correctly. I also checked the hub path, SSL and the CORS configuration.

Nginx sits in front of the API and passes requests along. A WebSocket starts as a normal HTTP request with `Upgrade` and `Connection` headers that say "switch this to a live connection". By default, Nginx doesn't pass those headers on, so the API never sees the request to upgrade. SignalR may then fall back to a slower method or fail. Forwarding those headers in the hub's location block lets the upgrade go through.

### Q15. How did you authenticate SignalR with JWT?
**Answer:** The SignalR connection uses the same JWT authentication as the API. For the WebSocket connection, the token can come through `access_token`. On the server I validate the token and use the authenticated user’s details from it. I do not trust a user id sent by the client for authorization.

The `access_token` query value is needed because browsers can't add an `Authorization` header when opening a WebSocket. The server reads the token from there, checks its signature and expiry just like a normal API call, and then knows who the user is. The user id comes from inside the signed token, because a user id sent separately by the client could be changed to anyone's id.

### Q16. What happens if a user is offline when a message is sent?
**Answer:** Nothing is lost, because I do not depend on SignalR to store the message. I save it in the database first and then send the real-time event. If the user is offline, the message just stays in the database. When they come back, the app loads the missed messages through the REST API, and we can also send a push notification if needed.

SignalR only delivers to people who are connected right now. It doesn't keep messages for later. So the database is the mailbox, and SignalR is the doorbell. If nobody is home, the doorbell does nothing, but the letter is still in the mailbox when they get back.

### Q17. How did you handle reconnects?
**Answer:** I used automatic reconnect on the Angular SignalR connection. But reconnect alone is not enough, because some messages can be missed while the socket is down. So after a reconnect, I refresh the relevant chat data from the API to bring the UI back in sync.

Automatic reconnect only fixes the connection. It doesn't replay what was sent while you were away. For example, if a user's train goes through a tunnel for 30 seconds and two messages arrive in that time, the reconnect alone won't show them. Reloading the chat from the API after reconnecting brings in those two messages.

### Q18. How did you handle read and unread messages?
**Answer:** The read state is stored in the database, because it has to survive a reconnect. SignalR is only used to tell the other participant that the message or chat was read. So the database is the source of truth, and SignalR updates the UI right away.

"Source of truth" means the one place you trust when there's any doubt. If read status only lived in SignalR events, it would be lost the moment someone refreshed the page or lost connection. Saving it in the database means it's always correct when the chat is loaded again, and SignalR just makes the "read" tick appear instantly for the other person.

### Q19. How did you prevent duplicate messages?
**Answer:** I make the database operation idempotent, meaning running it twice has the same effect as running it once. Each message should have a unique business or client request id, so a retry does not create another row. I save the message first and then broadcast the saved message. I don't treat the SignalR event itself as the database operation.

Duplicates usually come from retries: the network is slow, the client isn't sure the send worked, so it sends again. With a unique request id, the server sees the second attempt, recognises the same id, and doesn't save it twice. It's like a lift button: pressing it five times still calls the lift once.

### Q20. How did you make sure a user could not join another user's chat?
**Answer:** I do the authorization on the server. I check the authenticated user and verify that they are actually a participant in that conversation, or have the required permission. I never rely only on a chat id sent by Angular.

Anything the browser sends can be edited by the user. If the server trusted the chat id alone, someone could change it to another number and read other people's conversations. So the server takes the user from the token and checks in the database that this user really belongs to that chat before sending or joining anything. It's like a guest list at a private event: knowing the room number isn't enough, your name has to be on the list.

### Q21. What issue can happen if the same user opens the app on two devices?
**Answer:** The same user can then have more than one SignalR connection. I track each connection id separately and remove that connection when it disconnects. For presence, I would treat the user as online while at least one active connection exists, instead of marking them offline as soon as one device disconnects.

Each browser tab or device gets its own connection id. If I only kept one id per user, closing the laptop would mark the user offline even though they're still active on their phone. Counting active connections fixes that: the user is shown offline only when the last one closes.

### Q22. What was the most challenging part of the communication system?
**Answer:** Sending a message was the easy part. The hard part was making the whole flow reliable: authentication, the right participants, saving to the database, reconnects, read state, CORS and the production proxy. My approach was to keep the database as the source of truth, REST for history, and SignalR for live events.

A demo only needs one message to appear on another screen. A real product has to handle people losing connection, logging in on two devices, retrying, and running behind a proxy, all without losing or leaking a message. Splitting the jobs clearly (database stores, REST loads, SignalR announces) meant that when one part had a problem, the others still kept the chat correct.

### Q23. SignalR works locally but not in production. How would you debug it?
**Answer:** I would check it layer by layer: browser network logs, hub URL, JWT, CORS, Nginx WebSocket configuration, SSL, server logs, and finally whether the client is actually receiving the event. The first thing I want to know is whether the connection itself is failing, or whether it connects fine but the event is not being delivered.

Those two cases need different fixes. If the connection fails, the problem is usually in the path to the server: wrong URL, an expired or missing token, CORS blocking the origin, or the proxy not passing the WebSocket upgrade. The browser's network tab shows this quickly. If the connection works but no event shows up, the problem is more likely in the code: sending to the wrong group, a mismatched event name between server and client, or (with more than one server) a missing backplane. Going step by step, like checking a broken lamp from the plug to the bulb, stops me guessing.
