# Elixir Architecture, Anti-Patterns & Tooling Reference

> Supporting reference for the [Elixir skill](SKILL.md). Contains architecture layouts, configuration, Phoenix contexts, layered architecture, pipeline architecture, production patterns, and anti-patterns catalog. See also: [type-system.md](type-system.md), [debugging-profiling.md](debugging-profiling.md).

## Application Architecture

How to structure Elixir applications from single modules to multi-app deployments.
Covers project layouts, domain boundaries, layered architecture, and how OTP, behaviours,
and protocols combine to create maintainable systems.

## Configuration

Phoenix projects use five config files evaluated at different times:

| File | Evaluated | Purpose |
|------|-----------|---------|
| `config/config.exs` | Compile time | Shared defaults, library config, structural config |
| `config/dev.exs` | Compile time | Dev server, watchers, debug flags, hardcoded secrets |
| `config/test.exs` | Compile time | Test isolation, mock adapters, `server: false` |
| `config/prod.exs` | Compile time | `force_ssl`, cache manifests, prod adapters |
| `config/runtime.exs` | Boot time | **Secrets, env vars, anything that varies per deployment** |

**The critical rule:** Never call `System.get_env/1` in compile-time config files for values that vary per deployment. Use `runtime.exs` instead.

```elixir
# config/config.exs — shared compile-time defaults
import Config
config :my_app, ecto_repos: [MyApp.Repo]
config :my_app, MyAppWeb.Endpoint,
  url: [host: "localhost"],
  adapter: Bandit.PhoenixAdapter,
  pubsub_server: MyApp.PubSub,
  live_view: [signing_salt: "abc123"]
config :phoenix, :json_library, Jason
import_config "#{config_env()}.exs"  # MUST be last line

# config/runtime.exs — secrets and env vars
import Config
if config_env() == :prod do
  secret_key_base = System.fetch_env!("SECRET_KEY_BASE")
  config :my_app, MyAppWeb.Endpoint,
    url: [host: System.get_env("PHX_HOST", "example.com"), port: 443, scheme: "https"],
    secret_key_base: secret_key_base
  config :my_app, MyApp.Repo,
    url: System.fetch_env!("DATABASE_URL"),
    pool_size: String.to_integer(System.get_env("POOL_SIZE", "10"))
end
```

**Reading config at runtime:**

```elixir
# Compile-time — baked into .beam, recompile to change
@val Application.compile_env(:my_app, :some_key)      # nil if missing
@val Application.compile_env!(:my_app, :some_key)     # raises if missing

# Runtime — reads application env, set by config files
Application.fetch_env!(:my_app, :some_key)             # raises if missing
Application.get_env(:my_app, :some_key, default)       # returns default

# System env — raw OS variable, typically used in runtime.exs
System.fetch_env!("SECRET_KEY_BASE")                    # raises if missing
System.get_env("PORT", "4000")                          # returns default
```

**When to use `compile_env` vs `fetch_env!`:** Use `compile_env` when the value must be known at compile time (module attributes, macro expansion, `@gateway` in module body). Use `fetch_env!` for everything else — it reads the current value at runtime, respecting `runtime.exs` changes without recompilation.

## Project Layouts

### Single Application (Default — Start Here)

The standard Phoenix project. All code in one `lib/` tree, organized by context:

```
my_app/
├── lib/
│   ├── my_app/            # Domain layer
│   │   ├── accounts.ex    # Context: public API
│   │   ├── accounts/
│   │   │   ├── user.ex    # Schema (internal to context)
│   │   │   └── user_token.ex
│   │   ├── catalog.ex     # Another context
│   │   ├── catalog/
│   │   │   ├── product.ex
│   │   │   └── category.ex
│   │   ├── application.ex # Supervision tree
│   │   └── repo.ex        # Ecto Repo
│   └── my_app_web/        # Web layer
│       ├── controllers/
│       ├── live/
│       ├── components/
│       ├── router.ex
│       └── endpoint.ex
├── config/
├── test/
└── mix.exs
```

**When sufficient:** Most applications. Contexts provide domain boundaries without the overhead of multiple apps. Scale by adding contexts, not apps.

### Umbrella Project

Multiple OTP applications in one repository sharing build artifacts, deps, and config:

```
my_platform/                  # Root (no code here)
├── apps/
│   ├── core/                 # Domain logic, no web dependency
│   │   ├── lib/core/
│   │   │   ├── accounts.ex
│   │   │   └── billing.ex
│   │   └── mix.exs
│   ├── core_web/             # Phoenix web layer
│   │   ├── lib/core_web/
│   │   │   ├── endpoint.ex
│   │   │   └── router.ex
│   │   └── mix.exs           # deps: [{:core, in_umbrella: true}]
│   └── worker/               # Background processing
│       ├── lib/worker/
│       └── mix.exs           # deps: [{:core, in_umbrella: true}]
├── config/config.exs         # Shared config for ALL apps
├── mix.exs                   # apps_path: "apps"
└── mix.lock                  # Single lockfile
```

**Root mix.exs:**

```elixir
defmodule MyPlatform.MixProject do
  use Mix.Project
  def project do
    [apps_path: "apps", version: "0.1.0", deps: deps()]
  end
  defp deps, do: []  # Shared deps go here; app-specific deps in child mix.exs
end
```

**Child mix.exs:**

```elixir
defmodule Core.MixProject do
  use Mix.Project
  def project do
    [
      app: :core,
      version: "0.1.0",
      build_path: "../../_build",       # Shared build
      config_path: "../../config/config.exs",  # Shared config
      deps_path: "../../deps",          # Shared deps
      lockfile: "../../mix.lock",       # Shared lockfile
      deps: [{:ecto, "~> 3.12"}]       # App-specific deps
    ]
  end
end
```

