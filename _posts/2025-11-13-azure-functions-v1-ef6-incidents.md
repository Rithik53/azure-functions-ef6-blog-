---
layout: post
title: "When Legacy .NET Meets Azure Functions: Two Production Incidents and What They Taught Us"
permalink: /azure-functions-v1-ef6-incidents/
---

We recently lifted an old .NET Framework + Entity Framework 6 codebase into an Azure Functions v1 app that listens to Service Bus messages and writes to SQL Server.

On paper, the flow is straightforward:
- New booking → Service Bus topic
- Azure Function picks up the message
- We map it, update user/profile/payment tables via EF, and emit telemetry to Application Insights

In reality, we hit two very different issues that kept showing up in production:

1. **Host lock lease failures** on the Function host
2. **Entity Framework "A second operation started on this context…" errors** under load

To make it more confusing, these two errors often showed up a day or two after a restart, so for a while we were convinced the host lock problem was causing the EF exceptions.

This post walks through the symptoms, false leads, the actual root causes, and the fixes.

---

## Architecture at a Glance
<div align="center">
  <img src="{{ site.baseurl }}/assets/images/architecture-diagram.svg" alt="Architecture Diagram" width="1200"/>
</div>

---

## Issue 1 – "Failed to acquire host lock lease (403 Forbidden)"

### Symptoms

Every now and then we'd see the function app go into a bad state. In the Log stream, the host spammed messages like:

```
[Verbose] Host 'xxxx' failed to acquire host lock lease:
Microsoft.WindowsAzure.Storage: The remote server returned an error: (403) Forbidden.
Forbidden.
Forbidden.
...
```

Restarting the Function App "fixed" it for a while. After 24–48 hours, the same pattern returned.

At the same time, we were tightening security on the storage account:
- Public network access disabled
- Only private endpoints / VNet access

Our app settings, connection strings, and keys were all correct. We'd also wired up managed identity and blob data contributor roles. Yet the host could not acquire the lease blob in `azure-webjobs-hosts/locks`.

### Why does the host need a blob lease?

Azure Functions uses a host lock (blob lease) in the storage account to coordinate:
- Which host instance is "active" for timers/triggers
- Scale controller coordination
- Some internal runtime housekeeping

If the runtime can't read/write that blob, it can't safely start processing, and you'll see the "failed to acquire host lock lease" messages.

### The trap: we thought this was causing everything

Because the error only showed up after some time and restarting fixed things, it was very tempting to say:

> "Okay, the host can't acquire the lock → that must be why EF is throwing errors too."

So we got into a habit of restarting the function during off-hours as a band-aid, just to keep things running while we debugged. That bought us time but obviously wasn't a real fix.

### The real culprit: storage network rules + internal Azure IPs

During a long chat with Microsoft support, the key explanation was:

> Azure Functions uses internal Azure IPs to access the storage account in addition to any traffic from your VNet. If you set Public network access = Disabled, some of those internal calls can be blocked by the storage firewall.

We asked the obvious follow-up:

> "Are these Microsoft public IPs? Private IPs? And if they're internal, how does adding our VNet to 'Selected networks' help?"

The answer, in short:
- These are **internal Azure IP ranges**, not your VNet's IPs.
- When you set "Public network access = Disabled", you block those internal calls as well.
- When you switch to "Selected networks" and add your VNet, Azure's internal plumbing is still allowed to talk to the storage account while you keep overall access locked down.

