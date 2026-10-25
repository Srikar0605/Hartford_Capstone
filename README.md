I have the full backend. I'll review it for performance, latency and anti-patterns without changing any code.

The `Shared.*` projects (logging, exceptions, common) are referenced but aren't in this copy. Checking whether the original upload has them:

You never sent the Shared projects, so I'll review the five services and the gateway and treat Shared as unseen. Next, the startup and database setup for each service:

Checking indexes on the columns the hot queries filter by:

I reviewed all five services and the gateway: about 30k lines, excluding tests, the `Shared.*` projects you never sent, and the migrations. This is a code review, not a measurement. I can't run your SQL Server or Gemini setup here, so latencies below are estimates, and nothing has been changed.

Overall the data layer is solid. Most of the real risk is in how the services and the AI calls hang together.

## High impact

**1. AI calls block requests with no timeout.** Three requests wait for Gemini before they respond, and none of the Gemini clients sets a timeout:
- Claim submit waits on Claims Triage (`AiTriageClient`).
- Policy apply waits on the Compliance Agent when an application goes to manual review.
- Every chat message waits on ShieldBot.

The .NET default is 100 seconds, so a slow model can hang one of these requests for up to 100s. Chat retries a fallback model after a failure, so it can take about 200s before dropping to the local answer. In Vertex mode, each call also runs `GoogleCredential.GetApplicationDefaultAsync()` again, which fetches a new OAuth token every time instead of reusing a cached one.

- Fix: an 8–10s timeout per call, then the rule-based fallback you already have; one cached credential. Optionally, save the claim first and triage it right after, so the customer isn't waiting on Gemini.

**2. Writes span two services with no recovery path.**
- **Premium payment:** Payment saves the successful transaction *and* its "200 OK" idempotency record, then calls Policy to confirm the payment. If that call fails, the money is recorded but the policy stays unpaid. A retry from the client just replays the saved 200, so the policy is never confirmed.
- **Settlement:** this is a circular synchronous chain: Claims → Payment → Claims (`ValidateSettlement`) → back to Payment → back to Claims. If Claims' final save fails, for example on a row-version conflict, the payment exists with no claim pointing to it.
- Fix: an outbox table or a reconciliation job that re-sends unconfirmed payments. A simple background retry would do here.

**3. The retry setup only half works.** Every service client sets `HttpClient.Timeout = 5s` and also `AddStandardResilienceHandler()`.
- `HttpClient.Timeout` covers the whole call, retries included, so the 5s cap means retries rarely get a chance to run.
- When a retry does happen, it also applies to POSTs that aren't safe to repeat. If the first `ConfirmPayment` attempt actually succeeded but timed out, the retry hits the row-version check and fails with a 409 conflict.
- Nested calls share the same 5s budget. Claims waits 5s on Payment, while Payment itself waits up to 5s on Claims.

**4. The admin dashboard loads whole tables, and most of its numbers are made up** (`SystemController.GetDashboardMetrics`).
- It loads *every* policy and property into memory just to compute a sum and a group-by. The database should do that.
- Separately, most of the numbers are generated with `new Random(42)`, not measured:
  - p50/p90/p99 latency, CPU, memory and error rates;
  - churn and claim-status counts;
  - "claims paid", which is simply 68% of premiums written.

  Only premiums written and risk exposure are real. A reviewer who looks at the code will notice, so either label these as simulated or compute them for real.

## Medium

5. **Login and token refresh get slower over time.** Old refresh tokens are revoked but never deleted. Login and refresh load *all* of a user's tokens plus roles → permissions in one query without `AsSplitQuery()`, so the rows returned are tokens × permissions. That grows with every login (`UserRepository.cs` lines 60–121).
6. **Independent calls run one after another.**
   - Claim submit makes four service calls in sequence: customer validate, policy validate, coverage validate and coverage balance.
   - Policy apply makes two to Customer, eligibility then the full property list, only to pick one property.
   - Fix: run them in parallel (`Task.WhenAll`) or combine them into one endpoint.
7. **Detail pages and recommendations are chatty.**
   - Policy details take 8 sequential queries (`PolicyProjection`).
   - Claim details take about 7, including the ownership check run twice, because `GetAsync` calls `GetTimelineAsync`, which checks again.
   - Recommendations load the plans, then query each plan again individually inside the loop.
8. **Chat reloads the catalogue from the database on every message**, even though `PolicyCatalogCache` exists for exactly this.
9. **MFA challenges and trusted devices live only in memory** (singleton `ConcurrentDictionary`). They're lost on restart and break if you run more than one instance. Trusted-device entries are never cleaned up unless reused.
10. **The `IdempotencyRecords` tables grow forever.** In Claims, Payment and Policy there's an index on `ExpiresAt` but no job that deletes expired rows.

## Low

- **Missing index:** the double-insurance check filters `Properties.CustomerPropertyOwnershipId`, which has no index, so it scans the table.
- **Customer search:** `Contains` becomes `LIKE '%x%'`, which can't use an index. That's fine at your current size.
- **Address autocomplete cache:** it has no size limit and caches one entry per typed query for 6 hours.
- **Gateway:** Ocelot has no QoS or timeout settings (default 90s), so it holds connections open while Gemini hangs.
- **Fat controller:** `SystemController` uses the DbContext directly and builds the AI prompt.
- **Mixed patterns:** Auth and Customer use MediatR, while Policy, Claims and Payment use service classes. That's inconsistent, though it doesn't affect speed.
- **Coordinate backfill:** the CLI-only backfill uses `new HttpClient()` and sets private properties through reflection.

## Already done well

Reads use `AsNoTracking`, the indexes on hot filters are good, and page sizes are capped at 100. You also have row-version concurrency, idempotency keys, file uploads that stream instead of buffering, a plan catalogue cache, `IHttpClientFactory` clients and a chat rate limiter. That's a solid base.

**Rough latency picture:** without a Gemini key, every request is probably tens of milliseconds locally. With a key, claim submit, manual-review applications and chat are dominated by Gemini (typically a few seconds) and have no upper bound today. #1 is the fix that matters most.

You asked me to discuss before changing anything, so tell me which of these you want fixed. My suggestion is #1, #3 and #5 first: each is small, contained and backend-only. I can also turn this into a shareable write-up if you need it for a review.