**Key properties:**
- Single `mix test` runs all apps; `mix test --app core` runs one
- Single `config/config.exs` for all apps (both strength and limitation)
- Sibling deps via `in_umbrella: true` — resolved to `path: "../sibling"`
- All apps share the same dependency versions (no version conflicts possible)
- `mix new my_platform --umbrella` scaffolds the structure

**When to use:** Multiple teams working on distinct subsystems. Separate deployment targets (web vs worker vs API). Enforcing hard module boundaries at the OTP application level.

### Poncho Project

Independent Mix projects in one repository linked by path dependencies:

```
my_platform/
├── core/                     # Independent project
│   ├── lib/core/
│   ├── config/config.exs     # Own config
│   ├── mix.exs               # Own deps, own lockfile
│   └── mix.lock
├── web/                      # Independent project
│   ├── lib/web/
│   ├── config/config.exs
│   ├── mix.exs               # deps: [{:core, path: "../core"}]
│   └── mix.lock
└── worker/
    ├── lib/worker/
    ├── mix.exs               # deps: [{:core, path: "../core"}]
    └── mix.lock
```

**Key differences from umbrella:**
- Each app has its own config, deps, and lockfile
- Different apps can use different dependency versions
- No shared build directory — fully independent compilation
- No `mix test` from root — test each app separately
- More independence, more management overhead

**When to use:** Apps need different dependency versions. Apps have different release cycles. Migrating toward fully independent Hex packages. Teams need complete autonomy.

### Layout Decision Guide

| Signal | Layout |
|--------|--------|
| Single team, one deployable | Single app with contexts |
| Need hard compile-time boundaries | Umbrella |
| Different teams, shared config OK | Umbrella |
| Different dependency versions needed | Poncho |
| Extracting a library for reuse | Poncho → eventual Hex package |
| "Should I split?" uncertainty | Don't split — use contexts |

## Phoenix Contexts — Domain-Driven Boundaries

Contexts are Elixir modules that group related functionality as a public API boundary.
They implement DDD's bounded context pattern.

### Context Structure

```elixir
defmodule MyApp.Catalog do
  @moduledoc "Product catalog management."

  import Ecto.Query, warn: false
  alias MyApp.Repo
  alias MyApp.Catalog.{Product, Category}

  # --- Public API ---

  def list_products, do: Repo.all(Product)
  def get_product!(id), do: Repo.get!(Product, id)

  def create_product(attrs \\ %{}) do
    %Product{}
    |> Product.changeset(attrs)
    |> Repo.insert()
  end

  def update_product(%Product{} = product, attrs) do
    product
    |> Product.changeset(attrs)
    |> Repo.update()
  end

  def delete_product(%Product{} = product), do: Repo.delete(product)

  # For LiveView forms — returns changeset for tracking
  def change_product(%Product{} = product, attrs \\ %{}) do
    Product.changeset(product, attrs)
  end
end
```

**Schema is internal to context:**

```elixir
defmodule MyApp.Catalog.Product do
  use Ecto.Schema
  import Ecto.Changeset

  schema "products" do
    field :title, :string
    field :price, :decimal
    many_to_many :categories, MyApp.Catalog.Category,
      join_through: "product_categories"
    timestamps(type: :utc_datetime)
  end

  @doc false  # Not part of public API
  def changeset(product, attrs) do
    product
    |> cast(attrs, [:title, :price])
    |> validate_required([:title, :price])
    |> validate_number(:price, greater_than: 0)
  end
end
```

### Generators Create Contexts

```bash
mix phx.gen.context Catalog Product products title:string price:decimal
# Creates: lib/my_app/catalog.ex, lib/my_app/catalog/product.ex

mix phx.gen.live Catalog Product products title:string price:decimal
# Creates: context + LiveView + components + routes

mix phx.gen.html Catalog Product products title:string price:decimal
# Creates: context + controller + templates + routes

mix phx.gen.json Accounts User users name:string email:string
mix phx.gen.auth Accounts User users
```

### Generator Field Types

```bash
name:string  age:integer  price:decimal  active:boolean  body:text
published_at:datetime  birth_date:date  inserted_at:utc_datetime
user_id:references:users  tags:array:string  metadata:map
email:string:unique  # With modifiers
```

### Context Boundary Rules

**When to create a new context:**
- Different business domain (Catalog vs ShoppingCart vs Accounts)
- Different teams will own the code
- Data has distinct lifecycle (orders vs products)
- When in doubt, prefer separate contexts — merge later if needed

**When NOT to split:**
- Entities share the same aggregate root (Order and OrderItem → same context)
- Tightly coupled operations that always change together
- Splitting would require constant cross-context calls

### Cross-Context Communication

Contexts call each other through their public API — never bypass to schemas or Repo:

```elixir
defmodule MyApp.ShoppingCart do
  alias MyApp.Catalog

  def add_item_to_cart(cart, product_id) do
    # Call through Catalog context API — not Repo.get!(Product, id)
    product = Catalog.get_product!(product_id)

    %CartItem{quantity: 1, price_when_carted: product.price}
    |> CartItem.changeset(%{})
    |> Ecto.Changeset.put_assoc(:cart, cart)
    |> Ecto.Changeset.put_assoc(:product, product)
    |> Repo.insert()
  end
end
```

**Cross-context data references:** Use foreign keys at the schema level but always
fetch through context functions. Never `Repo.preload` across context boundaries.

### Internal Context Organization

Inside a context, organize however your team prefers — the context module is the boundary:

```
lib/my_app/catalog/
├── product.ex              # Schema
├── category.ex             # Schema
├── product_queries.ex      # Complex query builders
├── import_worker.ex        # Background processing
└── price_calculator.ex     # Pure business logic
```

All internal modules are private. Only `MyApp.Catalog` is the public API.

## Layered Architecture in Elixir

Elixir applications naturally form layers. The key principle: **dependencies point inward** —
outer layers depend on inner layers, never the reverse.

### The Standard Phoenix Layers

