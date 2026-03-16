# Event Sourcing & CQRS Quick Reference

Quick-lookup tables for Commanded library, event store configuration, router DSL, middleware, projections, process managers, serialization, and mix tasks. Patterns sourced from Commanded library test suite and documentation.

## Dependencies

```elixir
# mix.exs
defp deps do
  [
    {:commanded, "~> 1.4"},
    {:commanded_eventstore_adapter, "~> 1.4"},  # PostgreSQL event store
    {:jason, "~> 1.4"},
    # Optional
    {:commanded_ecto_projections, "~> 1.4"},     # Ecto-based projections
    {:commanded_uniqueness_middleware, "~> 1.0"}  # Unique constraint validation
  ]
end
```

## Configuration

```elixir
# config/config.exs
config :my_app, MyApp.CommandedApp,
  event_store: [
    adapter: Commanded.EventStore.Adapters.EventStore,
    event_store: MyApp.EventStore
  ],
  pubsub: :local,
  registry: :local

config :my_app, event_stores: [MyApp.EventStore]

config :my_app, MyApp.EventStore,
  serializer: Commanded.Serialization.JsonSerializer,
  username: "postgres",
  password: "postgres",
  database: "my_app_eventstore",
  hostname: "localhost",
  pool_size: 10

# config/test.exs — use Sandbox pool
config :my_app, MyApp.EventStore,
  pool: Ecto.Adapters.SQL.Sandbox
```

## Core Modules

```elixir
# Event Store
defmodule MyApp.EventStore do
  use EventStore, otp_app: :my_app
  def init(config), do: {:ok, config}
end

# Commanded Application
defmodule MyApp.CommandedApp do
  use Commanded.Application,
    otp_app: :my_app,
    event_store: [
      adapter: Commanded.EventStore.Adapters.EventStore,
      event_store: MyApp.EventStore
    ]

  router(MyApp.Router)
end
```

## Router DSL

```elixir
defmodule MyApp.Router do
  use Commanded.Commands.Router

  # Identity: which field identifies the aggregate instance
  identify MyApp.Account, by: :account_id, prefix: "account-"
  identify MyApp.Order, by: :order_id, prefix: "order-"

  # Identity by function
  identify MyApp.Transfer, by: &(&1.from_account_id), prefix: "transfer-"

  # Dispatch commands to aggregate
  dispatch [OpenAccount, DepositFunds, WithdrawFunds, CloseAccount],
    to: MyApp.Account

  # Dispatch with middleware
  dispatch CreateOrder,
    to: MyApp.Order,
    middleware: [MyApp.Middleware.Validation]

  # Dispatch with separate command handler
  dispatch OpenAccount,
    to: OpenAccountHandler,
    aggregate: MyApp.Account,
    identity: :account_number

  # Dispatch with lifespan control
  dispatch CloseAccount,
    to: MyApp.Account,
    lifespan: MyApp.AccountLifespan
end
```

## Command Dispatch Options

| Option | Type | Description |
|--------|------|-------------|
| `timeout` | integer | Command execution timeout (ms) |
| `consistency` | `:eventual` \| `:strong` | Wait for projections before returning |
| `metadata` | map | Attached to events (user_id, correlation_id) |
| `returning` | `:aggregate_state` \| `:aggregate_version` \| `:events` \| `:execution_result` | What to return on success |

```elixir
# Dispatch with options — from Commanded test suite
MyApp.CommandedApp.dispatch(command)
MyApp.CommandedApp.dispatch(command, timeout: 30_000)
MyApp.CommandedApp.dispatch(command, consistency: :strong)
MyApp.CommandedApp.dispatch(command, returning: :aggregate_state)
MyApp.CommandedApp.dispatch(command,
  metadata: %{user_id: "user-1", correlation_id: UUID.uuid4()}
)

# Router dispatch with application
MyApp.Router.dispatch(command, application: MyApp.CommandedApp)
```

## Aggregate Callbacks

