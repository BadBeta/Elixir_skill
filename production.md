# Production Elixir Patterns

## Production Phoenix Patterns (from changelog.com)

> **Full implementations:** See `examples.md` → "Production Phoenix Patterns"

| Pattern | Purpose |
|---------|---------|
| **Schema Base Module** | Inject common query helpers (`newest_first`, `limit`, `older_than`) into all schemas via `use MyApp.Schema` |
| **Kit Modules** | Small, focused utility modules (`StringKit`, `ListKit`) instead of one large helper |
| **Response Cache Plug** | Cache entire HTTP responses for unauthenticated users by request path |
| **Policy Module** | Authorization with `defoverridable` defaults - `is_admin/1`, `is_owner/2` helpers |
| **Controller Context Injection** | Override `action/2` to inject common assigns into all controller actions |
| **HTTP Client Retry** | Wrap HTTP client with fallback SSL options on failure |
| **Oban Telemetry Reporter** | Capture job failures via `:telemetry.attach` and forward to error tracking |
| **Cache Cascade Deletion** | Delete related cache entries when primary entity changes |
| **Soft Delete Subscription** | Track lifecycle with `unsubscribed_at` timestamp instead of hard delete |

## Edge/IoT Patterns (from ExNVR)

> **Full implementations:** See `examples.md` → "Edge/IoT Patterns"

| Pattern | Purpose |
|---------|---------|
| **Tick-Based Threshold Monitor** | GenServer that triggers actions only after conditions persist for N consecutive checks - prevents thrashing on transient conditions |
| **Periodic System Status Collector** | GenServer that aggregates static info once and dynamic metrics (CPU, memory, uptime) on schedule |
| **Schedule Validation** | Validate time-based schedules with interval overlap detection using `reduce_while` |
| **Role-Based Authorization** | Pattern-matching authorization with Plug integration - admin/user/owner roles |
| **NIF Module Pattern** | Two-module structure: NIF stubs + Elixir wrapper for native code integration |
| **Embedded Schema Validation** | Polymorphic `cast_embed` validation based on parent field (e.g., device type) |

## Job Processing Patterns (from Oban)

> **Full implementations:** See `examples.md` → "Job Processing Patterns"

| Pattern | Purpose |
|---------|---------|
| **Worker Behaviour with Macro** | `__using__` macro with `defoverridable` defaults for backoff/timeout. Return types control lifecycle: `:snooze` preserves attempt, `{:error, _}` retries, `{:cancel, _}` terminates |
| **Pluggable Engine** | Behaviour with `@optional_callbacks` for swappable backends (Postgres, Inline test mode) |
| **Validation with Smart Suggestions** | Type-checking options with `String.jaro_distance` for typo suggestions |
| **Exponential Backoff with Jitter** | `base_pad + mult * 2^min(attempt, max_pow)` with `:inc/:dec/:both` jitter modes to prevent thundering herd |
| **Dispatch Cooldown** | GenServer with coalesced timers - cooldown between dispatches, jittered refresh interval |
| **Leader Election** | PostgreSQL advisory locks via `pg_try_advisory_lock` for cluster coordination |
| **Notifier (Distributed Signals)** | PubSub with gzip+base64 encoding, scoped by `:any` or specific ident |
| **Telemetry Integration** | `with_span/3` helper emitting `:start/:stop/:exception` events with duration |
| **Cron Expression** | MapSet-based field matching with shortcuts (`@hourly`, `@daily`), step and range parsing |
| **CTE Optimization Fence** | Subquery CTE forces Postgres to respect LIMIT before `FOR UPDATE SKIP LOCKED` |
| **Job Assertion Helpers** | `assert_enqueued/1`, `refute_enqueued/1` with timeout polling and JSON `@>` partial match |

## Telemetry Deep-Dive

The `:telemetry` library is the standard way to instrument Elixir applications. Phoenix, Ecto, Oban, and Broadway all emit telemetry events automatically.

**Core API:**

```elixir
# Emit an event (in your application code)
:telemetry.execute(
  [:my_app, :orders, :created],              # event name (list of atoms)
  %{count: 1, total_cents: order.total},     # measurements (numbers)
  %{order_id: order.id}                      # metadata (any data)
)

# Listen for events (in your telemetry module or application startup)
:telemetry.attach("my-handler",
  [:my_app, :orders, :created],              # event to listen for
  &MyApp.Telemetry.handle_event/4,           # handler (use & capture, not anon fn)
  nil                                        # config passed to handler
)

# Listen for multiple events with one handler
:telemetry.attach_many("my-handler",
  [[:phoenix, :endpoint, :stop], [:my_app, :repo, :query]],
  &MyApp.Telemetry.handle_event/4, nil
)
```

**`:telemetry.span/3` — instrument a block (emits start + stop/exception automatically):**