```
┌─────────────────────────────────────────────┐
│  Web Layer (MyAppWeb)                        │
│  Endpoint → Router → Pipeline → Controller   │
│  LiveView, Channels, Components              │
├─────────────────────────────────────────────┤
│  Domain Layer (MyApp)                        │
│  Contexts → Business Logic → Schemas         │
│  Pure functions, validation, rules           │
├─────────────────────────────────────────────┤
│  Infrastructure Layer                        │
│  Repo, PubSub, external API clients         │
│  Email, file storage, message queues         │
├─────────────────────────────────────────────┤
│  OTP Foundation                              │
│  Application, Supervision tree, Registry     │
│  GenServer, Task, ETS                        │
└─────────────────────────────────────────────┘
```

**Dependency direction:** Web → Domain → Infrastructure → OTP

### Behaviours as Layer Contracts

Behaviours define the contracts between layers — inner layers declare what they need,
outer layers provide the implementation:

```elixir
# Domain layer defines the contract
defmodule MyApp.PaymentGateway do
  @callback charge(amount :: Decimal.t(), card_token :: String.t()) ::
    {:ok, transaction_id :: String.t()} | {:error, reason :: String.t()}

  @callback refund(transaction_id :: String.t()) ::
    {:ok, refund_id :: String.t()} | {:error, reason :: String.t()}
end

# Infrastructure layer implements it
defmodule MyApp.PaymentGateway.Stripe do
  @behaviour MyApp.PaymentGateway

  @impl true
  def charge(amount, card_token) do
    # Stripe API call
  end

  @impl true
  def refund(transaction_id) do
    # Stripe API refund
  end
end

# Test layer provides a mock
Mox.defmock(MyApp.PaymentGateway.Mock, for: MyApp.PaymentGateway)

# Config selects implementation
# config :my_app, :payment_gateway, MyApp.PaymentGateway.Stripe
# config :my_app, :payment_gateway, MyApp.PaymentGateway.Mock  # test
```

**Context uses the behaviour, not the implementation:**

```elixir
defmodule MyApp.Billing do
  @gateway Application.compile_env(:my_app, :payment_gateway)

  def charge_order(order) do
    @gateway.charge(order.total, order.card_token)
  end
end
```

### Protocols as Polymorphic Boundaries

While behaviours define *contracts for implementations*, protocols define *contracts across types*.
Use protocols when different data types from different contexts need to be handled uniformly:

```elixir
# Define in shared/core layer
defprotocol MyApp.Exportable do
  @doc "Convert to export format"
  @spec to_csv_row(t()) :: [String.t()]
  def to_csv_row(entity)
end

# Each context implements for its types
defimpl MyApp.Exportable, for: MyApp.Catalog.Product do
  def to_csv_row(product) do
    [product.title, Decimal.to_string(product.price)]
  end
end

defimpl MyApp.Exportable, for: MyApp.Accounts.User do
  def to_csv_row(user) do
    [user.name, user.email]
  end
end
```

### Behaviour vs Protocol Decision

| Use Behaviour when... | Use Protocol when... |
|-----------------------|---------------------|
| Swapping implementations (Stripe vs PayPal) | Dispatching on data type (Product vs User) |
| External service integration | Cross-context polymorphism |
| Testing with Mox | Data format conversion |
| One module implements the whole contract | Each type implements its own logic |
| Compile-time or runtime config selects impl | Type of argument selects impl at call site |

### Real-World Layering Examples

**Ecto's layers:**
- Protocol layer: `Ecto.Queryable` — schemas, strings, queries all become queries
- Behaviour layer: `Ecto.Adapter` — PostgreSQL, MySQL, SQLite implement same contract
- Schema layer: Data definitions and changesets
- Repo layer: Public API that ties everything together

**Plug's layers:**
- Behaviour: `Plug` — `init/1` + `call/2` contract
- Builder: Compiles a pipeline of plugs into nested function calls
- Conn: Immutable request/response struct threaded through plugs

**Ash's layers:**
- Domain: Groups of resources with shared config
- Resource: Declarative entity definitions (actions, policies, relationships)
- DataLayer behaviour: Swappable persistence (ETS, PostgreSQL, custom)
- Authorizer behaviour: Pluggable authorization (policies, RBAC)
- Notifier behaviour: Event publishing after actions

## Pipeline Architecture

Elixir excels at pipeline patterns — sequential transformation of data through stages.

### Plug Pipeline (Synchronous, Request/Response)

```elixir
defmodule MyAppWeb.Router do
  use Phoenix.Router

  pipeline :browser do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_live_flash
    plug :protect_from_forgery
    plug :put_secure_browser_headers
    plug MyAppWeb.Auth  # Custom plug
  end

  pipeline :api do
    plug :accepts, ["json"]
    plug MyAppWeb.RateLimiter
    plug MyAppWeb.ApiAuth
  end
end
```

Each plug transforms `%Plug.Conn{}` and passes it forward. `halt/1` short-circuits.

### GenStage Pipeline (Back-Pressure, Async)

For high-throughput data processing where producers and consumers run at different speeds:

```
Producer (generates data)
  ↓ demand flows up, data flows down
ProducerConsumer (transforms)
  ↓
Consumer (processes results)
```

**Use when:** Event streams, log processing, real-time data pipelines. Back-pressure prevents
memory exhaustion when consumers are slower than producers.

### Broadway Pipeline (Declarative, Production-Ready)

Broadway builds on GenStage with a declarative API, batching, and fault tolerance:

```elixir
defmodule MyApp.Pipeline do
  use Broadway

  def start_link(_opts) do
    Broadway.start_link(__MODULE__,
      name: __MODULE__,
      producer: [
        module: {BroadwayKafka.Producer, hosts: [...], topics: ["events"]}
      ],
      processors: [
        default: [concurrency: 10]
      ],
      batchers: [
        default: [batch_size: 100, batch_timeout: 1000, concurrency: 5]
      ]
    )
  end

  @impl true
  def handle_message(_processor, message, _context) do
    message
    |> Message.update_data(&process_event/1)
  end

  @impl true
  def handle_batch(_batcher, messages, _batch_info, _context) do
    # Bulk insert all messages in one DB call
    data = Enum.map(messages, & &1.data)
    MyApp.Repo.insert_all(Event, data)
    messages
  end
end
```

