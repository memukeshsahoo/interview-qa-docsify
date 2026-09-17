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