Even though not every doc spells it out explicitly, the behaviour is consistent with the guidance in:
- [Create a virtual network rule for Azure Storage](https://docs.microsoft.com/azure/storage/common/storage-network-security)
- [Set the default public network access rule](https://docs.microsoft.com/azure/storage/common/storage-network-security)

In our environment:
- Storage account: Public network access disabled, locked to a private endpoint
- Function App: integrated with a VNet
- **Result**: some of the internal runtime calls from the Functions infrastructure to the storage account were returning 403 Forbidden, breaking the host lock lease.

### The fix

We changed the storage network configuration to **Selected networks** and allowed the Function App's VNet:

1. Storage Account → Networking
2. Set **Public network access** to `Enabled from selected virtual networks and IP addresses`
3. Add the same VNet/subnet used by the Function App
4. Keep private endpoints as required

After this:
- The host lock lease warnings disappeared
- The function app no longer went into that weird "unhealthy after a day" state
- We stopped scheduling manual restarts just to keep the host alive

### Diagram: what actually happens

<div align="center">
  <img src="{{ site.baseurl }}/assets/images/whatactuallyhappens.svg" alt="What Actually Happens Diagram" width="1200"/>
</div>

---

## Issue 2 – EF6 + Azure Functions = "A second operation started on this context…"

Once the host was behaving, we still saw another problem inside our code, and for a while we blamed the host lock for this too.

### The EF error

For some bookings the logs looked like this:

```
status:Mapped Payload
Error: A second operation started on this context before a previous asynchronous operation completed.
Use 'await' to ensure that any asynchronous operations have completed before calling another method on this context.
Any instance members are not guaranteed to be thread safe.
```

Digging through the inner exceptions:

```
InnerException Level 2:
The INSERT statement conflicted with the FOREIGN KEY constraint 'FK_Payment_UserProfile'.
The conflict occurred in database '...', table 'dbo.UserProfiles', column 'UserId'.
```

So EF was complaining about:
- Concurrent operations on the same DbContext, and
- Foreign key conflicts, where a Payment record was pointing to a UserProfile that wasn't there (or not yet committed)

Again, restarting the function app would make it disappear for a while, which initially reinforced our "host issue causes everything" narrative. But this was a separate bug.

### The original design: one static DbContext for everything

Our legacy code had a static dependency container:

```csharp
public static class DependencyContainer
{
    public static readonly MessageProcessor Processor;

    static DependencyContainer()
    {
        var connectionString = Environment.GetEnvironmentVariable("EntityFrameworkConnectionString");
        var dbContext = new ApplicationDbContext(connectionString);

        var userRepo = new UserRepository(dbContext);
        var orderRepo = new OrderRepository(dbContext);
        // ... other repositories

        Processor = new MessageProcessor(
            dbContext,
            userRepo,
            orderRepo
            // ...
        );
    }
}
```

The function used this like:

```csharp
[FunctionName("ProcessMessageFunction")]
public static async Task Run(
    [ServiceBusTrigger(...)] string message,
    TraceWriter log)
{
    var processor = DependencyContainer.Processor;
    var payloads = processor.DeserializeMessage(message);

    foreach (var payload in payloads)
    {
        await processor.ProcessData(payload);
    }
}
```

So:
- One static `ApplicationDbContext` created at startup
- All Service Bus messages, across all function instances, shared that same context

### What we tried first (that didn't work)

Because the error text talks about async operations, we naturally went in that direction:
- Converted repository methods to async/await
- Switched to `SaveChangesAsync()` everywhere
- Added checks and tried to make sure every EF call was awaited properly
- Added more detailed status logging to see where it blew up

But even with everything awaited, the error kept coming back under load.

That's when we had to accept the more fundamental truth:

> **Entity Framework 6 DbContext is not thread-safe.**  
> Even if every method is async/await, you cannot safely share one context instance across concurrent operations.

In an Azure Functions app, multiple messages are processed in parallel. By sharing a static context, we had:
- Message 1 querying and saving on the same DbContext
- Message 2 doing the same at the same time
- EF's change tracker and connection state getting corrupted

**Result**: "second operation started on this context" and FK conflicts.

### Concurrency diagram

<div align="center">
  <img src="{{ site.baseurl }}/assets/images/concurrency.svg" alt="Concurrency Diagram" width="1200"/>
</div>

Restarting the function app happened to "reset" that shared context, so for a short time everything looked healthy again. That's why this bug and the host lock bug felt connected, even though they weren't.

### The fix – one DbContext per message, properly disposed

Once we stopped fighting EF and accepted that the context lifetime was wrong, the solution became obvious.

#### 1. The container now creates a fresh DbContext per call

```csharp
public static class DependencyContainer
{
    private static readonly string _connectionString;

    static DependencyContainer()
    {
        System.Data.Entity.SqlServer.SqlProviderServices.Instance.GetType();
        _connectionString = Environment.GetEnvironmentVariable("EntityFrameworkConnectionString");
    }

    public static MessageProcessor CreateProcessor()
    {
        var dbContext = new ApplicationDbContext(_connectionString);

        var userRepo = new UserRepository(dbContext);
        var orderRepo = new OrderRepository(dbContext);
        var paymentRepo = new PaymentRepository(dbContext);
        // ... other repositories

        return new MessageProcessor(
            dbContext,
            userRepo,
            orderRepo,
            paymentRepo
            // ...
        );
    }
}
```

#### 2. MessageProcessor implements IDisposable

```csharp
public class MessageProcessor : IDisposable
{
    private readonly ApplicationDbContext _dbContext;
    private bool _disposed;

    public MessageProcessor(ApplicationDbContext dbContext, /* ... */)
    {
        _dbContext = dbContext;
        // ...
    }

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (disposing)
            {
                _dbContext?.Dispose();
            }
            _disposed = true;
        }
    }

    // all your async methods using _dbContext...
}
```

#### 3. Function creates and disposes the processor per trigger execution

```csharp
[FunctionName("ProcessMessageFunction")]
public static async Task Run(
    [ServiceBusTrigger(
        "%ServiceBusTopic%",
        "%ServiceBusSubscription%",
        AccessRights.Listen,
        Connection = "ServiceBusConnectionString"
    )] string message,
    TraceWriter log)
{
    log.Info($"📨 Received message: {message}");

    using (var processor = DependencyContainer.CreateProcessor())
    {
        string status = "NotStarted";
        var payloads = processor.DeserializeMessage(message);

        foreach (var payload in payloads)
        {
            try
            {
                var result = await processor.ProcessData(payload);
                status = result?.ToString() ?? "null";
                log.Info($"Processed {payload.Id} with status: {status}");
            }
            catch (Exception ex)
            {
                log.Error($"❌ Error processing {payload.Id} with status: {status}: {ex.Message}");
            }
        }
    }   // DbContext disposed here
}
```

#### 4. Some small EF clean-ups

While we were in there, we also:
- Added `AsNoTracking()` to read-only queries
- Removed a few unnecessary `Reload()` calls
- Improved error logging (printing inner exceptions and status at each step)

After these changes:
- The "second operation started on this context" error stopped completely
- The FK conflicts vanished
- We could remove the off-hours restart hack entirely

---

## Putting both issues side by side

<div align="center">
  <img src="{{ site.baseurl }}/assets/images/bothissue.svg" alt="Both Issues Side by Side" width="1200"/>
</div>

You can see why it was easy to tie them together: in both cases "wait a day, then restart" looked like a fix. Under the hood though, they were two completely different problems.

---

## Lessons we're taking forward

### 1. Storage networking matters for the runtime itself, not just your code

- When you lock down a storage account used by Functions, read the fine print on network rules.
- Prefer **Selected networks + VNet rules** over a blanket "Disable Public Access" if the storage is backing Azure Functions.
- Keep the docs handy:
  - [Virtual network rules for Azure Storage](https://docs.microsoft.com/azure/storage/common/storage-network-security)
  - [Default public network access rules](https://docs.microsoft.com/azure/storage/common/storage-network-security)

### 2. DbContext is a unit of work, not a singleton

- EF6 DbContext is not thread-safe, no matter how async/await you make your code.
- In Azure Functions, treat each trigger execution like a web request: create a fresh context and dispose it.

### 3. Beware of "restart-driven debugging"

- If something always breaks after N hours/days and restarting fixes it briefly, that usually means you have:
  - a **lifetime leak** (shared state, static context, stale connections), or
  - a **platform-level configuration** like the storage networking issue.
- Use restarts to buy time, but don't let them become the "solution".