**Broadway vs GenStage:** Use Broadway for message queue consumption (Kafka, SQS, RabbitMQ).
Use raw GenStage only when Broadway's declarative model doesn't fit.

### Absinthe Phase Pipeline (Branching, Document Transform)

Phases that can skip, insert, or replace subsequent phases:

```elixir
# Each phase returns control flow instructions
{:ok, result}                    # continue to next phase
{:jump, result, TargetPhase}     # skip ahead
{:insert, result, [ExtraPhases]} # insert phases before remaining
{:error, reason}                 # abort pipeline
```

**Use when:** Complex multi-stage document/AST transformations where phases may need to
branch based on intermediate results.

### OTP Applications as Architectural Boundaries

Each OTP application is a deployment unit with its own supervision tree, configuration,
and lifecycle. In umbrella/poncho projects, this maps directly to architectural boundaries.

#### Application = Boundary

```elixir
defmodule Core.Application do
  use Application

  @impl true
  def start(_type, _args) do
    children = [
      Core.Repo,
      {Phoenix.PubSub, name: Core.PubSub},
      Core.Accounts.SessionCleanup,
      Core.Catalog.SearchIndex
    ]
    Supervisor.start_link(children, strategy: :one_for_one, name: Core.Supervisor)
  end
end
```

**Each application controls:**
- Which processes start and in what order
- Restart strategy for its subsystems
- Configuration namespace (`Application.get_env(:core, ...)`)
- What it exposes to other applications (public modules)

#### Supervision Tree as Architecture Diagram

The supervision tree IS the architecture — it shows dependencies, failure domains, and
restart semantics:

```
Application.Supervisor (:one_for_one)
├── Telemetry                 # Observe everything from the start
├── Repo                      # Database (most things depend on this)
├── PubSub                    # Communication bus
├── DomainServices            # (:rest_for_one supervisor)
│   ├── Cache                 # Domain cache
│   ├── EventProcessor        # Depends on cache
│   └── NotificationService   # Depends on both
├── WorkerManager             # (:one_for_all supervisor)
│   ├── WorkerRegistry        # Registry for workers
│   └── WorkerSupervisor      # DynamicSupervisor for workers
└── Endpoint                  # Web — last (serve only when ready)
```

**Architectural rules expressed through supervision:**
- **Start order = dependency order** (top to bottom)
- **`:rest_for_one`** = later children depend on earlier ones
- **`:one_for_all`** = tightly coupled subsystem (Registry + DynamicSupervisor)
- **`:one_for_one`** = independent subsystems at the top level
- **Endpoint last** = don't accept requests until everything is ready

### Process Architecture Patterns

#### Process-Per-Service (Most Common)

One long-lived process per service. Processes represent capabilities, not entities:

```elixir
children = [
  MyApp.Repo,           # One DB connection pool
  MyApp.Cache,          # One cache service
  MyApp.Mailer,         # One email sender
  MyApp.RateLimiter     # One rate limiting service
]
```

**Use when:** Service has shared state (connection pool, cache, counters). Most applications.

#### Process-Per-Entity (When Needed)

One process per domain entity. Each instance manages its own state:

```elixir
# Game server — one process per active game
DynamicSupervisor.start_child(GameSupervisor, {GameServer, game_id: id})

# IoT device — one process per connected device
DynamicSupervisor.start_child(DeviceSupervisor, {DeviceHandler, device_id: id})
```

**Use when:** Entities have independent state, need isolated failure domains, or communicate
asynchronously. Watch for: memory use scales linearly with entity count.

#### Hybrid: Service + Entity Pool

Service process manages the lifecycle, DynamicSupervisor manages instances:

```elixir
# Service layer decides when to spawn entities
defmodule MyApp.GameManager do
  def start_game(params) do
    game_id = generate_id()
    {:ok, _pid} = DynamicSupervisor.start_child(
      MyApp.GameSupervisor,
      {MyApp.GameServer, {game_id, params}}
    )
    {:ok, game_id}
  end

  def find_game(game_id) do
    MyApp.GameRegistry.lookup(game_id)
  end
end
```

### Event-Driven Architecture

#### PubSub for Decoupled Communication

Contexts communicate asynchronously through Phoenix.PubSub:

```elixir
# Publisher context — fires and forgets
defmodule MyApp.Orders do
  def complete_order(order) do
    with {:ok, order} <- mark_completed(order) do
      Phoenix.PubSub.broadcast(MyApp.PubSub, "orders", {:order_completed, order})
      {:ok, order}
    end
  end
end

# Subscriber context — handles events independently
defmodule MyApp.Notifications.OrderListener do
  use GenServer

  def start_link(_), do: GenServer.start_link(__MODULE__, nil, name: __MODULE__)

  @impl true
  def init(_) do
    Phoenix.PubSub.subscribe(MyApp.PubSub, "orders")
    {:ok, nil}
  end

  @impl true
  def handle_info({:order_completed, order}, state) do
    MyApp.Notifications.send_confirmation(order)
    {:noreply, state}
  end
end
```

**Benefits:** Contexts don't need to know about each other. Adding a new subscriber
requires zero changes to the publisher.

#### Registry for Fine-Grained PubSub

When Phoenix.PubSub topics are too coarse, use Registry with `:duplicate` keys:

```elixir
# Subscribe to specific entity events
Registry.register(MyApp.EventRegistry, {:order, order_id}, [])

# Broadcast to specific entity subscribers only
Registry.dispatch(MyApp.EventRegistry, {:order, order_id}, fn entries ->
  for {pid, _} <- entries, do: send(pid, {:order_updated, order})
end)
```