```elixir
result = :telemetry.span([:my_app, :external_api], %{url: url}, fn ->
  result = HTTPClient.get(url)
  {result, %{status: result.status}}     # {return_value, stop_metadata}
end)
# Emits: [:my_app, :external_api, :start]  with %{monotonic_time:, system_time:}
# Emits: [:my_app, :external_api, :stop]   with %{duration:} on success
# Emits: [:my_app, :external_api, :exception] with %{duration:} on failure
```

**Built-in events from Phoenix, Ecto, and Oban:**

| Library | Event | Measurements |
|---------|-------|-------------|
| Phoenix | `[:phoenix, :endpoint, :stop]` | `%{duration: native_time}` |
| Phoenix | `[:phoenix, :router_dispatch, :stop]` | `%{duration: native_time}` |
| Ecto | `[:my_app, :repo, :query]` | `%{total_time:, query_time:, queue_time:, decode_time:}` |
| Oban | `[:oban, :job, :stop]` | `%{duration:, queue_time:}` |
| Broadway | `[:broadway, :processor, :stop]` | `%{duration:}` |

**Slow query logger example:**

```elixir
:telemetry.attach("slow-queries", [:my_app, :repo, :query],
  fn _event, measurements, metadata, _config ->
    ms = System.convert_time_unit(measurements.total_time, :native, :millisecond)
    if ms > 100, do: Logger.warning("Slow query (#{ms}ms): #{metadata.query}")
  end, nil)
```

**Telemetry.Metrics — declarative metric definitions for dashboards/reporters:**

```elixir
import Telemetry.Metrics

def metrics do
  [
    summary("phoenix.endpoint.stop.duration", unit: {:native, :millisecond}),
    summary("my_app.repo.query.total_time", unit: {:native, :millisecond}),
    counter("my_app.orders.created.count"),
    distribution("my_app.external_api.request.duration",
      unit: {:native, :millisecond}, buckets: [100, 250, 500, 1000]),
    summary("vm.memory.total", unit: {:byte, :megabyte})
  ]
end
```

VM metrics (`:vm.memory`, `:vm.total_run_queue_lengths`) require the `:telemetry_poller` dependency.

## HTTP Clients

**Req** is the modern default HTTP client for Elixir, built on Finch. Batteries included: JSON decode, retries, redirects, compression, auth.

```elixir
# deps: [{:req, "~> 0.5"}]

# Simple requests
resp = Req.get!("https://api.example.com/data")
resp = Req.post!("https://api.example.com/data", json: %{name: "test"})

# Reusable client with shared config (connection pooling)
client = Req.new(
  base_url: "https://api.example.com",
  auth: {:bearer, token},
  retry: :transient,          # retry on 408/429/500/502/503/504
  max_retries: 3,
  receive_timeout: 15_000
)
{:ok, resp} = Req.get(client, url: "/users")
{:ok, resp} = Req.post(client, url: "/users", json: %{name: "new"})

# Streaming large responses
Req.get!("https://example.com/large.csv", into: File.stream!("/tmp/out.csv"))
```

| Library | Use When |
|---------|----------|
| **Req** | Default choice. High-level, auto-JSON, retries, redirects, auth, compression. |
| **Finch** | Fine-grained connection pool control, HTTP/2 multiplexing, low-level streaming. |
| **HTTPoison** | Legacy projects only. No reason to choose for new projects. |

**Testing HTTP calls with Req.Test:**

```elixir
# config/test.exs
config :my_app, weather_req_options: [plug: {Req.Test, MyApp.Weather}]

# In your module — merge test options
def get_temperature(location) do
  [base_url: "https://weather-api.com", params: [q: location]]
  |> Keyword.merge(Application.get_env(:my_app, :weather_req_options, []))
  |> Req.request()
end

# In test (async-safe)
test "returns temperature" do
  Req.Test.stub(MyApp.Weather, fn conn ->
    Req.Test.json(conn, %{"celsius" => 25.0})
  end)
  assert {:ok, %{body: %{"celsius" => 25.0}}} = MyApp.Weather.get_temperature("Oslo")
end

# Simulate network errors
test "handles timeout" do
  Req.Test.stub(MyApp.Weather, fn conn ->
    Req.Test.transport_error(conn, :timeout)
  end)
  assert {:error, %Req.TransportError{reason: :timeout}} = MyApp.Weather.get_temperature("Oslo")
end
```

**Alternative: Mox + behaviour** for HTTP testing (works with any client):

```elixir
# Define behaviour → production impl → mock in test
defmodule MyApp.HTTPClient do
  @callback get(String.t(), keyword()) :: {:ok, map()} | {:error, term()}
end

# test_helper.exs
Mox.defmock(MyApp.HTTPClientMock, for: MyApp.HTTPClient)
```
