# concurrent-job-queue-api

An in-memory job queue behind an ASP.NET Core (.NET 10) API: POST an integer array, get a job ID, poll for the sorted result.

The sorting is a stand-in for work; the repo is about the plumbing around it. `QueueService` is a hosted service that owns a `System.Threading.Channels` channel and a fixed pool of consumers, one per CPU core by default. Every job stays in a `ConcurrentDictionary`, so its status and result can be polled by ID. The API has JWT login with role-based endpoints and a paginated job list. A console client generates load: 1000 concurrent workers, 100 requests each by default.

## Architecture

```
 Client (load generator)          PublicApi (ASP.NET Core, .NET 10)
 1000 workers x 100 requests
        │                        ┌─────────────────────────────────────────┐
        │  POST /Queue/enqueue   │ QueueController                         │
        ├───────────────────────▶│     │ Enqueue(int[]) → job id           │
        │                        │     ▼                                   │
        │  GET /Queue/get-job    │ QueueService (IHostedService)           │
        ├───────────────────────▶│   Channel<Job> ──▶ consumer 1..N        │
        │                        │   ConcurrentDictionary<Guid, Job>       │
        │  GET /Queue/get-jobs   │     (status, output, duration, error)   │
        ├───────────────────────▶│                                         │
        │  POST /login → JWT     │ UserController → TokenService           │
        └───────────────────────▶│                                         │
                                 └─────────────────────────────────────────┘
```

## Components

**`src/PublicApi`** — the API. `QueueController` exposes the three queue endpoints. `QueueService` starts `ConsumerCount` consumer tasks in `StartAsync`, each reading the channel with `ReadAllAsync`; `StopAsync` completes the writer and waits for the consumers. A job moves `Pending` → `Running` → `Completed` or `Failed`. `UserController` and `TokenService` issue HMAC-SHA512 JWTs, valid for 7 days, for two hard-coded users. `ExceptionMiddleware` turns unhandled exceptions into a JSON 500 and includes the stack trace only in Development.

**`src/Client`** — console load generator. `ConcurrentWorkers` tasks share one `HttpClient`; each sends `RequestsPerWorker` arrays of 1 to 9999 random integers to `/Queue/enqueue`, then the client prints how many requests were sent, could not be sent, or failed on the server.

**`tests/UnitTests`** — xUnit and Moq, 9 tests over `QueueController`, `QueueService` and `UserController`.

## Quick start

```bash
dotnet run --project src/PublicApi      # http://localhost:5189, Swagger UI at /swagger
curl -X POST http://localhost:5189/Queue/enqueue \
  -H 'Content-Type: application/json' -d '[10,0,9,1,8,2,7,3,6,4,5]'
curl "http://localhost:5189/Queue/get-job?jobId=<id returned above>"
dotnet run --project src/Client         # default load: 100 000 requests, see Caveats
dotnet test
```

## Endpoints

| Method | Path | Needs, when authentication is on | Returns |
|---|---|---|---|
| `POST` | `/login` | nothing; body `{"userName": "admin", "password": "admin"}` | `{"token": "..."}` |
| `POST` | `/Queue/enqueue` | any logged-in user; body is a JSON integer array | the job ID |
| `GET` | `/Queue/get-job?jobId=<id>` | nothing | the job with status, input, output, duration, error; 404 if unknown |
| `GET` | `/Queue/get-jobs?PageNumber=0&PageSize=500` | role `admin` | one page of jobs plus `currentPage`, `totalPages`, `pageSize`, `totalCount` |

Users: `admin` / `admin` with roles `admin` and `user`; `john` / `john` with role `user`. Send the token as `Authorization: Bearer <token>`.

## Configuration

| Variable | Purpose | Default |
|---|---|---|
| `UseAuthentication` | `true` enforces the JWT rules above. `false` lets every request through. | `false` |
| `TokenKey` | JWT signing key, at least 64 bytes (HMAC-SHA512). | set in `appsettings.Development.json`; in any other environment the app fails at startup without it |
| `ConsumerCount` | Number of consumer tasks reading the queue. | `Environment.ProcessorCount` |
| `MaxInputLength` | Largest accepted input array; longer inputs get a 400. | `1000000` |
| `ConcurrentWorkers` (client) | Parallel workers in the load client. | `1000` |
| `RequestsPerWorker` (client) | Requests each worker sends. | `100` |
| `RemoteAddress` (client) | Base address of the API. | `http://localhost:5189/` |

API values go in `src/PublicApi/appsettings.json` or environment variables; client values in `src/Client/appsettings.json`.

## Design decisions

- **A `Channel<Job>` with N consumers, not a lock around a `Queue<T>`.** Why: the channel does the multi-writer, multi-reader hand-off and the async wait when the queue is empty, and shutdown is one `TryComplete` call. Trade-off: the channel is unbounded, so the API accepts jobs faster than it can sort them and there is no back-pressure.
- **The job store is separate from the queue.** Jobs live in a `ConcurrentDictionary<Guid, Job>` from enqueue onwards, so `get-job` is a lookup and never touches the channel. `Status` is assigned after `Output` and `Duration`. Trade-off: nothing is ever evicted.
- **Authorization is switched off by a handler, not by removing attributes.** With `UseAuthentication=false` an `IAuthorizationHandler` succeeds every requirement, so `[Authorize]` stays on the controllers and the same code runs both ways. Trade-off: one config value disables all access control, and it is off by default.
- **Pagination is zero-based and capped at 500 per page.** Why: `get-jobs` would otherwise serialise every job with its input and output arrays. Trade-off: each call snapshots the whole dictionary before paging, and dictionary order is not insertion order, so pages are not stable while jobs are being added.

## Caveats

- Queue and job store are unbounded and in memory. Nothing is persisted; a restart loses every job.
- The API keeps the input and output array of every job. A full default client run is 100 000 jobs × about 5 000 integers on average × 4 bytes × 2 arrays, roughly 4 GB on the API side. Lower `ConcurrentWorkers` or `RequestsPerWorker` first.
- Authentication is off by default. Users and passwords are hard-coded and compared in plain text. This is a demo of the wiring, not an identity system.
- The load client sends no token, so it only works with `UseAuthentication=false`.
- Sorting runs synchronously inside the consumer task (`Array.Sort` on a copy of the input).
- `dotnet restore` prints warning NU1903: `Microsoft.OpenApi` 2.4.1, pulled in by Swashbuckle, has a known high-severity advisory.