### Hex Packages as Architectural Units

When code matures beyond a single project, extract to Hex packages:

```elixir
# Private Hex org for company libraries
defp deps do
  [
    {:core_auth, "~> 1.0", organization: "my_company"},
    {:shared_schemas, "~> 2.0", organization: "my_company"}
  ]
end

# Path dep during development, Hex in production
{:my_lib, path: "../my_lib"}   # Development
{:my_lib, "~> 1.0"}           # Production (published to Hex)
```

**Progression:** Context module → Umbrella app → Poncho app → Hex package.
Each step increases independence and formality of the API boundary.

### Architecture BAD/GOOD

**Bypassing context boundaries:**
```elixir
# BAD: Controller queries Repo directly
def index(conn, _params) do
  products = Repo.all(Product)
  render(conn, :index, products: products)
end

# GOOD: Controller calls context
def index(conn, _params) do
  products = Catalog.list_products()
  render(conn, :index, products: products)
end
```

**God context with unrelated functionality:**
```elixir
# BAD: Single context does everything
defmodule MyApp.Admin do
  def list_users, do: ...
  def create_product, do: ...
  def process_payment, do: ...
  def send_notification, do: ...
end

# GOOD: Separate contexts per domain
defmodule MyApp.Accounts do ... end
defmodule MyApp.Catalog do ... end
defmodule MyApp.Billing do ... end
defmodule MyApp.Notifications do ... end
```

**Hard-coding external service in domain logic:**
```elixir
# BAD: Domain tightly coupled to Stripe
defmodule MyApp.Billing do
  def charge(order) do
    Stripe.Charge.create(%{amount: order.total, source: order.token})
  end
end

# GOOD: Behaviour-based abstraction
defmodule MyApp.Billing do
  @gateway Application.compile_env(:my_app, :payment_gateway)
  def charge(order), do: @gateway.charge(order.total, order.token)
end
```

**Business logic in GenServer callbacks:**
```elixir
# BAD: Domain logic mixed with process management
def handle_call({:apply_discount, code}, _from, state) do
  discount = case code do
    "SAVE10" -> Decimal.new("0.10")
    "SAVE20" -> Decimal.new("0.20")
    _ -> Decimal.new("0")
  end
  new_total = Decimal.mult(state.total, Decimal.sub(1, discount))
  {:reply, {:ok, new_total}, %{state | total: new_total}}
end

# GOOD: Pure function for logic, GenServer for state only
defmodule MyApp.Pricing do
  def apply_discount(total, code) do
    rate = discount_rate(code)
    Decimal.mult(total, Decimal.sub(1, rate))
  end

  defp discount_rate("SAVE10"), do: Decimal.new("0.10")
  defp discount_rate("SAVE20"), do: Decimal.new("0.20")
  defp discount_rate(_), do: Decimal.new("0")
end

def handle_call({:apply_discount, code}, _from, state) do
  new_total = MyApp.Pricing.apply_discount(state.total, code)
  {:reply, {:ok, new_total}, %{state | total: new_total}}
end
```

**Premature umbrella split:**
```elixir
# BAD: Umbrella with one real app and tons of cross-deps
apps/
├── auth/          # Used by every other app
├── core/          # Used by every other app
├── web/           # Depends on auth, core, billing, catalog
├── billing/       # Depends on auth, core
├── catalog/       # Depends on auth, core
└── notifications/ # Depends on auth, core, billing, catalog

# GOOD: Single app with clean contexts
lib/my_app/
├── accounts.ex
├── billing.ex
├── catalog.ex
└── notifications.ex
```

**Wrong supervision strategy:**
```elixir
# BAD: Registry and DynamicSupervisor under :one_for_one
# If Registry crashes, DynamicSupervisor has stale registrations
children = [
  {Registry, keys: :unique, name: MyApp.WorkerRegistry},
  {DynamicSupervisor, name: MyApp.WorkerSupervisor}
]
Supervisor.init(children, strategy: :one_for_one)  # WRONG

# GOOD: Tightly coupled processes under :one_for_all
Supervisor.init(children, strategy: :one_for_all)
```

### Production Architecture Decision Tree

When planning a new Elixir project or making structural decisions, work through these in order. **Priorities: robustness first, then replaceability, then simplicity.**

#### 1. Domain Boundaries (First Decision)

```
What business domains does this system handle?
├── Single domain → Single context (e.g., MyApp.Billing)
├── 2-5 related domains → Multiple contexts in one Mix app
├── 5+ domains with team boundaries → Consider umbrella/poncho
└── Unsure → Start with one context, split when pain emerges
```

**Key principle:** Contexts are cheap to split later but expensive to merge. Start broader, narrow down.

#### 2. Data Persistence Strategy

```
What data patterns does each domain need?
├── Standard CRUD → Ecto schemas + context functions + Repo
├── Event history / audit trail → Event sourcing (Commanded)
├── Complex state transitions → gen_statem + Ecto for persistence
├── High-read, low-write → ETS cache + Ecto for durability
├── Real-time ephemeral state → GenServer / process state only
└── Multiple patterns → Different strategies per context (mix freely)
```

#### 3. Process Architecture

```
Does this feature need its own process?
├── No shared mutable state → Pure functions (NO process needed)
├── Shared state, moderate access → GenServer
├── Shared state, high reads → ETS (`:read_concurrency`)
├── Per-entity isolation → DynamicSupervisor + Registry
├── Explicit state transitions → gen_statem
├── Background / scheduled work → Oban job
├── Data pipeline with backpressure → Broadway / GenStage
└── One-off parallel work → Task.async_stream
```

**Key principle:** Start without processes. Add a GenServer only when you need serialized access to shared mutable state. Most business logic belongs in pure functions called by contexts.

#### 4. Integration Boundaries