| Callback | Signature | Returns | Purpose |
|----------|-----------|---------|---------|
| `execute/2` | `(state, command) -> result` | event \| [events] \| `{:ok, event}` \| `{:ok, [events]}` \| `{:error, reason}` | Handle command, emit events |
| `apply/2` | `(state, event) -> state` | Updated struct | Reconstruct state from event |
| `after_command/1` | `(command) -> timeout` | `:timer.minutes(30)` \| `:infinity` \| `{:stop, reason}` | Control aggregate lifespan |

## Aggregate Lifespan

```elixir
defmodule MyApp.AccountLifespan do
  @behaviour Commanded.Aggregates.AggregateLifespan

  def after_event(%AccountClosed{}), do: :stop
  def after_event(_), do: :timer.hours(1)
  def after_command(_), do: :infinity
  def after_error(_), do: :stop
end
```

## Command Middleware

```elixir
@behaviour Commanded.Middleware

@impl true
def before_dispatch(%Pipeline{command: command} = pipeline) do
  case validate(command) do
    :ok -> pipeline
    {:error, reason} ->
      pipeline |> Pipeline.respond({:error, reason}) |> Pipeline.halt()
  end
end

@impl true
def after_dispatch(pipeline), do: pipeline

@impl true
def after_failure(pipeline), do: pipeline
```

## Event Handler

```elixir
defmodule MyApp.EventHandlers.Notifier do
  use Commanded.Event.Handler,
    application: MyApp.CommandedApp,
    name: __MODULE__,
    consistency: :eventual,     # :eventual (default) or :strong
    start_from: :origin,        # :origin, :current, or event number
    subscription_opts: [
      checkpoint_threshold: 100,
      checkpoint_after: 5_000
    ]

  def handle(%AccountOpened{} = event, metadata) do
    # metadata contains: event_id, event_number, stream_id, stream_version,
    #   correlation_id, causation_id, created_at
    :ok
  end

  # Error callback — from Commanded's event_handler_error_handling_test.exs
  def error({:error, reason}, event, %{context: context}) do
    failures = Map.get(context, :failures, 0)
    cond do
      failures >= 3 -> :skip
      transient?(reason) -> {:retry, :timer.seconds(5 * (failures + 1))}
      true -> :skip
    end
  end
end
```

### Error Callback Return Values

| Return | Effect |
|--------|--------|
| `:skip` | Skip this event, continue processing |
| `{:retry, delay_ms}` | Retry after delay |
| `{:retry, command}` | Retry with modified command (process managers) |
| `{:stop, reason}` | Stop the handler/process manager |

## Ecto Projector

```elixir
defmodule MyApp.Projectors.AccountSummary do
  use Commanded.Projections.Ecto,
    application: MyApp.CommandedApp,
    repo: MyApp.Repo,
    name: "AccountSummary"

  project(%AccountOpened{} = event, _metadata, fn multi ->
    Ecto.Multi.insert(multi, :account, %AccountSummary{
      account_id: event.account_id,
      email: event.email,
      balance: event.initial_balance,
      status: :open
    })
  end)

  project(%FundsDeposited{} = event, _metadata, fn multi ->
    Ecto.Multi.update_all(multi, :account,
      from(a in AccountSummary, where: a.account_id == ^event.account_id),
      set: [balance: event.new_balance]
    )
  end)
end
```

## Process Manager Callbacks

| Callback | Signature | Returns |
|----------|-----------|---------|
| `interested?/1` | `(event) -> result` | `{:start, id}` \| `{:continue, id}` \| `{:stop, id}` \| `{:start!, id}` \| `{:continue!, id}` \| `false` |
| `handle/2` | `(state, event) -> commands` | command \| [commands] \| `[]` |
| `apply/2` | `(state, event) -> state` | Updated struct |
| `error/3` | `(error, command_or_event, failure_context) -> result` | `:skip` \| `{:retry, delay}` \| `{:stop, reason}` |

### interested?/1 Return Values

| Return | Meaning |
|--------|---------|
| `{:start, id}` | Start new process manager instance |
| `{:continue, id}` | Continue existing instance |
| `{:stop, id}` | Stop and cleanup instance |
| `{:start!, id}` | Start if not exists, error if exists |
| `{:continue!, id}` | Continue, error if not started |
| `false` | Not interested in this event |

## Snapshotting

