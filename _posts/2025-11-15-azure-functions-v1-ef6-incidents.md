When Legacy .NET Meets Azure Functions: Two Production Incidents and What They Taught Us

We recently lifted an old .NET Framework + Entity Framework 6 codebase into an Azure Functions v1 app that listens to Service Bus messages and writes to SQL Server.

On paper, the flow is straightforward:
	•	New booking → Service Bus topic
	•	Azure Function picks up the message
	•	We map it, update user/profile/payment tables via EF, and emit telemetry to Application Insights

In reality, we hit two very different issues that kept showing up in production:
	1.	Host lock lease failures on the Function host
	2.	Entity Framework “A second operation started on this context…” errors under load

To make it more confusing, these two errors often appeared a day or two after a restart, so for a while we were convinced the host lock problem was causing the EF exceptions.

Below is what actually happened.

⸻

Architecture at a Glance

flowchart LR
    A[External System] -->|Booking Message| B[Azure Service Bus Topic]
    B --> C[Azure Function App (v1, .NET Framework)]
    C -->|Host locks, logs| D[Storage Account (AzureWebJobsStorage)]
    C -->|EF6 / ADO.NET| E[SQL Database]
    C -->|Telemetry| F[Application Insights]


⸻

Issue 1 – “Failed to acquire host lock lease (403 Forbidden)”

Symptoms

Every few seconds, the host spammed:

Host 'xxxx' failed to acquire host lock lease:
Microsoft.WindowsAzure.Storage: The remote server returned an error: (403) Forbidden.
Forbidden.
Forbidden.

Restarting the Function App “fixed” it temporarily.

A day later, it crept back.

Our storage account security was tightened at the same time:
	•	Public network access disabled
	•	Only private endpoints and VNets were allowed

Everything seemed correct: keys, connection strings, RBAC, managed identity. But the host couldn’t renew the lock blob in azure-webjobs-hosts/locks.

⸻

Why the host needs a blob lease

Azure Functions uses a host lock (blob lease) so that:
	•	Only one instance is “active” when needed
	•	Timers/triggers do not overlap incorrectly
	•	Internal housekeeping routines coordinate safely

If the runtime can’t access this blob → host lock failure → runtime goes unhealthy.

⸻

Why we initially assumed this caused the EF errors
	•	The issue always started after 24–48 hours.
	•	Restarting the Function App fixed it every time.
	•	We were restarting the function during off-hours just to keep the system stable.

This made us believe the host issue was triggering the other failures.

But the truth was more nuanced.

⸻

Real Root Cause: Storage Network Rules Blocking Internal Azure IPs

After detailed discussions with Microsoft, the explanation was:

Azure Functions uses internal Azure IP ranges (not part of your VNet) to access the storage account.
If you set Public network access = Disabled, those internal IPs get blocked, even if the Function App is in the same VNet.

We asked:

“Are these Microsoft public IPs or private?”

The answer:

They are internal Azure IPs, neither in your VNet nor truly “public”.
When you choose ‘Disable public network access’, the storage firewall blocks them.
When you use Selected networks + your VNet, those internal calls continue to work.

The official docs do not explicitly state this, but the behavior is consistent with:
	•	https://learn.microsoft.com/en-us/azure/storage/common/storage-network-security-virtual-networks?tabs=azure-portal
	•	https://learn.microsoft.com/en-us/azure/storage/common/storage-network-security-set-default-access?tabs=azure-portal

⸻

Fix: Switch From “Disabled” to “Selected Networks”

To restore connectivity:
	1.	Open Storage Account → Networking
	2.	Set:
Public Network Access = Enabled from selected networks and IP addresses
	3.	Add the same VNet/Subnet as the Function App
	4.	Keep private endpoints intact

This allowed both:
	•	Our Function App traffic
	•	Azure internal system traffic

Once done:
	•	Host lock lease errors vanished
	•	Function App stayed healthy permanently
	•	We stopped scheduling midnight restarts to “stabilize” it

⸻

How it works (visual)

sequenceDiagram
    participant Func as Azure Function Host
    participant SA as Storage Account
    participant Infra as Azure Internal Infra

    Note over Func,SA: With Public Access Disabled
    Func->>SA: Acquire host lock lease (blob)
    SA-->>Func: 403 Forbidden (internal IP blocked)

    Note over Func,SA: After Selected Networks + VNet
    Func->>SA: Acquire host lock lease
    SA-->>Func: 200 OK (lease acquired)


⸻

Issue 2 – EF6 + Azure Functions = “A second operation started on this context…”

Even after fixing host issues, another completely different error remained:

A second operation started on this context before a previous async operation completed.
Any instance members are not guaranteed to be thread-safe.

And sometimes deeper:

INSERT statement conflicted with the FOREIGN KEY constraint
FK_ICubePayment_UserProfile

This was happening inside our business logic, not at the platform level.

And again, restarting the function app made it go away… temporarily.

Which further fooled us into thinking the two issues were related.

⸻

Root Cause: A Static Shared EF6 DbContext 🤦‍♂️

Our old Framework code used a static dependency container, which created one DbContext instance at cold start:

public static readonly BookingServicebusFunc BookingConsumer;

static DependencyContainer()
{
    var dbContext = new TVSModel(connectionString);
    BookingConsumer = new BookingServicebusFunc(dbContext, ...);
}

And our Azure Function used it like:

var consumer = DependencyContainer.BookingConsumer;
await consumer.ProcessOfflineBookingData(payload);