```
How does this component talk to external systems?
├── External API (Stripe, S3, etc.) → Behaviour + impl module + Mox in test
├── Internal service (another context) → Context public API (function calls)
├── Cross-context events → PubSub broadcast/subscribe (Phoenix.PubSub or :pg)
├── Cross-node communication → :erpc or distributed PubSub
└── Hardware / native code → NIF behind behaviour (for testability)
```

**Key principle:** Every external dependency should be behind a behaviour. This enables testing with Mox, swapping implementations, and isolating failure domains.

#### 5. Supervision Strategy

```
How should processes recover from failures?
├── Independent services → :one_for_one (most common at top level)
├── Ordered dependencies (cache → processor → notifier) → :rest_for_one
├── Tightly coupled (Registry + DynamicSupervisor) → :one_for_all
├── External-facing endpoint (web, API, MQTT) → Always last child (serve only when ready)
└── Database pools, PubSub, Telemetry → Always first children (infrastructure)
```

### Refactoring Decision Tree

Use when an existing codebase needs structural changes. **Priorities: preserve behaviour, improve boundaries, increase testability.**

| Signal | Refactoring | How |
|--------|------------|-----|
| Context module >500 lines | Split context | Extract related functions into a new context with its own schemas |
| Boundary module calls Repo directly | Introduce context | Move queries behind context functions |
| Business logic in GenServer callbacks | Extract pure functions | Create a domain module, call it from the callback |
| Hard-coded external service call | Introduce behaviour | Define callback, create impl module, configure via `compile_env` |
| Multiple `case` branches on same field | Extract to gen_statem | If transitions form a graph, use explicit state machine |
| Same query pattern repeated 3+ times | Composable query module | Create `MyApp.SomeContext.Queries` with chainable functions |
| Test requires complex setup of many modules | Missing boundaries | Introduce behaviour + Mox for isolation |
| Context A calls Context B calls Context A | Circular dependency | Extract shared logic into a third context, or merge |
| Single app with >20 contexts | Umbrella candidate | Group by deployment/team boundary into umbrella apps |
| GenServer bottleneck (message queue growing) | Partition or ETS | Use PartitionSupervisor or move hot reads to ETS |
| Multiple Repo.insert/update without transaction | Introduce Ecto.Multi | Wrap related operations in `Multi` + `Repo.transact` |
| Process state >100KB | Move to ETS | Large state causes GC pressure; ETS is off-heap |

### Component Reuse Patterns

Design components for reuse across projects by following these patterns:

| Pattern | When | Example |
|---------|------|---------|
| **Behaviour + default impl** | Swappable strategy | Payment gateway, notification channel, storage backend |
| **Context with behaviour boundary** | Reusable domain logic | Auth context with configurable token store |
| **Protocol + implementations** | Polymorphic data handling | Export formats, serialization, display rendering |
| **`use` macro with options** | Configurable scaffolding | Schema base module, test case templates |
| **Plug pipeline** | Composable request processing | Auth plugs, rate limiting, logging |
| **Telemetry events** | Observable components | Emit events, let consumers decide what to measure |
| **PubSub topics** | Decoupled communication | Publisher doesn't know subscribers exist |

**The replaceability test:** Can you swap this component's implementation without changing any business logic? If not, introduce a behaviour at the boundary.

## Production Patterns

> Production Phoenix patterns (changelog.com), Edge/IoT patterns (ExNVR), and Job Processing patterns (Oban) are in [production.md](production.md).
>
> **OTP supporting files:** Quick-reference tables in [otp-reference.md](otp-reference.md). Complete working examples (rate limiter, connection state machine, worker pool, cache, circuit breaker, pub/sub, task pipeline, telemetry, distributed counter, graceful shutdown) in [otp-examples.md](otp-examples.md). GenStage, Flow, Broadway, hot code upgrades, and production debugging in [otp-advanced.md](otp-advanced.md).

### Telemetry

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

### HTTP Clients

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

## Anti-Patterns Catalog

### Imperative Habits (Most Common LLM Mistakes)