```elixir
defmodule MyApp.Account do
  use Commanded.Aggregates.Aggregate,
    snapshotting: [
      snapshot_every: 100,       # Snapshot every N events
      snapshot_version: 1        # Version for migration
    ]

  defstruct [:account_id, :balance, :status]
  # ...
end
```

## Event Upcasting

```elixir
# From Commanded's upcaster_test.exs
defmodule MyApp.EventUpcaster do
  @behaviour Commanded.Event.Upcaster

  def upcast(%{"event_type" => "Elixir.MyApp.Events.AccountOpened"} = event, _metadata) do
    data = Map.put_new(event["data"], "subscription_plan", "free")
    %{event | "event_type" => "Elixir.MyApp.Events.AccountOpened.V2", "data" => data}
  end

  def upcast(event, _metadata), do: event
end

# Register in config
config :my_app, MyApp.EventStore,
  upcasters: [MyApp.EventUpcaster]
```

## Serialization

```elixir
# Events must derive Jason.Encoder
defmodule MyApp.Events.AccountOpened do
  @derive Jason.Encoder
  defstruct [:account_id, :email, :name, :initial_balance, :opened_at]
end

# Commands (optional but recommended)
defmodule MyApp.Commands.OpenAccount do
  @derive Jason.Encoder
  defstruct [:account_id, :email, :name, :initial_balance]
end
```

## Metadata Fields

| Field | Type | Description |
|-------|------|-------------|
| `event_id` | string | Unique event identifier |
| `event_number` | integer | Global event sequence number |
| `stream_id` | string | Aggregate stream identifier |
| `stream_version` | integer | Stream-local version |
| `correlation_id` | string | Links related commands/events |
| `causation_id` | string | ID of event that caused this one |
| `created_at` | DateTime | When event was recorded |

## Mix Tasks

```bash
# Event store management
mix event_store.create           # Create database
mix event_store.drop             # Drop database
mix event_store.init             # Initialize schema (tables)
mix event_store.gen.migration    # Generate migration for schema changes

# Combined
mix do event_store.create, event_store.init

# Reset handler/projector subscription (replays all events)
mix commanded.reset --app MyApp.CommandedApp --handler MyApp.Projectors.AccountSummary
```

## Supervision Tree

```elixir
defmodule MyApp.Application do
  use Application

  def start(_type, _args) do
    children = [
      MyApp.Repo,                                    # Ecto (if using projections)
      MyApp.CommandedApp,                            # Commanded application
      MyApp.ProcessManagers.OrderFulfillment,        # Process managers
      MyApp.Projectors.AccountSummary,               # Projectors
      MyApp.EventHandlers.EmailNotifier              # Event handlers (notifiers)
    ]

    Supervisor.start_link(children, strategy: :one_for_one)
  end
end
```

## Concurrency and Consistency

```elixir
# Optimistic concurrency (default) — from Commanded dispatch tests
case MyApp.CommandedApp.dispatch(command) do
  :ok -> {:ok, :success}
  {:error, :concurrency_error} -> {:error, :conflict}  # Retry or handle
  {:error, reason} -> {:error, reason}
end

# Strong consistency — waits for handlers marked :strong
MyApp.CommandedApp.dispatch(command, consistency: :strong)

# Handler with strong consistency
use Commanded.Event.Handler,
  consistency: :strong  # Dispatch blocks until this handler processes
```

## InMemory Event Store (Testing)

```elixir
# Commanded provides an in-memory adapter for testing
# No external database needed
config :my_app, MyApp.CommandedApp,
  event_store: [
    adapter: Commanded.EventStore.Adapters.InMemory,
    serializer: Commanded.Serialization.JsonSerializer
  ]
```

## Troubleshooting

| Problem | Solution |
|---------|----------|
| "EventStore database has not been created" | `mix event_store.create && mix event_store.init` |
| Concurrency errors on dispatch | Aggregate identity inconsistent — check `identify/2` in router |
| Events not being processed | Verify handler in supervision tree, check handler `name` matches |
| Projection out of sync | `mix commanded.reset --app App --handler Handler` |
| Slow aggregate rehydration | Enable snapshotting: `snapshot_every: 100` |