So:
	•	One global shared DbContext
	•	All Service Bus messages, across all threads, using that same context
	•	EF6 is NOT thread-safe, even with async/await everywhere

Under load:
	•	Message #1 saving a UserProfile
	•	Message #2 saving a Payment
	•	Both using the same DbContext
	•	EF internal state gets corrupted

Leading to:
	•	“second operation started on this context”
	•	Foreign key conflicts
	•	Partial writes
	•	Random failures after some uptime

Restarting the app reset the static context → making the problem “go away” until next time.

⸻

We Tried Everything Before Finding the Real Fix

Because the error text talks about async operations, we naturally went in that direction:
	•	Converted everything to async / await
	•	Switched to SaveChangesAsync() everywhere
	•	Added checks and tried to make sure every EF call was awaited properly
	•	Added more detailed status logging like status:Mapped Payload to see where it blew up

But even with everything awaited, the error kept coming back under load.

That’s when we had to accept the more fundamental truth:

Entity Framework 6 DbContext is not thread-safe.
Even if every method is async/await, you cannot safely share one context instance across concurrent operations.

In an Azure Functions app, multiple messages are processed in parallel. By sharing a static context, we had:
	•	Message 1 querying and saving on the same TVSModel
	•	Message 2 doing the same at the same time
	•	EF’s change tracker and connection state getting corrupted

Result: “second operation started on this context” and FK conflicts.

⸻

Final Fix: One DbContext Per Message + Proper Disposal

1. DependencyContainer now CREATES a new DbContext per call

public static BookingServicebusFunc CreateBookingConsumer()
{
    var dbContext = new TVSModel(_connectionString);
    return new BookingServicebusFunc(
        dbContext,
        new BookingDetailRepository(dbContext),
        new UserProfileRepository(dbContext),
        ...
    );
}


⸻

2. BookingServicebusFunc implements IDisposable

public class BookingServicebusFunc : IDisposable
{
    private readonly TVSModel _tvsDbContext;
    private bool _disposed;

    public BookingServicebusFunc(TVSModel tvsDbContext, /* ... */)
    {
        _tvsDbContext = tvsDbContext;
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
                _tvsDbContext?.Dispose();
            }
            _disposed = true;
        }
    }

    // all your async methods using _tvsDbContext...
}


⸻

3. Azure Function uses using(...) per invocation

[FunctionName("ProcessBookingFunction")]
public static async Task Run(
    [ServiceBusTrigger(
        "%BookingServiceBusTopic%",
        "%BookingServiceBusSubscriptionName%",
        AccessRights.Listen,
        Connection = "BookingServiceBusConnectionString"
    )] string mySbMsg,
    TraceWriter log)
{
    log.Info($"📨 Received booking message: {mySbMsg}");

    using (var consumer = DependencyContainer.CreateBookingConsumer())
    {
        string status = "NotStarted";
        var payloads = consumer.DeserializeBookingPayload(mySbMsg);

        foreach (var payload in payloads)
        {
            try
            {
                var result = await consumer.ProcessOfflineBookingData(payload);
                status = result?.ToString() ?? "null";
                log.Info($"Processed booking {payload.BookingUUID} with status: {status}");
            }
            catch (Exception ex)
            {
                log.Error($"❌ Error processing booking {payload.BookingUUID} with status : {status}: {ex.Message}");
            }
        }
    }   // DbContext disposed here
}

This ensures:
	•	No shared DbContext
	•	Clean unit-of-work per trigger execution
	•	No concurrency or tracking conflicts

After this change:
	•	The “second operation started on this context” error stopped completely
	•	The FK conflicts vanished
	•	We could remove the off-hours restart hack entirely

⸻

Putting Both Issues Side by Side

flowchart TB
    subgraph Issue1[Issue #1: Host Lock / Storage Networking]
      I1A[Public network access disabled] --> I1B[Internal Azure IPs blocked]
      I1B --> I1C[403 on host lock lease]
      I1C --> I1D[Host becomes unhealthy after a day]
      I1D --> I1E[Manual restart temporarily hides issue]
    end

    subgraph Issue2[Issue #2: EF Concurrency]
      I2A[Static shared DbContext] --> I2B[Parallel message processing]
      I2B --> I2C[Second operation on this context]
      I2C --> I2D[FK conflicts & partial writes]
      I2D --> I2E[Restart resets DbContext and hides issue]
    end

Both issues produced the same symptom:

➡ “Everything works fine after a restart, then breaks after some hours/days.”

But the root causes were completely different.

⸻

Lessons Learned

1. Azure Storage networking can impact the Function Runtime itself

Not just your code.

Prefer:

✔ Selected networks + VNet
Instead of
❌ Hard “Disable Public Access”

Docs:
https://learn.microsoft.com/en-us/azure/storage/common/storage-network-security-virtual-networks
https://learn.microsoft.com/en-us/azure/storage/common/storage-network-security-set-default-access

⸻

2. EF6 DbContext is a unit-of-work — not a singleton

In Azure Functions:
	•	Create 1 DbContext per invocation
	•	Dispose it after use
	•	Never share it across messages

⸻

3. Beware of “restart-driven debugging”

If an app works after restart and fails after N hours/days:
	•	Something has a wrong lifetime
	•	Something is cached incorrectly
	•	Or the platform configuration is blocking long-running behaviour

A restart is a symptom, not a solution.

⸻

If you want, I can also prepare:

✔ A shortened LinkedIn version
✔ A two-part series split
✔ A GitHub README variant
✔ A version with callouts & warnings for internal Confluence

Just tell me!