```elixir
# BAD - #1 MISTAKE: accumulation with Enum.each
# Rebinding inside each does NOT affect outer scope
def collect(items) do
  result = []
  Enum.each(items, fn item ->
    result = [transform(item) | result]  # This rebinding is local!
  end)
  result  # Always returns []
end

# GOOD
def collect(items), do: Enum.map(items, &transform/1)


# BAD - nested case instead of with
def register(params) do
  case validate_email(params) do
    {:ok, email} ->
      case validate_password(params) do
        {:ok, password} ->
          case create_user(email, password) do
            {:ok, user} -> {:ok, user}
            {:error, reason} -> {:error, reason}
          end
        {:error, reason} -> {:error, reason}
      end
    {:error, reason} -> {:error, reason}
  end
end

# GOOD
def register(params) do
  with {:ok, email} <- validate_email(params),
       {:ok, password} <- validate_password(params),
       {:ok, user} <- create_user(email, password) do
    {:ok, user}
  end
end


# BAD - if/else chain for type dispatch
def format(value) do
  if is_list(value) do
    Enum.join(value, ", ")
  else
    if is_map(value) do
      Jason.encode!(value)
    else
      to_string(value)
    end
  end
end

# GOOD
def format(value) when is_list(value), do: Enum.join(value, ", ")
def format(value) when is_map(value), do: Jason.encode!(value)
def format(value), do: to_string(value)


# BAD - nil checking with if
def greet(user) do
  if user != nil do
    if user.name != nil do
      "Hello, #{user.name}"
    else
      "Hello, anonymous"
    end
  else
    "Hello, guest"
  end
end

# GOOD
def greet(%{name: name}) when is_binary(name), do: "Hello, #{name}"
def greet(%{}), do: "Hello, anonymous"
def greet(nil), do: "Hello, guest"


# BAD - string concatenation in loop (O(n^2))
def build_csv(rows) do
  Enum.reduce(rows, "", fn row, acc ->
    acc <> Enum.join(row, ",") <> "\n"
  end)
end

# GOOD - IO list, single binary conversion
def build_csv(rows) do
  rows
  |> Enum.map(fn row -> [Enum.intersperse(row, ","), "\n"] end)
  |> IO.iodata_to_binary()
end

# ALSO GOOD
def build_csv(rows) do
  Enum.map_join(rows, "\n", &Enum.join(&1, ","))
end


# BAD - imperative index tracking
def process_with_index(list) do
  {result, _} = Enum.reduce(list, {[], 0}, fn item, {acc, i} ->
    {[{i, transform(item)} | acc], i + 1}
  end)
  Enum.reverse(result)
end

# GOOD
list
|> Enum.with_index()
|> Enum.map(fn {item, i} -> {i, transform(item)} end)


# BAD - try/rescue for flow control
def parse_int(str) do
  try do
    {:ok, String.to_integer(str)}
  rescue
    ArgumentError -> {:error, :invalid}
  end
end

# GOOD
def parse_int(str) do
  case Integer.parse(str) do
    {int, ""} -> {:ok, int}
    {_int, _rest} -> {:error, :trailing_chars}
    :error -> {:error, :invalid}
  end
end


# BAD - Enum.each discards return values silently
def send_notifications(users) do
  Enum.each(users, fn user ->
    Mailer.send(user.email, "Hello")  # Return value discarded, no error visibility
  end)
end

# GOOD - collect results
def send_notifications(users) do
  results = Enum.map(users, fn user ->
    case Mailer.send(user.email, "Hello") do
      {:ok, _} -> {:ok, user.id}
      {:error, reason} -> {:error, user.id, reason}
    end
  end)
  {sent, failed} = Enum.split_with(results, &match?({:ok, _}, &1))
  %{sent: length(sent), failed: failed}
end


# BAD - sequential rebinding instead of pipeline
def process(data) do
  data = Map.put(data, :step1, compute_step1(data))
  data = Map.put(data, :step2, compute_step2(data))
  data = Map.put(data, :step3, compute_step3(data))
  data
end

# GOOD - if steps are independent
def process(data) do
  Map.merge(data, %{
    step1: compute_step1(data),
    step2: compute_step2(data),
    step3: compute_step3(data)
  })
end

# GOOD - if steps depend on prior results
def process(data) do
  data
  |> then(fn d -> Map.put(d, :step1, compute_step1(d)) end)
  |> then(fn d -> Map.put(d, :step2, compute_step2(d)) end)
  |> then(fn d -> Map.put(d, :step3, compute_step3(d)) end)
end


# BAD - boolean flag parameters
fetch_users(true, false)

# GOOD - keyword options
fetch_users(active: true, preload: false)

# BETTER - separate functions for distinct behaviors
fetch_active_users()
```

### Control Flow

```elixir
# BAD: if/else instead of pattern matching
def status(user), do: if user.active, do: :active, else: :inactive

# GOOD
def status(%{active: true}), do: :active
def status(%{active: false}), do: :inactive

# BAD: Boolean parameters
fetch_users(true)

# GOOD: Separate functions
fetch_active_users()
```

### Pattern Matching Gotchas

```elixir
# BAD: %{} matches ANY map, not just empty maps
def handle(%{}), do: :empty       # Matches %{a: 1} too!
def handle(map), do: :has_keys

# GOOD: Guard for empty map
def handle(map) when map == %{}, do: :empty
def handle(map) when map_size(map) == 0, do: :empty
def handle(map), do: :has_keys

# BAD: Atom keys don't match string keys (very common mistake with params)
%{name: name} = %{"name" => "Jo"}  # MatchError! Atom :name != string "name"

# GOOD: Match with the correct key type
%{"name" => name} = params          # External data (JSON, forms) uses string keys
%{name: name} = internal_map        # Internal data uses atom keys

# BAD: Keyword list order matters in pattern matching
def process([a: _, b: _]), do: :matched
process([b: 1, a: 2])  # => Falls through, no match!

# GOOD: Use Keyword functions or convert to map
def process(opts) when is_list(opts) do
  a = Keyword.get(opts, :a)
  b = Keyword.get(opts, :b)
end

# BAD: Pattern matching integers with floats (strict type matching)
def handle(1), do: :one
handle(1.0)  # => FunctionClauseError! 1 != 1.0 in patterns

# BAD: Assuming 0.0 is the only zero (IEEE 754 has -0.0)
def handle(0.0), do: :zero
handle(Float.ceil(-0.01))  # => FunctionClauseError! Returns -0.0

# GOOD: Use guards for numeric comparisons, not pattern literals
def handle(n) when n == 0, do: :zero      # Works for 0, 0.0, and -0.0
def handle(n) when n == 1, do: :one       # Works for both 1 and 1.0
def handle(n) when is_number(n), do: :number

# BAD: Float equality (precision issues)
0.1 + 0.2 == 0.3  # => false!

# GOOD: Compare with tolerance for floats
def nearly_equal?(a, b, epsilon \\ 1.0e-10) do
  abs(a - b) < epsilon
end
```

### Cross-Type Comparisons

Elixir allows comparisons between any types via term ordering:

```elixir
# SURPRISING: This returns true!
"1" > 2  # => true (strings > numbers in term order)

# Term ordering (lowest to highest):
# number < atom < reference < function < port < pid < tuple < map < list < binary

# BAD: Comparing user input without type checking
def authorized?(level) when level > 5, do: true  # "admin" > 5 is true!

# GOOD: Validate type first
def authorized?(level) when is_integer(level) and level > 5, do: true
```

### Data Structures

