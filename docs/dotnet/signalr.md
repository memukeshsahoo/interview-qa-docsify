# SignalR (talk from a previous project)

Use **“in my last project”** language. Keep it honest and simple.

---

## Q1. What is SignalR and where did you use it?

**Answer:**

SignalR is the real-time messaging library in ASP.NET Core. Instead of the browser asking again and again, the server **pushes** updates to the browser, usually over WebSockets (with fallbacks if those are not available). I reach for it when polling every few seconds is too slow or too chatty.

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

Polling is simpler, I agree. But with 50 agents polling every 2 seconds, you hammer the API. SignalR sends **only when something changes**. That said, for a KPI that can be 30 seconds stale, polling is fine.

---

## Q2. How do you authenticate SignalR?

**Answer:**

The same way as the REST API, with the same user. The JWT goes on the connection, and I put `[Authorize]` on the hub. Even then, I **never** trust a tenant id sent by the client. I read the tenant from the **token** and put the connection in that tenant’s group.

**Example:**

```csharp
[Authorize]
public class IncidentHub : Hub { }
```

**Real-world example:**

A tester passed another tenant in the query string. The hub ignored it and used the claim from the token, so they did not get the other hospital’s events.

**Cross-question:** Token in `?access_token=`?

**Cross-answer:**

Yes, that is common for WebSockets. I just treat it as sensitive: a short-lived access JWT, no logging of full URLs, and HTTPS only.

---

## Q3. How do you send to the right users only?

**Answer:**

- `Clients.Caller` — only me  
- `Clients.Others` — everyone else on the hub (careful in multi-tenant)  
- `Clients.Group(tenantKey)` — this tenant  
- `Clients.User(userId)` — one user  

**Default for a tenant product: Groups named by tenant.** Never `Clients.All` in a multi-tenant app.

**Example:**

```csharp
await Clients.Group(tenantKey).SendAsync("commentAdded", incidentId, preview);
```

**Real-world example:**

A comment in tenant A showed up on tenant B’s board because the code used `Clients.All`. We switched to `Group(tenant)` and added a test for it.

**Cross-question:** User in two tenants?

**Cross-answer:**

We keep one connection per token/tenant. Switching tenant means getting a new token and reconnecting.

---

## Q4. How does SignalR scale to two servers?

**Answer:**

Not on its own. Each server keeps its own list of WebSocket connections. So a user connected to server 1 will not get a push sent from server 2 unless you add a **backplane** (Redis is the usual one) or a shared message bus. A backplane just passes each message to every server. Sticky sessions can hide the bug in a small setup, and then it fails later.

**Example:**

```csharp
builder.Services.AddSignalR().AddStackExchangeRedis(redisConnection);
```

**Real-world example:**

We added a second API node behind Nginx. The live “ticket taken” update worked only if both agents happened to land on the same node. Adding the Redis backplane made it work across nodes.

**Cross-question:** Hangfire vs SignalR?

**Cross-answer:**

They solve two different problems. Hangfire is for background jobs like emails and reports. SignalR is for live UI. In our flow, after the save we wrote to an **outbox** (or published an event), the worker sent the email, and the request also notified the hub.

---

## Q5. How did Angular consume SignalR in your project?

**Answer:**

We had **one** connection, owned by a **root service**. Screens subscribed to its events (or read a signal). We did not call `start()` in every component. On logout we called `stop()`, and we used `withAutomaticReconnect()` to handle short network blips.

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

Then as soon as you leave the board, the live toasts on other screens stop. A root service is better.

---

## Q6. What can go wrong with SignalR in production?

**Answer:**

- Proxy/Nginx must allow WebSockets (`Upgrade` headers).  
- A load balancer with no backplane.  
- Broadcasting too much (full entity graphs). Send small DTOs instead (id, status, by whom).  
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

No. Creates and updates still go through REST, because that is where we get validation, status codes and ProblemDetails. SignalR **notifies**. REST **changes** data.

---

## Q7. Mini story you can memorize (30 seconds)

**Answer:**

> In my previous project we used SignalR on the incident board. When an agent took a ticket, the API saved with a concurrency token, then notified the tenant group. Angular had one hub service. Other agents saw the card move. We grouped by tenant so we never broadcast to everyone. When we scaled to two servers we added a Redis backplane. Auth was the same JWT as the REST API.

**Cross-question:** What did you learn?

**Cross-answer:**

Live UI is easy to demo, and just as easy to leak across tenants if you use `Clients.All`. Groups, small DTOs and REST for writes — that is the safe setup.

---

## Q8. How does SignalR work in NriCare (current project)?

**Answer:**

The hub is `ChatHub`, mapped at `/chat/hub`. The JWT can come in as `?access_token=` because browsers cannot set the auth header on a WebSocket. On connect, I put the connection in a group named with the `user_id`. `SendMessage` saves the row first, then sends `ReceiveMessage` to the **participant groups**, not `Clients.All`. A chat message can also carry a quotation id. Push notifications for offline users run in a new DI scope inside `Task.Run`.

