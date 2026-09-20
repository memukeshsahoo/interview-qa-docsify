# SignalR (talk from a previous project)

Use **“in my last project”** language. Keep it honest and simple.

---

## Q1. What is SignalR and where did you use it?

**Answer:**

SignalR is real-time messaging on ASP.NET Core. The server **pushes** to browsers over WebSockets (with fallbacks). I used it when polling every few seconds was too slow or too chatty.

**In my previous project:** we used SignalR so agents saw ticket changes **live** — someone took a ticket, a new incident appeared, a chat/comment landed — without refreshing the Angular board.

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

Two agents opened the same new incident. Both clicked **Take**. Without SignalR, both thought they owned it. With SignalR + a row version on the API, the second agent’s board moved the card to “Taken by Priya” in a second.

**Cross-question:** Why not `setInterval` HTTP GET?

**Cross-answer:**

Polling is simpler. For 50 agents every 2 seconds you hammer the API. SignalR sends **only when something changes**. For a KPI that can be 30 seconds stale, polling is fine.

---

## Q2. How do you authenticate SignalR?

**Answer:**

Same user as the REST API. JWT on the connection. `[Authorize]` on the hub. I still **never** trust the client’s tenant id. I take tenant from the **token** and put the connection in that group.

**Example:**

```csharp
[Authorize]
public class IncidentHub : Hub { }
```

**Real-world example:**

A tester passed another tenant in a query string. The hub ignored it and used the claim. They did not receive the other hospital’s events.

**Cross-question:** Token in `?access_token=`?

**Cross-answer:**

Common for WebSockets. Treat it as sensitive: short access JWT, no logging of full URLs, HTTPS.

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

A comment in tenant A popped up on tenant B’s board because the code used `Clients.All`. We switched to `Group(tenant)` and added a test.

**Cross-question:** User in two tenants?

**Cross-answer:**

One connection per token/tenant. Switching tenant means a new token and reconnect.

---

## Q4. How does SignalR scale to two servers?

**Answer:**

Each server has its own WebSocket list. User on server 1 will not get a push from server 2 unless you add a **backplane** (Redis is the usual one) or a shared message bus. Sticky sessions can hide the bug in a small setup and fail later.

**Example:**

```csharp
builder.Services.AddSignalR().AddStackExchangeRedis(redisConnection);
```

**Real-world example:**

We added a second API node behind Nginx. Live “ticket taken” worked only if both agents landed on the same node. Redis backplane made it work across nodes.

**Cross-question:** Hangfire vs SignalR?

**Cross-answer:**

Hangfire: background jobs (email, reports). SignalR: live UI. After save we wrote an **outbox** (or published an event); the worker sent email; the request also notified the hub. Two different problems.

---

## Q5. How did Angular consume SignalR in your project?

**Answer:**

A **root service** owned **one** connection. Screens subscribed to events (or a signal). We did not `start()` in every component. On logout we `stop()`. `withAutomaticReconnect()` for blips.

Unrelated components (board + toast) both listened to the same service — same pattern as a shared state service.

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

The board and a small header “live” dot both used `IncidentHubService`. We did not open two sockets.

**Cross-question:** Connection in `ngOnInit` of the board only?

**Cross-answer:**

Then leaving the board drops live toasts on other screens. Root service is better.

---

## Q6. What can go wrong with SignalR in production?

**Answer:**

- Proxy/Nginx must allow WebSockets (`Upgrade` headers).  
- Load balancer without backplane.  
- Broadcasting too much (full entity graphs). Send small DTOs (id, status, by whom).  
- Not handling reconnect (UI stuck “connecting”).  
- Using SignalR for large file transfer — don’t; use REST download.

**Example:**

```csharp
await Clients.Group(tenant).SendAsync("incidentUpdated", new { dto.Id, dto.Status, dto.TakenBy });
```

**Real-world example:**

We pushed the full incident with comments. Messages were huge and the board janked. We switched to a small DTO and the board already had the rest or re-fetched one row.

**Cross-question:** SignalR for CRUD instead of REST?

**Cross-answer:**

No. Create/update still go through REST (validation, status codes, ProblemDetails). SignalR **notifies**. REST **changes** data.

---

## Q7. Mini story you can memorize (30 seconds)

**Answer:**

> In my previous project we used SignalR on the incident board. When an agent took a ticket, the API saved with a concurrency token, then notified the tenant group. Angular had one hub service. Other agents saw the card move. We grouped by tenant so we never broadcast to everyone. When we scaled to two servers we added a Redis backplane. Auth was the same JWT as the REST API.

**Cross-question:** What did you learn?

**Cross-answer:**

Live UI is easy to demo and easy to leak across tenants if you use `Clients.All`. Groups + small DTOs + REST for writes is the safe shape.

---

## Q8. How does SignalR work in NriCare (current project)?

**Answer:**