```elixir
# DANGEROUS: Atoms from user input (exhausts atom table ~1M limit)
String.to_atom(user_input)
Jason.decode!(json, keys: :atoms)       # Atoms from JSON keys
:erlang.binary_to_term(data)            # Atoms from untrusted binary

# GOOD: Explicit mapping (preferred for user input)
defp convert_status("pending"), do: :pending
defp convert_status("active"), do: :active
defp convert_status("inactive"), do: :inactive
defp convert_status(_), do: :unknown

# SAFE: to_existing_atom (only works if atom already exists)
String.to_existing_atom(user_input)
Jason.decode!(json, keys: :strings)     # Default, safe
:erlang.binary_to_term(data, [:safe])   # Rejects new atoms

# BAD: String concatenation in loops (O(n^2))
Enum.reduce(items, "", fn i, acc -> acc <> "#{i}\n" end)

# GOOD: IO lists
items |> Enum.map(&["Item: ", &1, "\n"]) |> IO.iodata_to_binary()
```

### Processes & OTP

```elixir
# BAD: GenServer bottleneck for reads
GenServer.call(Cache, {:get, key})

# GOOD: Direct ETS reads
:ets.lookup(:cache, key)

# BAD: Blocking operations in GenServer callbacks
def handle_call(:fetch, _from, state) do
  data = HTTPClient.get!(url)  # Blocks ALL message processing!
  {:reply, data, state}
end

# GOOD: Offload to Task, reply asynchronously
def handle_call(:fetch, from, state) do
  Task.start(fn ->
    data = HTTPClient.get!(url)
    GenServer.reply(from, data)
  end)
  {:noreply, state}
end

# BAD: Large data in process state (memory pressure, slow GC)
def init(_), do: {:ok, %{cache: load_million_records()}}

# GOOD: Store large data in ETS, reference from state
def init(_) do
  :ets.new(:cache, [:named_table, :public, read_concurrency: true])
  {:ok, %{table: :cache}}
end

# BAD: Unbounded mailbox growth
def handle_cast(:process, state) do
  # Slow processing while messages flood in
  Process.sleep(1000)
  {:noreply, state}
end

# GOOD: Apply backpressure or use call instead of cast
def handle_call(:process, _from, state) do
  # Caller blocks until complete - natural backpressure
  result = slow_operation()
  {:reply, result, state}
end

# BAD: Unsupervised GenServer - single point of failure
GenServer.start_link(Worker, [])

# GOOD: Always supervise
children = [{Worker, []}]
Supervisor.start_link(children, strategy: :one_for_one)

# BAD: Messy conditionals in supervisor init
children = if x, do: [A], else: []
children = if y, do: children ++ [B], else: children

# GOOD: Each module controls its own startup
children = [A, B, C]  # Each returns :ignore if disabled

# BAD: Task.async in a GenServer — crash kills the GenServer
def handle_call(:work, _from, state) do
  task = Task.async(fn -> risky_work() end)
  result = Task.await(task)
  {:reply, result, state}
end

# GOOD: Task.Supervisor.async_nolink — GenServer survives task crash
def handle_call(:work, _from, state) do
  task = Task.Supervisor.async_nolink(MyApp.TaskSupervisor, fn -> risky_work() end)
  {:reply, :started, %{state | task_ref: task.ref}}
end

# BAD: DynamicSupervisor without max_children — unbounded growth
DynamicSupervisor.init(strategy: :one_for_one)

# GOOD: Set a limit based on system capacity
DynamicSupervisor.init(strategy: :one_for_one, max_children: 10_000)

# BAD: Registry + DynamicSupervisor under :one_for_one — stale registrations if Registry crashes
children = [{Registry, keys: :unique, name: Reg}, {DynamicSupervisor, name: DS}]
Supervisor.init(children, strategy: :one_for_one)

# GOOD: Use :one_for_all — Registry crash restarts DynamicSupervisor too
Supervisor.init(children, strategy: :one_for_all)

# BAD: Using :queue.len/1 in loops (O(n) each call)
while :queue.len(q) > 0, do: ...

# GOOD: Track length manually
%{queue: q, length: 0}
```

**OTP Health Checks:**
- Monitor mailbox size: `Process.info(pid, :message_queue_len)`
- Check memory: `Process.info(pid, :memory)`
- Unsupervised processes = silent failures

### Errors

```elixir
# BAD: Swallowing errors
try do
  risky_operation()
rescue
  _ -> :ok
end

# BAD: try/rescue for expected errors
try do
  Jason.decode!(json)
rescue
  e -> {:error, e}
end

# GOOD: Use non-bang functions
Jason.decode(json)
```

### Performance

```elixir
# BAD: N+1 queries
posts = Repo.all(Post)
Enum.map(posts, fn p -> Repo.get(User, p.user_id) end)

# GOOD: Preload
Post |> Repo.all() |> Repo.preload(:author)

# BAD: Loading large files into memory
File.read!("huge.txt") |> String.split("\n") |> Enum.map(&process/1)

# GOOD: Streaming
File.stream!("huge.txt") |> Stream.map(&process/1) |> Enum.to_list()
```

### Distribution

```elixir
# BAD: Using casts across nodes (no delivery guarantee)
GenServer.cast({:global, :service}, {:update, data})

# GOOD: Use calls for distributed operations (confirms delivery)
GenServer.call({:global, :service}, {:update, data})

# BAD: Sending lambdas to remote nodes (code must match exactly)
Node.spawn(:remote@host, fn -> process(data) end)

# GOOD: Use MFA (Module, Function, Arguments)
Node.spawn(:remote@host, MyModule, :process, [data])

# BAD: Ignoring network partitions
# Nodes silently disconnect, your code keeps trying

# GOOD: Monitor node connections
:net_kernel.monitor_nodes(true)
# Receive {:nodeup, node} and {:nodedown, node} messages

# BAD: Assuming global registration is instant
:global.register_name(:my_service, self())
# Other nodes may not see it immediately

# GOOD: Understand eventual consistency
# Global registration requires cluster-wide lock and propagation
```

**Distribution Principles:**
- Always use `call` (not `cast`) across nodes - you need delivery confirmation
- Use MFA, not lambdas, for remote spawning
- Handle `:nodedown` messages - partitions happen
- Global registration is eventually consistent