**Example:**

```csharp
app.MapHub<ChatHub>("/chat/hub").RequireCors("SignalRCors");

// JWT events
if (!string.IsNullOrEmpty(accessToken) && path.StartsWithSegments("/chat/hub"))
    context.Token = accessToken;
```

**Real-world example:**

An NR user and a service provider chat about a booking. Quotation messages use `ContentType.Quotation`. Presence (`UserOnline` / `UserOffline`) currently goes out to `Clients.All`, which I would restrict. The connection map is in-memory, so it only works per server.

**Cross-question:** Is `[Authorize]` on the hub?

**Cross-answer:**

No, the hub is not marked `[Authorize]`. Right now it throws `HubException("Unauthorized")` if `user_id` is missing. I would add `[Authorize]` so unauthenticated sockets never even reach `OnConnectedAsync`.


---
## Additional Questions
### Q9. What is the difference between SignalR and REST?
**Answer:** REST is request-response, and it is good for CRUD. SignalR is for real-time events from the server to the client. In a normal app I use REST to change data and SignalR to tell connected clients that something changed.

### Q10. How do you handle reconnects?
**Answer:** I turn on automatic reconnect, show the connection state in the UI, and make sure the client can resync any data it missed after reconnecting. A socket is not a reliable database.

### Q11. Why should SignalR messages be small?
**Answer:** Large messages mean more work for the network and the browser. I usually send just the id and the changed fields, and let the client update or refetch the record it needs.

### Q12. What was the biggest difficulty you faced while implementing SignalR?
**Answer:** The hardest part was making the real-time connection reliable outside local development. The hub worked fine locally, but once the client was deployed we hit issues with the WebSocket connection, CORS and the Nginx proxy configuration. I checked the network logs, verified the hub URL and the JWT, configured the proxy for the WebSocket upgrade, and fixed CORS for the real client origin. We kept REST APIs for chat history and SignalR for real-time events, which made reconnecting easier.

### Q13. Why did you use REST APIs and SignalR together for chat?
**Answer:** I used REST for chat list and message history because it is easier to query, paginate and load again. SignalR is better for new messages and read receipts because they need real-time delivery. If the socket disconnects, the app can reconnect and fetch the latest history through REST.

### Q14. What problem did you face with SignalR behind Nginx?
**Answer:** Normal HTTP proxying worked, but the WebSocket connection was not reliable. I checked the Nginx location block for the hub and made sure the WebSocket `Upgrade` and `Connection` headers were forwarded correctly. I also checked the hub path, SSL and the CORS configuration.

### Q15. How did you authenticate SignalR with JWT?
**Answer:** The SignalR connection uses the same JWT authentication as the API. For the WebSocket connection, the token can come through `access_token`. On the server I validate the token and use the authenticated user’s details from it. I do not trust a user id sent by the client for authorization.

### Q16. What happens if a user is offline when a message is sent?
**Answer:** Nothing is lost, because I do not depend on SignalR to store the message. I save it in the database first and then send the real-time event. If the user is offline, the message just stays in the database. When they come back, the app loads the missed messages through the REST API, and we can also send a push notification if needed.

### Q17. How did you handle reconnects?
**Answer:** I used automatic reconnect on the Angular SignalR connection. But reconnect alone is not enough, because some messages can be missed while the socket is down. So after a reconnect, I refresh the relevant chat data from the API to bring the UI back in sync.

### Q18. How did you handle read and unread messages?
**Answer:** The read state is stored in the database, because it has to survive a reconnect. SignalR is only used to tell the other participant that the message or chat was read. So the database is the source of truth, and SignalR updates the UI right away.

### Q19. How did you prevent duplicate messages?
**Answer:** I make the database operation idempotent, meaning running it twice has the same effect as running it once. Each message should have a unique business or client request id, so a retry does not create another row. I save the message first and then broadcast the saved message. I don't treat the SignalR event itself as the database operation.

### Q20. How did you make sure a user could not join another user's chat?
**Answer:** I do the authorization on the server. I check the authenticated user and verify that they are actually a participant in that conversation, or have the required permission. I never rely only on a chat id sent by Angular.

### Q21. What issue can happen if the same user opens the app on two devices?
**Answer:** The same user can then have more than one SignalR connection. I track each connection id separately and remove that connection when it disconnects. For presence, I would treat the user as online while at least one active connection exists, instead of marking them offline as soon as one device disconnects.

### Q22. What was the most challenging part of the communication system?
**Answer:** Sending a message was the easy part. The hard part was making the whole flow reliable: authentication, the right participants, saving to the database, reconnects, read state, CORS and the production proxy. My approach was to keep the database as the source of truth, REST for history, and SignalR for live events.

### Q23. SignalR works locally but not in production. How would you debug it?
**Answer:** I would check it layer by layer: browser network logs, hub URL, JWT, CORS, Nginx WebSocket configuration, SSL, server logs, and finally whether the client is actually receiving the event. The first thing I want to know is whether the connection itself is failing, or whether it connects fine but the event is not being delivered.