Hub is `ChatHub` at `/chat/hub`. JWT can arrive as `?access_token=` because browsers cannot set the header on WebSockets. On connect I put the connection in a group named with `user_id`. `SendMessage` saves the row, then `ReceiveMessage` goes to **participant groups**, not `Clients.All`. Chat can also carry a quotation id. Push for offline users is a new DI scope in `Task.Run`.

**Example:**

```csharp
app.MapHub<ChatHub>("/chat/hub").RequireCors("SignalRCors");

// JWT events
if (!string.IsNullOrEmpty(accessToken) && path.StartsWithSegments("/chat/hub"))
    context.Token = accessToken;
```

**Real-world example:**

NR user and service provider chat about a booking. Quotation messages use `ContentType.Quotation`. Presence (`UserOnline` / `UserOffline`) currently broadcasts `Clients.All`. I would restrict that. Connection map is in-memory, so it is per server.

**Cross-question:** Is `[Authorize]` on the hub?

**Cross-answer:**

The hub is not marked `[Authorize]`. It throws `HubException("Unauthorized")` if `user_id` is missing. I would add `[Authorize]` so unauthenticated sockets never enter `OnConnectedAsync`.


---
## Additional Questions
### Q9. What is the difference between SignalR and REST?
**Answer:** REST is request-response and is good for CRUD. SignalR is for server-to-client real-time events. In a normal app I use REST to change data and SignalR to notify connected clients.

### Q10. How do you handle reconnects?
**Answer:** I use automatic reconnect, show connection state in the UI, and make sure the client can resync missed data after reconnect. A socket is not a reliable database.

### Q11. Why should SignalR messages be small?
**Answer:** Large messages increase network and browser work. I usually send an id and changed fields, then let the client update or refetch the required record.

### Q12. What was the biggest difficulty you faced while implementing SignalR?
**Answer:** The main difficulty was making the real-time connection reliable outside local development. The hub worked locally, but we faced issues with WebSocket connection, CORS and Nginx proxy configuration when the client was deployed. I checked the network logs, verified the hub URL and JWT, configured the proxy for WebSocket upgrade, and fixed CORS for the actual client origin. We kept REST APIs for chat history and SignalR for real-time events, so reconnecting was easier.

### Q13. Why did you use REST APIs and SignalR together for chat?
**Answer:** I used REST for chat list and message history because it is easier to query, paginate and load again. SignalR is better for new messages and read receipts because they need real-time delivery. If the socket disconnects, the app can reconnect and fetch the latest history through REST.

### Q14. What problem did you face with SignalR behind Nginx?
**Answer:** Normal HTTP proxying was working, but the WebSocket connection was not reliable. I checked the Nginx location for the hub and made sure the WebSocket Upgrade and Connection headers were forwarded correctly. I also checked the hub path, SSL and CORS configuration.

### Q15. How did you authenticate SignalR with JWT?
**Answer:** The SignalR connection uses the same JWT authentication as the API. For the WebSocket connection the token can come through `access_token`. On the server I validate the token and use the authenticated user information. I do not trust a user id sent by the client for authorization.

### Q16. What happens if a user is offline when a message is sent?
**Answer:** I do not depend on SignalR to store the message. I save the message in the database first and then send the real-time event. If the user is offline, the message remains in the database. When the user comes back, the app loads the missed messages through the REST API and can also use push notification if required.

### Q17. How did you handle reconnects?
**Answer:** I used automatic reconnect on the Angular SignalR connection. But reconnect alone is not enough because some messages may be missed while the socket is down. After reconnect, I refresh the relevant chat data from the API so the UI is synchronized again.

### Q18. How did you handle read and unread messages?
**Answer:** The read state is stored in the database because it must survive a reconnect. SignalR is used to notify the other participant that the message or chat was read. So the database is the source of truth and SignalR updates the UI immediately.

### Q19. How did you prevent duplicate messages?
**Answer:** I make the database operation idempotent. A message should have a unique business or client request id so a retry does not create another row. I save the message first and then broadcast the saved message, instead of treating the SignalR event itself as the database operation.

### Q20. How did you make sure a user could not join another user's chat?
**Answer:** I do the authorization on the server. I check the authenticated user and verify that the user is actually a participant in that conversation or has the required permission. I never rely only on a chat id sent by Angular.

### Q21. What issue can happen if the same user opens the app on two devices?
**Answer:** The same user can have multiple SignalR connections. I treat the connection id separately and remove that connection when it disconnects. For presence, I would consider the user online while at least one active connection exists, instead of marking them offline when only one device disconnects.

### Q22. What was the most challenging part of the communication system?
**Answer:** The difficult part was not sending a message. It was making the whole flow reliable: authentication, correct participants, database persistence, reconnects, read state, CORS and production proxying. My approach was to keep the database as the source of truth, REST for history and SignalR for live events.

### Q23. SignalR works locally but not in production. How would you debug it?
**Answer:** I would check it layer by layer: browser network logs, hub URL, JWT, CORS, Nginx WebSocket configuration, SSL, server logs and finally whether the client is actually receiving the event. I would first confirm whether the connection itself is failing or whether the connection works but the event is not being delivered.
