# Elixir Language Patterns — Deep Dive

> Supporting reference for the [Elixir skill](SKILL.md). Contains extended pattern matching, guards, comprehensions, pipelines, behaviours, protocols, streams, error handling, and advanced patterns. See also: [code-style.md](code-style.md), [documentation.md](documentation.md).

## Pattern Matching (extended)

### Multi-Clause Functions Over Conditionals

Use multiple function clauses with pattern matching instead of if/else:

```elixir
# BAD
def process_user(user) do
  if user != nil do
    if user.active, do: {:ok, user}, else: {:error, "inactive"}
  else
    {:error, "not found"}
  end
end

# GOOD
def process_user(%{active: true} = user), do: {:ok, user}
def process_user(%{active: false}), do: {:error, "inactive"}
def process_user(nil), do: {:error, "not found"}
```

Order clauses from most specific to least specific (catch-all last).

#### Pattern-Based Dispatch

```elixir
defmodule Formatter do
  # Specific patterns first
  def format(%Date{} = date), do: Calendar.strftime(date, "%Y-%m-%d")
  def format(%DateTime{} = dt), do: Calendar.strftime(dt, "%Y-%m-%d %H:%M:%S")
  def format(%Time{} = time), do: Calendar.strftime(time, "%H:%M:%S")

  # Fallback last
  def format(value) when is_binary(value), do: value
  def format(value), do: to_string(value)
end
```

#### Bad: If/Else Chain

```elixir
# AVOID THIS
def format(value) do
  if is_struct(value, Date) do
    Calendar.strftime(value, "%Y-%m-%d")
  else
    if is_struct(value, DateTime) do
      Calendar.strftime(value, "%Y-%m-%d %H:%M:%S")
    else
      if is_binary(value) do
        value
      else
        to_string(value)
      end
    end
  end
end
```

#### Default Arguments

```elixir
# Header clause defines defaults
def paginate(query, opts \\ [])
def paginate(query, opts) do
  page = Keyword.get(opts, :page, 1)
  per_page = Keyword.get(opts, :per_page, 20)

  query
  |> limit(^per_page)
  |> offset(^((page - 1) * per_page))
end
```

### Core Pattern Matching

```elixir
# Function arguments — struct + field extraction
def current_path(%Plug.Conn{query_string: ""} = conn), do: conn.request_path
def current_path(%Plug.Conn{query_string: q} = conn), do: "#{conn.request_path}?#{q}"

# Tagged tuples
def handle({:ok, value}), do: process(value)
def handle({:error, reason}), do: log_error(reason)

# Lists — head/tail, non-empty, specific length
def sum([head | tail], acc), do: sum(tail, acc + head)
def sum([], acc), do: acc
def first([item | _]), do: item                    # First element
def first_two([a, b | _rest]), do: {a, b}          # First two
def singleton?([_]), do: true                       # Exactly one element
def singleton?(_), do: false
def non_empty?([_ | _]), do: true                  # Any non-empty list
def non_empty?(_), do: false

# String prefix matching with <>
def join("room:" <> room_id, _payload, socket), do: ...  # Phoenix Channel pattern
def parse("Bearer " <> token), do: {:ok, token}
def parse(_), do: {:error, :invalid_scheme}
# NOTE: <> can ONLY match prefixes, not suffixes. Use String.ends_with?/2 for suffixes.
```

#### Extracting Nested Data

```elixir
def get_user_city(%{address: %{city: city}}), do: {:ok, city}
def get_user_city(_), do: {:error, :no_city}

# With default
def get_user_city(%{address: %{city: city}}), do: city
def get_user_city(%{address: _}), do: "Unknown"
def get_user_city(_), do: "No address"
```

#### Binary Pattern Matching

```elixir
# Parse fixed-width fields
def parse_record(<<
  id::binary-size(8),
  _separator::binary-size(1),
  name::binary-size(20),
  _rest::binary
>>) do
  %{id: String.trim(id), name: String.trim(name)}
end

# Parse UTF-8 characters
def first_char(<<char::utf8, _rest::binary>>), do: <<char::utf8>>
def first_char(""), do: ""
```

#### Map Updates

```elixir
def increment_count(%{count: count} = state) do
  %{state | count: count + 1}
end

# With nested update
def update_user_email(state, user_id, email) do
  update_in(state, [:users, user_id, :email], fn _ -> email end)
end
```

### Pin Operator (^) — Match Against Existing Values

```elixir
# Simple pin — match against existing variable (don't rebind)
target_id = 42
Enum.find(users, fn %{id: ^target_id} -> true; _ -> false end)

# Pin as map key — essential for dynamic key lookup
key = :email
case map do
  %{^key => value} -> {:ok, value}   # Without ^, key would rebind!
  _ -> :error
end

# Double pin — assert both key AND value exist
%{^key => ^value} = map  # Crashes if key missing or value differs

# Pin in receive — correlate async responses (the definitive use case)
ref = make_ref()
send(server, {self(), ref, :request})
receive do
  {^ref, {:ok, result}} -> result    # Only matches OUR response
  {^ref, {:error, reason}} -> raise reason
end

# Pin in case with nested maps
case state.channels do
  %{^pid => {^topic, join_ref}} -> handle_existing(join_ref)
  %{^pid => _} -> handle_wrong_topic()
  _ -> handle_new()
end
```

### Nested Destructuring

```elixir
# 2-level deep — struct inside tuple
{%User{__meta__: %Ecto.Schema.Metadata{}}, _opts} = arg

# 2-level deep — map inside struct, binding both levels
def process(%JoinExpr{source: %Ecto.Query{} = subquery, qual: qual} = join) do
  {join, subquery, qual}
end

# Destructure accumulator + data in one function head
defp on_change(
  %{cardinality: :one, field: field} = meta,          # meta struct
  %{action: action, data: current} = changeset,        # changeset struct
  {parent, changes, halt, valid?}                       # tuple accumulator
) do
  ...
end

# Matching on __struct__ for dynamic struct dispatch
def primary_key(%{__struct__: schema} = struct) do
  schema.__schema__(:primary_key)
end
```

### match?/2 — Boolean Pattern Matching

```elixir
# Use match?/2 when you only need true/false, not extracted values
Enum.filter(items, &match?({:ok, _}, &1))
Enum.all?(values, &match?(%{__struct__: ^schema}, &1))
if match?({:error, _}, result), do: log_error(result)

# match? with guards
match?({in_type, _} when in_type in [:in, :one_of], config[:type])

# Use case when you need the matched values
case result do
  {:ok, value} -> use(value)      # Need value — use case
  {:error, _} -> handle_error()
end
```

### Multi-Clause Anonymous Functions

```elixir
# In Enum.reduce — different clauses for different data shapes
Enum.reduce(subscribers, %{}, fn
  {pid, _}, cache when pid == sender ->
    cache                                    # Skip sender
  {pid, {:fastlane, fast_pid, serializer}}, cache ->
    send(fast_pid, serialize(serializer))    # Fastlane delivery
    cache
  {pid, _}, cache ->
    send(pid, msg)                           # Normal delivery
    cache
end)

# In Task.async_stream result handling
|> Enum.map(fn
  {:ok, result} -> result
  {:exit, :timeout} -> :timed_out
end)
```

### Exception Pattern Matching

```elixir
# Rescue specific exception types
try do
  risky_operation()
rescue
  e in ArgumentError -> {:error, e.message}           # Bind + type
  e in [KeyError, UndefinedFunctionError] -> handle(e) # Multiple types
  _ in RuntimeError -> :runtime_error                  # Type only, ignore value
end

# catch vs rescue — catch matches throws (not exceptions)
try do
  Jason.decode!(data)
catch
  {:position, pos} -> {:error, %{position: pos}}     # Thrown values
  :throw, value -> {:error, value}                     # Explicit kind
end
```

### Pattern Matching in receive

```elixir
# Selective receive with pinned reference — request/response correlation
def call(server, request, timeout) do
  ref = make_ref()
  mon = Process.monitor(server)
  send(server, {:"$call", {self(), ref}, request})
  receive do
    {^ref, reply} ->
      Process.demonitor(mon, [:flush])
      reply
    {:DOWN, ^mon, _, _, reason} ->
      exit(reason)
  after
    timeout -> exit(:timeout)
  end
end

# Non-blocking drain — after 0 returns immediately if no match
def drain_mailbox(acc \\ []) do
  receive do
    msg -> drain_mailbox([msg | acc])
  after
    0 -> Enum.reverse(acc)
  end
end
```

## Guards (extended)

> Core guard patterns, allowed expressions, custom guards, and `in`/multiple `when` are in [SKILL.md](SKILL.md). This section covers additional examples.

```elixir
# Combined guard with pattern match
def valid_user?(%{age: age, active: active}) when age >= 18 and active == true, do: true
def valid_user?(_), do: false

# Type-safe operations — guard all inputs
def divide(a, b) when is_number(a) and is_number(b) and b != 0, do: a / b
def divide(_, 0), do: {:error, :division_by_zero}
def divide(_, _), do: {:error, :invalid_arguments}

# Guards in case (not just function heads)
def categorize(value) do
  case value do
    n when is_integer(n) and n < 0 -> :negative
    0 -> :zero
    n when is_integer(n) and n > 0 -> :positive
    f when is_float(f) -> :float
    _ -> :other
  end
end

# Additional Bitwise guards (beyond core list)
# Bitwise.&&&, Bitwise.|||, Bitwise.bsl, Bitwise.bsr also allowed in guards
```

## The With Statement (extended)

> Basic `with` pattern is in [SKILL.md](SKILL.md). This shows the full helper function pattern:

```elixir
defmodule MyApp.Orders do
  def create_order(user_id, cart_id, payment_method) do
    with {:ok, user} <- fetch_user(user_id),
         {:ok, cart} <- fetch_cart(cart_id, user),
         :ok <- validate_cart(cart),
         {:ok, charge} <- process_payment(cart, payment_method),
         {:ok, order} <- insert_order(user, cart, charge) do
      send_confirmation(user, order)
      {:ok, order}
    else
      {:error, :user_not_found} ->
        {:error, "User not found"}

      {:error, :cart_not_found} ->
        {:error, "Cart not found or doesn't belong to user"}

      {:error, :empty_cart} ->
        {:error, "Cannot create order with empty cart"}

      {:error, :payment_failed, reason} ->
        Logger.error("Payment failed: #{reason}")
        {:error, "Payment failed: #{reason}"}

      {:error, changeset} when is_struct(changeset, Ecto.Changeset) ->
        {:error, changeset}
    end
  end

  defp fetch_user(id) do
    case Repo.get(User, id) do
      nil -> {:error, :user_not_found}
      user -> {:ok, user}
    end
  end

  defp fetch_cart(cart_id, user) do
    case Repo.get_by(Cart, id: cart_id, user_id: user.id) do
      nil -> {:error, :cart_not_found}
      cart -> {:ok, Repo.preload(cart, :items)}
    end
  end

  defp validate_cart(%{items: []}), do: {:error, :empty_cart}
  defp validate_cart(_cart), do: :ok

  defp process_payment(cart, payment_method) do
    case PaymentGateway.charge(cart.total, payment_method) do
      {:ok, charge} -> {:ok, charge}
      {:error, reason} -> {:error, :payment_failed, reason}
    end
  end
end
```

## For Comprehensions (full)

Single-pass iteration with filtering and collection — more efficient than pipelines for complex transformations:

```elixir
# Basic with filtering (conditions after <-)
for line <- File.stream!("config.txt"), line != "", String.contains?(line, "=") do
  String.split(line, "=", parts: 2)
end
```

**Pattern matching in generators** — non-matching elements are silently skipped (no error):

```elixir
# Only process successful results — {:error, _} tuples silently skipped
for {:ok, val} <- results, do: val

# Destructure nested data
for {_key, %{name: name, active: true}} <- users_map, do: name

# Filter nil values while building a map (from Logger stdlib)
for {_k, v} = elem <- keyword, v != nil, into: %{}, do: elem
```

**Collect into different types with `into:`**

```elixir
# Into a map
for {k, v} <- [a: 1, b: 2], into: %{} do
  {k, v * 2}
end
# => %{a: 2, b: 4}

# Into a MapSet (from Mix xref — collect modules from multiple apps)
for app <- apps,
    module <- Application.spec(app, :modules),
    into: MapSet.new(),
    do: module

# Into a string
for c <- ?a..?z, into: "", do: <<c>>
# => "abcdefghijklmnopqrstuvwxyz"
```

**Binary comprehensions** — iterate bytes/bits of a binary:

```elixir
# URL encoding (from URI stdlib)
for <<byte <- string>>, into: "" do
  case percent(byte, &char_unreserved?/1) do
    "%20" -> "+"
    percent -> percent
  end
end

# Strip whitespace from base64 (from Base stdlib)
for <<char::8 <- string>>, char not in ~c"\s\t\r\n", into: <<>>, do: <<char::8>>

# Multi-byte binary parsing
for <<c1::8, c2::8 <- data>>, into: <<>> do
  <<decode(c1)::4, decode(c2)::4>>
end
```

**Accumulator control with `reduce:`**

```elixir
# Simple accumulator
for line <- File.stream!("data.csv"), reduce: %{totals: 0, count: 0} do
  acc ->
    value = line |> String.trim() |> String.to_integer()
    %{acc | totals: acc.totals + value, count: acc.count + 1}
end

# Tuple accumulator — partition fields (from Ecto schema)
for {name, {_, writable}} <- fields, reduce: {[], []} do
  {keep, drop} ->
    case writable do
      :always -> {[name | keep], drop}
      _ -> {keep, [name | drop]}
    end
end

# Process options while building state (from Absinthe)
for key <- @packed_keys, reduce: {options, reverse_map} do
  {options, reverse_map} ->
    value = Keyword.get(options, key, %{})
    if value == %{} do
      {options, reverse_map}
    else
      {packed, reverse_map} = pack_value(value, reverse_map)
      {Keyword.put(options, key, packed), reverse_map}
    end
end
```

**`:uniq` option** — deduplicate results inline:

```elixir
# Collect unique formatter options from dependencies (from Mix)
for dep <- deps,
    dep_opts = eval_formatter(dep),
    parenless_call <- dep_opts[:export][:locals_without_parens] || [],
    uniq: true,
    do: parenless_call

# Collect unique type variables
for type_expr <- args, var <- collect_vars(type_expr), uniq: true, do: var
```

**Multiple generators** (Cartesian product):

```elixir
for x <- 1..3, y <- 1..3, x <= y do
  {x, y}
end
# => [{1,1}, {1,2}, {1,3}, {2,2}, {2,3}, {3,3}]

# Nested generators with filter (from Mox — generate mock functions)
for behaviour <- behaviours,
    {fun, arity} <- behaviour.behaviour_info(:callbacks),
    {fun, arity} not in callbacks_to_skip do
  define_mock(fun, arity)
end
```

**When to prefer `for` over pipelines:**
- Pattern matching silently filters (no `Enum.filter` needed)
- Building maps/MapSets/strings (`into:`)
- Complex accumulation with tuple state (`reduce:`)
- Multiple generators with cross-product
- Binary iteration (`<<byte <- binary>>`)
- Deduplication needed (`:uniq`)

**When to prefer `Enum` pipeline over `for`:**
- Simple map/filter/reduce on a single collection
- Need lazy evaluation (`Stream`)
- Readability with named pipeline steps

## case and cond (full)

```elixir
# case - pattern match on single value
case fetch_user(id) do
  {:ok, user} -> process(user)
  {:error, :not_found} -> create_user(id)
  {:error, reason} -> log_error(reason)
end

# cond - multiple boolean conditions (no else if!)
cond do
  x > 10 -> :large
  x > 5 -> :medium
  x > 0 -> :small
  true -> :zero_or_negative  # Always end with true ->
end
```

**cond with variable binding** — assign in condition, use in body:

```elixir
# Priority-based config resolution (from Absinthe)
cond do
  executor = Keyword.get(opts, :executor) ->
    executor

  executor = get_in(opts, [:context, :streaming_executor]) ->
    executor

  schema && function_exported?(schema, :__streaming_executor__, 0) ->
    schema.__streaming_executor__()

  executor = Application.get_env(:my_app, :executor) ->
    executor

  true ->
    DefaultExecutor
end

# Only/except option handling (from Jason encoder)
cond do
  only = Keyword.get(opts, :only) ->
    validate_fields!(only, fields, ":only")
    only

  except = Keyword.get(opts, :except) ->
    validate_fields!(except, fields, ":except")
    fields -- [:__struct__ | except]

  true ->
    fields -- [:__struct__]
end
```

**cond for capability detection** — graceful degradation:

```elixir
cond do
  Code.ensure_loaded?(FastModule) -> FastModule
  Code.ensure_loaded?(FallbackModule) -> FallbackModule
  true -> DefaultModule
end
```

**cond for range/threshold branching:**

```elixir
# Format time values (from Credo)
cond do
  time > 1_000_000 -> "#{div(time, 1_000_000)}s"
  time > 1_000 -> "#{div(time, 1_000)}ms"
  true -> "#{time}μs"
end
```

**When to use cond vs case vs multi-clause:**

| Use | When |
|-----|------|
| `case` | Matching on ONE value's shape/pattern |
| `cond` | Multiple DIFFERENT boolean expressions |
| Multi-clause | Function dispatches on argument patterns |
| `with` | Chaining operations that may fail |

**cond is best for:** priority fallback chains, range/threshold checks, binding-in-condition patterns, and situations where each branch tests a completely different expression.

## Behaviours, Callbacks & @impl

### Rules for Behaviours (LLM)

1. **ALWAYS use `@impl true`** on every callback implementation — catches typos and missing callbacks at compile time
2. **Once you use `@impl` on ANY callback, you MUST use it on ALL callbacks** — the compiler warns about inconsistency
3. **Use `@impl BehaviourModule`** (not `@impl true`) when implementing multiple behaviours with overlapping callback names
4. **ALWAYS name parameters in `@callback` specs** — e.g., `key :: String.t()` not just `String.t()` — serves as documentation
5. **NEVER use behaviour inheritance** — compose multiple small behaviours instead (Ecto adapter pattern)
6. **ALWAYS pair `defoverridable` with `@behaviour`** when providing defaults in `__using__`
7. **Use behaviours for module-level polymorphism** (adapters, strategies); use **protocols for data-level polymorphism** (dispatch on first argument's type)
8. **Handle `@optional_callbacks` at call sites** with `function_exported?/3` — optional means the function may not exist

### Defining Behaviours

```elixir
defmodule MyApp.Storage do
  @doc "Fetch an item by key"
  @callback fetch(key :: String.t()) :: {:ok, term()} | {:error, :not_found}

  @callback store(key :: String.t(), value :: term()) :: :ok | {:error, term()}

  @optional_callbacks [store: 2]  # Not all implementations need write
end

defmodule MyApp.Storage.ETS do
  @behaviour MyApp.Storage

  @impl true
  def fetch(key), do: ...

  @impl true
  def store(key, value), do: ...
end
```

### @impl true vs @impl ModuleName

```elixir
# Single behaviour — use @impl true (common case)
defmodule MyServer do
  use GenServer
  @impl true
  def init(state), do: {:ok, state}
  @impl true
  def handle_call(:get, _from, state), do: {:reply, state, state}
end

# Multiple behaviours with overlapping names — use @impl ModuleName
defmodule MyAdapter do
  @behaviour Ecto.Adapter
  @behaviour Ecto.Adapter.Queryable
  @behaviour Ecto.Adapter.Schema

  @impl Ecto.Adapter
  def init(config), do: ...

  @impl Ecto.Adapter.Queryable
  def execute(adapter_meta, query_meta, query_cache, params, opts), do: ...

  @impl Ecto.Adapter.Schema
  def insert(adapter_meta, schema_meta, fields, on_conflict, returning, opts), do: ...
end
```

**Compiler warning traps:**
- `@impl true` on a non-callback function → warns "no behaviour specifies such callback"
- `@impl` on a private function → warns "@impl is always discarded for private functions"
- `@impl` set but no `def` follows → warns "module attribute @impl was set but no definition follows"
- Missing `@impl` when others have it → warns about the missing one

### Behaviour + use Macro — The Full Pattern

Libraries combine `@behaviour` with `use` to provide defaults:

```elixir
defmodule MyLib.Worker do
  # Step 1: Define the behaviour
  @callback process(term()) :: {:ok, term()} | {:error, term()}
  @callback handle_failure(term(), term()) :: :retry | :skip

  # Step 2: __using__ injects defaults + makes them overridable
  defmacro __using__(opts) do
    quote do
      @behaviour MyLib.Worker

      # Default implementation — users override what they need
      @impl MyLib.Worker
      def handle_failure(_item, _reason), do: :skip

      defoverridable MyLib.Worker  # Makes ALL callbacks overridable at once
      # Or: defoverridable handle_failure: 2  (specific functions)
    end
  end
end

# Usage — only implement what differs from defaults
defmodule MyApp.EmailWorker do
  use MyLib.Worker

  @impl true
  def process(email), do: {:ok, send_email(email)}

  # handle_failure/2 uses the default :skip — no need to implement
end
```

### Dynamic Dispatch via Behaviours (Adapter Pattern)

```elixir
# Define the behaviour (contract)
defmodule MyApp.Mailer do
  @callback send_email(to :: String.t(), subject :: String.t(), body :: String.t()) ::
    {:ok, term()} | {:error, term()}
end

# Implementations
defmodule MyApp.Mailer.SMTP do
  @behaviour MyApp.Mailer
  @impl true
  def send_email(to, subject, body), do: ...
end

defmodule MyApp.Mailer.SendGrid do
  @behaviour MyApp.Mailer
  @impl true
  def send_email(to, subject, body), do: ...
end

# Runtime dispatch — module from config
defmodule MyApp.Mailer.Dispatcher do
  def send(to, subject, body) do
    impl = Application.get_env(:my_app, :mailer, MyApp.Mailer.SMTP)
    impl.send_email(to, subject, body)
  end
end

# config/config.exs
config :my_app, :mailer, MyApp.Mailer.SendGrid

# config/test.exs — use Mox for testing
config :my_app, :mailer, MyApp.MockMailer
```

### Multi-Behaviour Composition (Ecto Adapter Pattern)

Elixir uses flat composition of multiple small behaviours — never inheritance:

```elixir
# Small, focused behaviour modules
defmodule MyLib.Adapter do
  @callback init(config :: keyword()) :: {:ok, state :: term()}
end

defmodule MyLib.Adapter.Read do
  @callback fetch(state :: term(), key :: term()) :: {:ok, term()} | {:error, :not_found}
end

defmodule MyLib.Adapter.Write do
  @callback store(state :: term(), key :: term(), value :: term()) :: :ok | {:error, term()}
end

# Implementors opt-in to what they support
defmodule MyLib.Adapter.ReadOnly do
  @behaviour MyLib.Adapter
  @behaviour MyLib.Adapter.Read
  # No Write behaviour — this adapter is read-only
end
```

### Behaviour vs Protocol — When to Use Which

| | Behaviour | Protocol |
|---|---|---|
| **Dispatch on** | Module identity (passed as config) | Data type of first argument |
| **Define with** | `@callback` | `defprotocol` |
| **Implement with** | `@behaviour` + `@impl` | `defimpl` |
| **Testing** | Works with Mox | Does not work with Mox |
| **Use for** | Adapters, strategies, services | Type-specific formatting, conversion |
| **Examples** | Ecto.Adapter, Plug, Phoenix.Channel | Enumerable, String.Chars, Jason.Encoder |

### Behaviour Introspection

```elixir
# Check if a module defines a behaviour
function_exported?(GenServer, :behaviour_info, 1)  # true

# List all callbacks
GenServer.behaviour_info(:callbacks)
# [{:init, 1}, {:handle_call, 3}, {:handle_cast, 2}, ...]

# List optional callbacks
GenServer.behaviour_info(:optional_callbacks)
# [{:handle_call, 3}, {:handle_cast, 2}, ...]

# Check what behaviours a module implements
MyServer.__info__(:attributes)[:behaviour]
# [GenServer]

# Check if a module implements a specific callback at runtime
function_exported?(MyModule, :handle_call, 3)
```

### defoverridable — Layered Default Implementations

Two forms exist:

```elixir
# Form 1: Override all callbacks of a behaviour at once
defmacro __using__(_opts) do
  quote do
    @behaviour Plug
    def init(opts), do: opts
    def call(conn, opts), do: build_pipeline(conn, opts)
    defoverridable Plug          # Makes init/1 and call/2 overridable
  end
end

# Form 2: Override specific functions by name/arity
defoverridable init: 1, call: 2, action: 2
```

**The `@before_compile` + `defoverridable` + `super` wrapping pattern** (used by Phoenix.Endpoint, Plug.Debugger, Plug.ErrorHandler):

```elixir
defmacro __before_compile__(_env) do
  quote do
    defoverridable call: 2
    def call(conn, opts) do
      try do
        super(conn, opts)      # Calls the previous definition
      rescue
        e -> handle_error(conn, e)
      end
    end
  end
end
```

This allows multiple modules to wrap `call/2` in layers — each layer calls `super` to invoke the previous definition.

### The DSL Recipe — __using__ + Accumulated Attributes + @before_compile

This three-step pattern is how Phoenix, Plug, Absinthe, and Ash build DSLs:

```elixir
# Step 1: __using__ — set up imports, register attributes, schedule compilation
defmacro __using__(_opts) do
  quote do
    @behaviour MyDSL
    import MyDSL, only: [register_handler: 2]
    Module.register_attribute(__MODULE__, :handlers, accumulate: true)
    @before_compile MyDSL
  end
end

# Step 2: Macros push data into accumulated attributes
defmacro register_handler(event, handler) do
  quote do
    @handlers {unquote(event), unquote(handler)}
  end
end

# Step 3: @before_compile reads attributes and generates functions
defmacro __before_compile__(env) do
  handlers = Module.get_attribute(env.module, :handlers)
  # NOTE: accumulated attributes are in REVERSE order (last declared first)

  for {event, handler} <- Enum.reverse(handlers) do
    quote do
      def handle(unquote(event)), do: unquote(handler).()
    end
  end
end
```

**Critical:** Accumulated attributes are stored in **reverse order** — `Module.register_attribute(mod, :attr, accumulate: true)` prepends each new value. Either:
- Design your `Enum.reduce` to consume reversed lists (Plug.Builder approach)
- Call `Enum.reverse()` explicitly when order matters (Phoenix.Socket approach)

### Behaviour for Swappable Storage Backends

Define a behaviour to allow different storage implementations:

```elixir
defmodule MyApp.ConfigStorage do
  @moduledoc "Behaviour for configuration storage backends"

  @callback setup() :: :ok
  @callback put(key :: atom, value :: term) :: :ok
  @callback get(key :: atom) :: term | nil
  @callback delete(key :: atom) :: :ok
  @callback list() :: [atom]

  @optional_callbacks setup: 0

  # Dynamic backend selection
  def get_module do
    case Application.get_env(:my_app, :config_storage, :persistent_term) do
      :ets -> MyApp.ConfigStorage.ETS
      :persistent_term -> MyApp.ConfigStorage.PersistentTerm
      module when is_atom(module) -> module
    end
  end
end

defmodule MyApp.ConfigStorage.PersistentTerm do
  @behaviour MyApp.ConfigStorage

  @impl true
  def put(key, value) do
    :persistent_term.put({__MODULE__, key}, value)
  end

  @impl true
  def get(key) do
    :persistent_term.get({__MODULE__, key}, nil)
  end

  @impl true
  def delete(_key) do
    # Intentionally no-op: persistent_term deletion is expensive
    :ok
  end

  @impl true
  def list do
    :persistent_term.get()
    |> Enum.filter(fn {{mod, _}, _} -> mod == __MODULE__; _ -> false end)
    |> Enum.map(fn {{_, key}, _} -> key end)
  end
end

defmodule MyApp.ConfigStorage.ETS do
  @behaviour MyApp.ConfigStorage
  @table __MODULE__

  @impl true
  def setup do
    :ets.new(@table, [:named_table, :public, read_concurrency: true])
    :ok
  end

  @impl true
  def put(key, value), do: :ets.insert(@table, {key, value}) && :ok

  @impl true
  def get(key) do
    case :ets.lookup(@table, key) do
      [{^key, value}] -> value
      [] -> nil
    end
  end

  @impl true
  def delete(key), do: :ets.delete(@table, key) && :ok

  @impl true
  def list, do: :ets.tab2list(@table) |> Enum.map(&elem(&1, 0))
end
```

## Protocols

### Rules for Protocols (LLM)

1. **PREFER single-function protocols** — the vast majority of stdlib/library protocols define exactly 1 function. Multi-function only for optimization callbacks (Enumerable pattern)
2. **ALWAYS put `@derive` BEFORE `defstruct`** (or `schema`) — the compiler warns if it comes after
3. **NEVER implement `for: Map` expecting it to match structs** — structs dispatch through `struct_impl_for/1`, not the Map implementation. `defimpl for: Map` only matches bare maps
4. **Use `@fallback_to_any true`** only when there IS a sensible default. Don't use it when failure should be loud (Enumerable, String.Chars don't use it)
5. **ALWAYS implement all 3 commands** in Collectable: `{:cont, elem}`, `:done`, `:halt`
6. **For Enumerable, return `{:error, __MODULE__}`** from `count/1`, `member?/2`, `slice/1` when O(1) isn't possible — this triggers the reduce-based fallback
7. **Use `Protocol.derive/3`** for structs you don't own instead of reaching into their modules
8. **Guard `for: BitString` implementations** with `is_binary/1` — BitString matches both binaries and bitstrings

### Defining Protocols

```elixir
# Most protocols: single function, no fallback
defprotocol MyApp.Renderable do
  @spec render(t()) :: iodata()
  def render(term)
end

# With @fallback_to_any for a sensible default or helpful error
defprotocol MyApp.Parameterizable do
  @fallback_to_any true
  @spec to_param(t()) :: String.t()
  def to_param(term)
end
```

**The 11 built-in types** you can implement for: `Atom`, `BitString`, `Float`, `Function`, `Integer`, `List`, `Map`, `PID`, `Port`, `Reference`, `Tuple` — plus `Any` (fallback).

### Implementing Protocols

```elixir
# For built-in types
defimpl MyApp.Renderable, for: BitString do
  def render(binary) when is_binary(binary), do: binary  # Guard: BitString includes non-binary bitstrings
  def render(bits), do: raise Protocol.UndefinedError, protocol: @protocol, value: bits
end

defimpl MyApp.Renderable, for: List do
  def render(iolist), do: iolist  # IO lists are already renderable
end

# For multiple types at once
defimpl MyApp.Renderable, for: [Integer, Float] do
  def render(number), do: to_string(number)
end

# For a struct — inside the struct's module (`:for` is optional)
defmodule MyApp.Widget do
  defstruct [:name, :html]

  defimpl MyApp.Renderable do
    def render(%{html: html}), do: html
  end
end

# For a struct — outside the module
defimpl MyApp.Renderable, for: MyApp.Alert do
  def render(%{message: msg, level: level}), do: ~s(<div class="#{level}">#{msg}</div>)
end
```

### @derive — Compile-Time Protocol Implementation

```elixir
defmodule MyApp.User do
  # @derive MUST come before defstruct/schema
  @derive {Jason.Encoder, only: [:id, :name, :email]}    # Selective JSON encoding
  @derive {Phoenix.Param, key: :username}                  # URL: /users/john_doe
  @derive {Inspect, only: [:id, :name]}                    # Hide password_hash in logs
  defstruct [:id, :name, :email, :username, :password_hash]
end

# For structs you don't own — use Protocol.derive/3
require Protocol
Protocol.derive(JSON.Encoder, SomeLibrary.Thing, only: [:id, :name])
```

**How @derive works:** The protocol's `__deriving__` macro (defined in the protocol or its Any implementation) generates a `defimpl` block at compile time. The derived implementation can pattern-match struct fields directly in function heads for optimal performance.

### @fallback_to_any — When to Use

| Protocol | Has `@fallback_to_any`? | Why |
|---|---|---|
| `Inspect` | Yes | Default `%Module{...}` printing for all structs |
| `Phoenix.Param` | Yes | Convention: assumes `:id` field exists |
| `Plug.Exception` | Yes | Default: return 500 status, empty actions |
| `Jason.Encoder` | Yes | But raises in Any — provides helpful derive instructions |
| `Enumerable` | **No** | `Enum.map` on non-enumerable must fail loudly |
| `String.Chars` | **No** | Silent garbage output would hide bugs |
| `Collectable` | **No** | No sensible default |

**Alternative to `@fallback_to_any`:** Use `@undefined_impl_description` (Elixir 1.18+) to customize the error message without providing a fallback:

```elixir
defprotocol MyApp.Encoder do
  @undefined_impl_description """
  protocol must be explicitly implemented.
  Add `@derive {MyApp.Encoder, only: [...]}` before defstruct.
  """
  def encode(term)
end
```

### Making a Derivable Protocol

```elixir
defprotocol MyApp.Cacheable do
  @fallback_to_any true
  @spec cache_key(t()) :: String.t()
  def cache_key(term)
end

defimpl MyApp.Cacheable, for: Any do
  # __deriving__/3 in Any impl (legacy approach, works in all versions)
  defmacro __deriving__(module, _struct, opts) do
    key_fields = Keyword.get(opts, :keys, [:id])
    quote do
      defimpl MyApp.Cacheable, for: unquote(module) do
        def cache_key(struct) do
          parts = Enum.map(unquote(key_fields), &Map.fetch!(struct, &1))
          Enum.join([unquote(inspect(module)) | parts], ":")
        end
      end
    end
  end

  # Fallback: use module name + :id
  def cache_key(%{__struct__: mod, id: id}), do: "#{inspect(mod)}:#{id}"
  def cache_key(term), do: raise Protocol.UndefinedError, protocol: @protocol, value: term
end

# Usage
@derive {MyApp.Cacheable, keys: [:tenant_id, :id]}
defstruct [:tenant_id, :id, :name]
```

### Struct Dispatch Precedence

Protocol dispatch for structs follows this priority:

1. **Explicit `defimpl` for the struct** — highest priority
2. **`@derive`-generated implementation** — compiled from `__deriving__` macro
3. **`Any` implementation** — only if `@fallback_to_any true`

**Structs NEVER match the `Map` implementation** — even though structs are maps, protocol dispatch checks `struct_impl_for/1` first. The `Map` impl only matches bare `%{}` maps:

```elixir
# BAD — this does NOT match structs
defimpl MyProtocol, for: Map do
  def my_func(map), do: ...   # Only bare maps, not %User{}, %Post{}, etc.
end

# GOOD — implement for the specific struct
defimpl MyProtocol, for: MyApp.User do
  def my_func(user), do: ...
end
```

### Protocol Introspection

```elixir
# Check if a value's type implements a protocol
Enumerable.impl_for([1, 2, 3])    #=> Enumerable.List
Enumerable.impl_for("string")     #=> nil (not implemented)
Enumerable.impl_for!("string")    #=> raises Protocol.UndefinedError

# Check consolidation (important for debugging)
Protocol.consolidated?(Enumerable)  #=> true (release) / false (dev)

# List all implementations (only when consolidated)
Enumerable.__protocol__(:impls)     #=> {:consolidated, [List, Map, ...]}

# Protocol metadata
Enumerable.__protocol__(:functions) #=> [count: 1, member?: 2, reduce: 3, slice: 1]

# Implementation metadata
Enumerable.List.__impl__(:protocol) #=> Enumerable
Enumerable.List.__impl__(:for)      #=> List
```

### Protocol Consolidation Gotchas

- **Mix auto-consolidates** at compile time — after consolidation, new `defimpl`s have no effect
- **In tests:** If test support files define protocol implementations, ensure `test/support` is in `elixirc_paths`:
  ```elixir
  defp elixirc_paths(:test), do: ["lib", "test/support"]
  defp elixirc_paths(_), do: ["lib"]
  ```
- **For optional dependencies:** Guard with `Code.ensure_loaded?/1`:
  ```elixir
  if Code.ensure_loaded?(Decimal) do
    defimpl Jason.Encoder, for: Decimal do
      def encode(decimal, opts), do: Jason.Encode.string(Decimal.to_string(decimal), opts)
    end
  end
  ```

### Application.compile_env vs Runtime Config

```elixir
# compile_env — value affects CODE GENERATION (conditional compilation)
# Used when the value determines what code is compiled
debug_errors? = Application.compile_env(:my_app, [MyEndpoint, :debug_errors], false)
if debug_errors?, do: plug Plug.Debugger  # Plug is included/excluded at compile time

# Runtime config — for operational values that can change between deploys
# Port, host, secret_key_base, database URL, etc.
config = Application.get_env(:my_app, MyEndpoint)
port = config[:port] || 4000
```

**Rule:** Use `compile_env` only when the value determines what code is generated. Use runtime config for everything else. `compile_env` triggers recompilation when the value changes.

## Stream & Enumerable Protocol

### The Enumerable Protocol — How It All Fits Together

`Enum` and `Stream` both operate on anything that implements the `Enumerable` protocol. This is the foundation — lists, maps, ranges, MapSets, streams, and any custom data structure that implements `Enumerable` all work with `Enum.*` and `Stream.*` functions.

```
Enumerable (protocol)
├── List, Map, Range, MapSet       ← built-in, eager
├── Stream                         ← lazy wrapper around Enumerable
├── File.stream!/1                 ← lazy file I/O
└── Your custom struct             ← implement Enumerable for it
```

**Key insight:** `Enum` functions are *eager* — they consume the entire enumerable and produce a result immediately. `Stream` functions are *lazy* — they return a new `%Stream{}` struct that describes the computation but doesn't execute it until consumed by an `Enum` function.

```elixir
# Eager — builds 3 intermediate lists
result = list
|> Enum.map(&transform/1)     # produces full list
|> Enum.filter(&valid?/1)     # produces full list
|> Enum.take(10)              # produces final list

# Lazy — builds 0 intermediate lists, processes element-by-element
result = list
|> Stream.map(&transform/1)   # returns %Stream{} (no work done)
|> Stream.filter(&valid?/1)   # returns %Stream{} (no work done)
|> Enum.take(10)              # NOW executes: pulls elements one at a time
```

### When to Use Stream vs Enum

| Use Enum when... | Use Stream when... |
|---|---|
| Collection fits in memory | Data is large or infinite |
| You need all results | You need only a subset (first N, take_while) |
| Simple 1-2 step pipeline | Chaining 3+ transforms on large data |
| Performance matters on small data | Avoiding intermediate allocations matters |
| Debugging (easier to inspect) | Wrapping I/O resources (files, sockets) |

**Rule of thumb:** Start with `Enum`. Switch to `Stream` when you have a reason — large data, partial consumption, or I/O resources.

### Stream Creators

```elixir
# From values
Stream.iterate(0, & &1 + 1)        # Infinite: 0, 1, 2, 3, ...
Stream.cycle([:a, :b, :c])         # Infinite: :a, :b, :c, :a, :b, ...
Stream.repeatedly(fn -> :rand.uniform() end)  # Infinite: random values
Stream.interval(1000)               # Emits incrementing integers every 1s

# From state (most versatile)
Stream.unfold(state, fun)
# fun receives state, returns {emit_value, next_state} or nil to halt
Stream.unfold(5, fn
  0 -> nil                         # halt
  n -> {n, n - 1}                  # emit n, continue with n-1
end) |> Enum.to_list()
# => [5, 4, 3, 2, 1]

# From external resources (files, sockets, database cursors)
Stream.resource(start_fun, next_fun, after_fun)
# start_fun: () -> resource       (open)
# next_fun: resource -> {[items], resource} | {:halt, resource}
# after_fun: resource -> :ok      (cleanup, always called)
File.stream!("path")               # Built-in: lazy line-by-line
File.stream!("path", :line)        # Same, explicit
File.stream!("path", 4096)         # Read in 4KB chunks (binary)
```

### Stream Transforms (Lazy — No Execution Until Consumed)

```elixir
Stream.map(stream, fun)             # transform each element
Stream.filter(stream, fun)          # keep matching elements
Stream.reject(stream, fun)          # remove matching elements
Stream.flat_map(stream, fun)        # transform and flatten
Stream.chunk_every(stream, n)       # group into chunks of n
Stream.chunk_while(stream, acc, chunk_fun, after_fun)  # custom chunking
Stream.take(stream, n)              # first n elements
Stream.take_every(stream, nth)      # every nth element
Stream.take_while(stream, fun)      # take while predicate holds
Stream.drop(stream, n)              # skip first n
Stream.drop_while(stream, fun)      # skip while predicate holds
Stream.dedup(stream)                # remove consecutive duplicates
Stream.dedup_by(stream, fun)        # dedup by key function
Stream.uniq(stream)                 # remove all duplicates (uses memory!)
Stream.uniq_by(stream, fun)         # unique by key function
Stream.scan(stream, acc, fun)       # like reduce but emits each accumulator
Stream.with_index(stream)           # add index: {element, index}
Stream.zip(stream_a, stream_b)      # pair elements from two streams
Stream.concat(stream_a, stream_b)   # append streams
Stream.intersperse(stream, sep)     # insert separator between elements
Stream.transform(stream, acc, reducer)  # most powerful: stateful map with halt
```

### Consuming Streams (Triggers Execution)

```elixir
Enum.to_list(stream)       # collect all into list
Enum.take(stream, n)       # collect first n
Enum.reduce(stream, ...)   # fold to single value
Enum.count(stream)         # count elements (consumes all!)
Enum.any?(stream, fun)     # short-circuits on first true
Enum.find(stream, fun)     # short-circuits on first match
Stream.run(stream)         # execute for side effects, returns :ok
```

### Practical Stream Patterns

```elixir
# Process large file without loading into memory
File.stream!("huge.csv")
|> Stream.map(&String.trim/1)
|> Stream.reject(&(&1 == ""))
|> Stream.map(&String.split(&1, ","))
|> Enum.take(1000)  # only reads ~1000 lines from disk

# Paginated API consumption
Stream.resource(
  fn -> fetch_page(1) end,
  fn
    %{data: [], next: nil} -> {:halt, nil}
    %{data: data, next: cursor} -> {data, fetch_page(cursor)}
  end,
  fn _ -> :ok end
)
|> Stream.take(500)
|> Enum.to_list()

# Batched database inserts
large_list
|> Stream.chunk_every(1000)
|> Stream.each(fn batch -> Repo.insert_all(MySchema, batch) end)
|> Stream.run()

# Fibonacci via unfold
fibs = Stream.unfold({0, 1}, fn {a, b} -> {a, {b, a + b}} end)
Enum.take(fibs, 10)  # => [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

# Sliding window
Stream.chunk_every(data, 3, 1, :discard)
# [1,2,3,4,5] => [[1,2,3], [2,3,4], [3,4,5]]

# Rate-limited processing
stream
|> Stream.each(fn item ->
  process(item)
  Process.sleep(100)  # 10 items/sec
end)
|> Stream.run()

# Stream.transform for stateful processing with early halt
Stream.transform(events, 0, fn
  _event, acc when acc >= 100 -> {:halt, acc}  # stop after 100 processed
  event, acc -> {[process(event)], acc + 1}
end)
```

### Implementing the Enumerable Protocol

Make any custom data structure work with `Enum.*` and `Stream.*` by implementing `Enumerable`:

```elixir
defmodule CircularBuffer do
  defstruct [:data, :size, :start, :count]

  def new(size) do
    %CircularBuffer{data: :array.new(size, default: nil), size: size, start: 0, count: 0}
  end

  def push(%CircularBuffer{} = buf, item) do
    pos = rem(buf.start + buf.count, buf.size)
    data = :array.set(pos, item, buf.data)
    if buf.count < buf.size do
      %{buf | data: data, count: buf.count + 1}
    else
      %{buf | data: data, start: rem(buf.start + 1, buf.size)}
    end
  end
end

# Implement Enumerable so Enum.map/filter/reduce/etc. all work
defimpl Enumerable, for: CircularBuffer do
  # Required: reduce/3 — the core iteration function
  def reduce(_buf, {:halt, acc}, _fun), do: {:halted, acc}
  def reduce(buf, {:suspend, acc}, fun), do: {:suspended, acc, &reduce(buf, &1, fun)}
  def reduce(%{count: 0}, {:cont, acc}, _fun), do: {:done, acc}
  def reduce(%{data: data, start: start, size: size, count: count} = buf, {:cont, acc}, fun) do
    item = :array.get(start, data)
    next = %{buf | start: rem(start + 1, size), count: count - 1}
    reduce(next, fun.(item, acc), fun)
  end

  # Required: count/1 — return {:ok, count} for O(1) or {:error, __MODULE__} to fall back to reduce
  def count(%{count: count}), do: {:ok, count}

  # Required: member?/2 — return {:ok, bool} for O(1) lookup or {:error, __MODULE__} to fall back
  def member?(_buf, _value), do: {:error, __MODULE__}  # no fast lookup, use reduce

  # Required: slice/1 — return {:ok, size, slicing_fun} for O(1) slicing or {:error, __MODULE__}
  def slice(_buf), do: {:error, __MODULE__}
end

# Now it works with all Enum and Stream functions:
buf = CircularBuffer.new(5) |> CircularBuffer.push(1) |> CircularBuffer.push(2) |> CircularBuffer.push(3)
Enum.to_list(buf)          # => [1, 2, 3]
Enum.map(buf, & &1 * 10)   # => [10, 20, 30]
Enum.sum(buf)              # => 6
buf |> Stream.map(& &1 * 2) |> Enum.to_list()  # => [2, 4, 6]
```

**The four Enumerable callbacks:**

| Callback | Purpose | Return for fast path | Return for fallback |
|---|---|---|---|
| `reduce/3` | Core iteration (required, always implement fully) | n/a — always implement | n/a |
| `count/1` | Element count | `{:ok, integer}` | `{:error, __MODULE__}` |
| `member?/2` | Membership test | `{:ok, boolean}` | `{:error, __MODULE__}` |
| `slice/1` | Random access | `{:ok, size, slice_fun}` | `{:error, __MODULE__}` |

When you return `{:error, __MODULE__}`, Elixir falls back to using `reduce/3` — always correct but O(n). Implement the fast path when your data structure supports it (e.g., `count/1` for structures that track size).

### The Collectable Protocol — Building Collections

`Collectable` is the inverse of `Enumerable` — it defines how to build a collection. Used by `Enum.into/2`, `for` comprehensions, and `Map.new/2`.

```elixir
# Enum.into uses Collectable to pour data into a target
Enum.into([a: 1, b: 2], %{})       # => %{a: 1, b: 2}
Enum.into(stream, File.stream!("out.txt"))  # stream into file

# for comprehensions use Collectable for :into
for {k, v} <- map, into: %{}, do: {k, v * 2}

# Implement Collectable for custom structs
defimpl Collectable, for: CircularBuffer do
  def into(buf) do
    collector_fun = fn
      buf_acc, {:cont, item} -> CircularBuffer.push(buf_acc, item)
      buf_acc, :done -> buf_acc
      _buf_acc, :halt -> :ok
    end
    {buf, collector_fun}
  end
end

# Now works with into:
buf = Enum.into([1, 2, 3], CircularBuffer.new(5))
for x <- 1..5, into: CircularBuffer.new(5), do: x
```

## Error Handling

### defexception Patterns

```elixir
# Simple — message-only exception
defmodule MyApp.NotFoundError do
  defexception message: "resource not found"
end

# With constructor — builds message from params
defmodule MyApp.MissingParamError do
  defexception [:message, plug_status: 400]  # plug_status maps to HTTP status

  def exception(key: key) do
    %__MODULE__{message: "expected key #{inspect(key)} to be present in params"}
  end
end

# WrapperError — preserving context through error boundaries
defmodule MyApp.WrapperError do
  defexception [:conn, :kind, :reason, :stack]

  @impl true
  def message(%{kind: kind, reason: reason, stack: stack}) do
    Exception.format_banner(kind, reason, stack)
  end
end
```

**Convention:** The `plug_status` field on exceptions is recognized by `Plug.Exception` protocol to map exceptions to HTTP status codes. Any struct with `plug_status` gets that status automatically via `@fallback_to_any true`.

### ok/error Tuples

```elixir
@spec fetch_user(integer()) :: {:ok, User.t()} | {:error, atom()}
def fetch_user(id) do
  case Repo.get(User, id) do
    nil -> {:error, :not_found}
    user -> {:ok, user}
  end
end

# Bang functions raise on error
def fetch_user!(id), do: Repo.get(User, id) || raise "User not found"
```

### Let It Crash

Don't catch everything — let supervisors restart:

```elixir
def handle_call({:process, data}, _from, state) do
  result = process_data(data)  # Crashes propagate to supervisor
  {:reply, {:ok, result}, state}
end
```

**When NOT to let it crash:**
- Error kernel processes (supervisors, registries, ETS owners) — use defensive code
- Processes holding connections/resources that need cleanup — use `terminate/2`
- Expected failures (user input, external APIs) — return `{:error, reason}` tuples

### Error Kernel Design

The **error kernel** is the minimal, critical part of your system that must remain operational:

```
Application
├── Supervisor (error kernel)
│   ├── Registry (error kernel - process discovery)
│   ├── ETS owner (error kernel - shared state survives worker crashes)
│   └── DynamicSupervisor
│       └── Worker processes (can crash freely)
```

**Principles:**
1. Keep the error kernel small and simple
2. Non-critical components should crash and restart with clean state
3. Supervisors and registries are part of the error kernel
4. Critical state can be stored in ETS (survives child process crashes)
5. Only use defensive programming in error kernel components

**Pattern:** Store critical state in ETS owned by a supervisor-level process. Workers read/write to ETS but their crashes don't lose data:

```elixir
defmodule MyApp.StateStore do
  use GenServer

  def start_link(_), do: GenServer.start_link(__MODULE__, nil, name: __MODULE__)

  @impl true
  def init(_) do
    :ets.new(:critical_state, [:named_table, :public, :set])
    {:ok, nil}  # Minimal state - just keeps table alive
  end
end

# Workers can crash - ETS data persists
defmodule MyApp.Worker do
  def save(key, value), do: :ets.insert(:critical_state, {key, value})
  def load(key), do: :ets.lookup(:critical_state, key)
end
```

### Exit Reason Classification

Exit reasons determine whether a supervised child restarts:

| Exit reason | `:permanent` | `:transient` | `:temporary` |
|------------|-------------|-------------|-------------|
| `:normal` | restart | no restart | no restart |
| `:shutdown` | restart | no restart | no restart |
| `{:shutdown, term}` | restart | no restart | no restart |
| Any other (abnormal) | restart | restart | no restart |

**Key insight:** `:permanent` restarts on ALL exits including `:normal`. Use `:transient` for processes that should restart only on crashes. Use `:temporary` for fire-and-forget work (tasks).

```elixir
# Transient: restarts only on abnormal exit (crashes)
use GenServer, restart: :transient

# Temporary: never restarts (one-shot work)
use GenServer, restart: :temporary
```

### Result Module Pattern

```elixir
defmodule MyApp.Result do
  @type t(ok, err) :: {:ok, ok} | {:error, err}
  @type t(ok) :: t(ok, any())

  def ok(value), do: {:ok, value}
  def error(reason), do: {:error, reason}

  def map({:ok, value}, fun), do: {:ok, fun.(value)}
  def map({:error, _} = error, _fun), do: error

  def flat_map({:ok, value}, fun), do: fun.(value)
  def flat_map({:error, _} = error, _fun), do: error

  def unwrap!({:ok, value}), do: value
  def unwrap!({:error, reason}), do: raise "Unwrap error: #{inspect(reason)}"

  def unwrap_or({:ok, value}, _default), do: value
  def unwrap_or({:error, _}, default), do: default

  # Use in pipeline
  def then({:ok, value}, fun), do: fun.(value)
  def then({:error, _} = error, _fun), do: error
end

# Usage
import MyApp.Result

fetch_user(id)
|> Result.flat_map(&validate_user/1)
|> Result.map(&format_user/1)
|> Result.unwrap_or(%{})
```

## Advanced Patterns (Req, Broadway, Absinthe patterns)

### Bidirectional Step Pipeline

Design pipelines where steps can convert between success and error states:

```elixir
@type request_step :: (Request.t() ->
  Request.t() | {Request.t(), Response.t() | Exception.t()})

@type response_step :: ({Request.t(), Response.t()} ->
  {Request.t(), Response.t() | Exception.t()})

@type error_step :: ({Request.t(), Exception.t()} ->
  {Request.t(), Response.t() | Exception.t()})

def run_pipeline(request) do
  request
  |> run_request_steps()
  |> run_adapter()
  |> run_response_steps()  # Can return exception to trigger error steps
  |> run_error_steps()     # Can return response to recover
end
```

**Key insight:** Error steps returning a response "recover" the pipeline. Response steps returning an exception trigger error handling. This enables retry, circuit-breaker, and fallback patterns.

### Option Registration Pattern

Validate options at compile/runtime to catch typos early:

```elixir
defmodule MyLib.Request do
  defstruct registered_options: MapSet.new()

  def register_options(request, options) when is_list(options) do
    update_in(request.registered_options, &MapSet.union(&1, MapSet.new(options)))
  end

  def put_option(request, key, value) do
    if key in request.registered_options do
      put_in(request.options[key], value)
    else
      raise ArgumentError, "unknown option #{inspect(key)}"
    end
  end
end

# Plugin registers its options
def attach(request) do
  request
  |> Request.register_options([:retry, :retry_delay, :max_retries])
end
```

### Private Data Namespace

Reserve a map for library/framework extensions without polluting the public API:

```elixir
defstruct data: nil,
          metadata: %{},
          private: %{}  # Reserved for libraries

def put_private(%__MODULE__{} = struct, key, value) when is_atom(key) do
  put_in(struct.private[key], value)
end

def get_private(%__MODULE__{} = struct, key, default \\ nil) do
  Map.get(struct.private, key, default)
end
```

### Error Delegation Pattern

Wrap underlying library errors while leveraging their formatting:

```elixir
defmodule MyApp.TransportError do
  defexception [:reason]

  @impl true
  def message(%__MODULE__{reason: reason}) do
    # Delegate to underlying library's error formatting
    Mint.TransportError.message(%Mint.TransportError{reason: reason})
  end
end
```

### Atomics for Lock-Free Rate Limiting

Use `:atomics` for high-throughput counters without GenServer bottleneck:

```elixir
defmodule RateLimiter do
  use GenServer

  def init(opts) do
    counter = :atomics.new(1, [])
    :atomics.put(counter, 1, opts[:allowed])
    schedule_reset(opts[:interval])
    {:ok, %{counter: counter, allowed: opts[:allowed], interval: opts[:interval]}}
  end

  # Lock-free decrement - no message passing
  def rate_limit(limiter, amount) do
    %{counter: counter} = :sys.get_state(limiter)
    :atomics.sub_get(counter, 1, amount)  # Returns new value
  end

  def handle_info(:reset, state) do
    :atomics.put(state.counter, 1, state.allowed)
    schedule_reset(state.interval)
    {:noreply, state}
  end
end
```

### Persistent Term with Namespacing

Use `{Module, key}` tuples for efficient global config:

```elixir
defmodule MyApp.Config do
  def put(key, value) do
    :persistent_term.put({__MODULE__, key}, value)
  end

  def get(key, default \\ nil) do
    :persistent_term.get({__MODULE__, key}, default)
  end

  # Intentional: Don't delete from persistent_term
  # Memory overhead is negligible for named resources
  # Keeping entries allows process restarts without metadata loss
  def delete(_key), do: :ok
end
```

### Status as Tagged Exception Tuple

Capture full exception context for rich error handling:

```elixir
@type status ::
  :ok
  | {:failed, reason :: term}
  | {:throw | :error | :exit, term, Exception.stacktrace()}

def process_with_status(message, fun) do
  result = fun.(message)
  %{message | status: :ok, data: result}
rescue
  e -> %{message | status: {:error, e, __STACKTRACE__}}
catch
  :throw, value -> %{message | status: {:throw, value, __STACKTRACE__}}
  :exit, reason -> %{message | status: {:exit, reason, __STACKTRACE__}}
end
```

### Collectable with Streaming Hash

Compute checksums while streaming without buffering:

```elixir
def collect_with_hash(into, algorithm) do
  hash_state = :crypto.hash_init(algorithm)

  collector_fun = fn
    {acc, hash}, {:cont, chunk} ->
      {[chunk | acc], :crypto.hash_update(hash, chunk)}
    {acc, hash}, :done ->
      {Enum.reverse(acc), :crypto.hash_final(hash)}
    _, :halt ->
      :ok
  end

  {{[], hash_state}, collector_fun}
end

# Usage
{body, checksum} =
  stream
  |> Enum.into(collect_with_hash([], :sha256))
```

### CallerAcknowledger for Testing Async Flows

Test acknowledgment-based systems by sending messages back to test process:

```elixir
defmodule CallerAcknowledger do
  @behaviour MyApp.Acknowledger

  def init({pid, ref}, _data) when is_pid(pid) do
    {__MODULE__, {pid, ref}, nil}
  end

  @impl true
  def ack({pid, ref}, successful, failed) do
    send(pid, {:ack, ref, successful, failed})
  end
end

# In test
test "processes and acknowledges messages" do
  ref = make_ref()
  ack = CallerAcknowledger.init({self(), ref}, nil)

  MyApp.Pipeline.push_message(%Message{data: "test", acknowledger: ack})

  assert_receive {:ack, ^ref, [%{data: "test"}], []}, 5000
end
```

### Coordinated Shutdown (Terminator Pattern)

Three-phase shutdown to prevent message loss:

```elixir
defmodule Terminator do
  use GenServer

  def handle_info(:terminate, state) do
    # Phase 1: Signal consumers to stop accepting new work
    for name <- state.consumers do
      if pid = GenServer.whereis(name), do: send(pid, :will_terminate)
    end

    # Phase 2: Drain producer buffers
    for name <- state.producers do
      if pid = GenServer.whereis(name), do: Producer.drain(pid)
    end

    # Phase 3: Wait for completion with monitoring
    refs = Enum.map(state.producers, &Process.monitor(GenServer.whereis(&1)))
    await_completion(refs)

    {:stop, :normal, state}
  end
end
```

### Telemetry Integration

The `:telemetry` library is the standard for instrumentation in Elixir. Two patterns:

```elixir
# Pattern 1: :telemetry.span — wraps an operation with start/stop/exception events
# The callback MUST return {result, metadata} tuple
:telemetry.span([:my_app, :repo, :query], %{source: table}, fn ->
  result = execute_query(query)
  {result, %{source: table, result: result}}  # {return_value, stop_metadata}
end)

# Pattern 2: Manual start/stop — when you need more control
start = System.monotonic_time()
:telemetry.execute([:my_app, :request, :start], %{system_time: System.system_time()}, metadata)

try do
  result = do_work()
  duration = System.monotonic_time() - start
  :telemetry.execute([:my_app, :request, :stop], %{duration: duration}, metadata)
  result
rescue
  e ->
    duration = System.monotonic_time() - start
    :telemetry.execute([:my_app, :request, :exception], %{duration: duration}, metadata)
    reraise e, __STACKTRACE__
end
```

**Naming convention:** `[:app, :component, :action]` — action is `:start`/`:stop`/`:exception`. Measurements always include `%{duration: ...}` for stop/exception, `%{system_time: ...}` for start.

### Plug halt/1 Semantics

`halt/1` sets a flag — it does NOT stop execution of the current plug:

```elixir
# halt/1 just sets halted: true on the conn
def halt(%Conn{} = conn), do: %{conn | halted: true}

# The compiled pipeline checks the flag BETWEEN plugs, not during
# Your plug must still return the conn after halting
def call(conn, _opts) do
  conn
  |> put_status(403)
  |> send_resp(403, "Forbidden")
  |> halt()   # Sets flag; subsequent plugs in pipeline are skipped
  # Code AFTER halt() in this function still executes!
end
```

### Task.async_stream Patterns

```elixir
# Fire-and-forget with Stream.run() — for side effects only
source_files
|> Task.async_stream(&check_file/1,
  max_concurrency: System.schedulers_online(),
  timeout: :infinity,
  ordered: false)        # Don't wait for order — faster for independent work
|> Stream.run()          # Consume stream, discard results

# Collecting results — unwrap the {:ok, result} tuples
results =
  items
  |> Task.async_stream(&process/1, timeout: :infinity, ordered: false)
  |> Enum.map(fn {:ok, result} -> result end)

# Handling timeouts — use on_timeout: :kill_task
filenames
|> Task.async_stream(&parse/1, timeout: 5_000, on_timeout: :kill_task)
|> Stream.zip(filenames)
|> Enum.map(fn
  {{:exit, :timeout}, filename} -> {:error, {:timeout, filename}}
  {{:ok, result}, _filename} -> {:ok, result}
end)
```

### AST Traversal with Accumulators

`Macro.prewalk/3` and `Macro.postwalk/3` use the `{node, acc}` return pattern:

```elixir
# Collect all function calls in an AST
def find_calls(ast) do
  {_ast, calls} = Macro.prewalk(ast, [], fn
    # Return {node, acc} — node controls traversal, acc accumulates results
    {func, _meta, args} = node, acc when is_atom(func) and is_list(args) ->
      {node, [{func, length(args)} | acc]}

    # Return {nil, acc} to PRUNE a subtree from further traversal
    {:@, _, _}, acc ->
      {nil, acc}    # Skip module attribute subtrees

    node, acc ->
      {node, acc}   # Pass through unchanged
  end)
  Enum.reverse(calls)
end

# Nested prewalk — search within a matched node's children
defp walk({:@, _, [{:spec, _, args}]} = node, issues) do
  case Macro.prewalk(args, [], &find_structs/2) do
    {_ast, []} -> {node, issues}
    {_ast, structs} -> {node, [build_issue(structs) | issues]}
  end
end
```

### Changeset Semantics — Validations vs Constraints

Changesets are **eagerly evaluated data structures**, not lazy. Each function in the pipeline executes immediately and returns a new `%Changeset{}`:

```elixir
user
|> cast(params, [:email, :name])           # Executes NOW — casts and filters params
|> validate_required([:email])             # Executes NOW — checks presence, adds error
|> validate_format(:email, ~r/@/)          # Executes NOW — checks format
|> unique_constraint(:email)               # DEFERRED — just registers a constraint handler
|> Repo.insert()                           # DB hit; if unique index fails, constraint fires
```

**Critical distinctions:**
- **Validations** (`validate_*`) run immediately when called. If any fail, `valid?` is `false`
- **Constraints** (`*_constraint`) are deferred — they register handlers that fire ONLY if the DB returns an error
- If `valid?` is `false`, Repo never hits the DB — constraints never execute
- `unique_constraint` is NOT a validation — it's a post-insert DB check that converts a DB error into a changeset error

**Ecto.Multi** validates static changesets BEFORE starting the transaction:
```elixir
Multi.new()
|> Multi.insert(:user, invalid_changeset)   # valid?: false
|> Repo.transaction()                        # Returns {:error, :user, changeset, %{}}
# Transaction never started — the invalid changeset was caught pre-transaction
# But function-based operations skip this check (changeset doesn't exist yet)
```

### Phase Pipeline Pattern

Used by Absinthe — each phase returns a tagged tuple controlling flow:

```elixir
@type result ::
  {:ok, data}                           # Continue to next phase
  | {:jump, data, destination_phase}    # Skip ahead to a specific phase
  | {:insert, data, extra_phases}       # Insert phases before remaining
  | {:replace, data, new_pipeline}      # Replace remaining pipeline
  | {:error, reason}                    # Abort

def run_phases([], input, done), do: {:ok, input, done}
def run_phases([phase | todo], input, done) do
  case phase.run(input) do
    {:ok, result}      -> run_phases(todo, result, [phase | done])
    {:jump, result, dest} -> run_phases(from(todo, dest), result, [phase | done])
    {:insert, result, extra} -> run_phases(extra ++ todo, result, [phase | done])
    {:error, reason}   -> {:error, reason, [phase | done]}
  end
end
```

### Pluggable Engine Pattern (from Oban)

```elixir
defmodule MyApp.Engine do
  @callback init(conf :: Config.t(), opts :: keyword()) :: {:ok, meta :: map()}
  @callback insert_job(conf :: Config.t(), changeset :: Changeset.t(), opts :: keyword()) ::
    {:ok, Job.t()} | {:error, term()}
  @callback fetch_jobs(conf :: Config.t(), meta :: map(), opts :: keyword()) ::
    {:ok, {meta :: map(), [Job.t()]}}
  @callback complete_job(conf :: Config.t(), job :: Job.t()) :: :ok
  @callback error_job(conf :: Config.t(), job :: Job.t(), reason :: term()) :: :ok

  @optional_callbacks [stage_jobs: 3, prune_jobs: 3]

  def insert_job(conf, changeset, opts), do: conf.engine.insert_job(conf, changeset, opts)

  def stage_jobs(conf, meta, opts) do
    if function_exported?(conf.engine, :stage_jobs, 3) do
      conf.engine.stage_jobs(conf, meta, opts)
    else
      Basic.stage_jobs(conf, meta, opts)
    end
  end
end
```

### Validation with Smart Suggestions

```elixir
defmodule MyApp.Validation do
  def validate_schema(opts, schema) when is_list(opts) and is_map(schema) do
    Enum.reduce_while(opts, :ok, fn {key, value}, :ok ->
      case Map.fetch(schema, key) do
        {:ok, type} ->
          case validate_type(key, value, type) do
            :ok -> {:cont, :ok}
            error -> {:halt, error}
          end
        :error ->
          suggestion = suggest_key(key, Map.keys(schema))
          {:halt, {:error, "unknown option #{inspect(key)}" <> suggestion}}
      end
    end)
  end

  defp suggest_key(key, known_keys) do
    key_string = to_string(key)
    known_keys
    |> Enum.map(&{&1, String.jaro_distance(key_string, to_string(&1))})
    |> Enum.filter(fn {_, d} -> d > 0.7 end)
    |> Enum.max_by(fn {_, d} -> d end, fn -> nil end)
    |> case do
      {suggestion, _} -> ", did you mean #{inspect(suggestion)}?"
      nil -> ""
    end
  end

  defp validate_type(_key, value, :pos_integer) when is_integer(value) and value > 0, do: :ok
  defp validate_type(key, value, :pos_integer), do: {:error, "expected #{inspect(key)} to be positive integer"}
  defp validate_type(_key, value, :atom) when is_atom(value), do: :ok
  defp validate_type(_key, value, {:in, allowed}) when value in allowed, do: :ok
end
```

### Exponential Backoff with Jitter

```elixir
defmodule MyApp.Backoff do
  def exponential(attempt, opts \\ []) do
    mult = Keyword.get(opts, :mult, 1)
    base_pad = Keyword.get(opts, :base, 15)
    max_pow = Keyword.get(opts, :max_pow, 10)
    base_pad + mult * :math.pow(2, min(attempt, max_pow)) |> round()
  end

  def jitter(value, opts \\ []) do
    mode = Keyword.get(opts, :mode, :both)
    mult = Keyword.get(opts, :mult, 0.1)
    diff = :rand.uniform() * mult * value |> round()
    case mode do
      :inc -> value + diff
      :dec -> value - diff
      :both -> if(:rand.uniform() > 0.5, do: value + diff, else: value - diff)
    end
  end

  def with_retry(fun, opts \\ []) do
    max_retries = Keyword.get(opts, :max_retries, :infinity)
    attempt = Keyword.get(opts, :attempt, 1)
    try do
      fun.()
    rescue
      e in [DBConnection.ConnectionError, Postgrex.Error] ->
        if max_retries == :infinity or attempt < max_retries do
          Process.sleep(exponential(attempt) |> jitter())
          with_retry(fun, Keyword.put(opts, :attempt, attempt + 1))
        else
          reraise e, __STACKTRACE__
        end
    end
  end
end
```

### Message with Acknowledger Pattern (from Broadway)

Decouple message processing from acknowledgment strategy:

```elixir
defmodule MyApp.Message do
  @enforce_keys [:data, :acknowledger]
  defstruct [
    :data,
    :metadata,
    :acknowledger,  # {module, ack_ref, ack_data}
    status: :ok,
    batcher: :default
  ]

  @type acknowledger :: {module, ack_ref :: term, ack_data :: term}

  def ack_immediately(%__MODULE__{} = message) do
    {module, ack_ref, ack_data} = message.acknowledger
    module.ack(ack_ref, [message], [])
    %{message | acknowledger: MyApp.NoopAcknowledger.init()}
  end

  def failed(%__MODULE__{} = message, reason) do
    %{message | status: {:failed, reason}}
  end
end

defmodule MyApp.Acknowledger do
  @callback ack(ack_ref :: term, successful :: [Message.t()], failed :: [Message.t()]) :: :ok
  @callback configure(ack_ref :: term, ack_data :: term, opts :: keyword) :: {:ok, ack_data}
  @optional_callbacks configure: 3
end

defmodule MyApp.NoopAcknowledger do
  @behaviour MyApp.Acknowledger

  def init, do: {__MODULE__, nil, nil}

  @impl true
  def ack(_ref, _successful, _failed), do: :ok
end

defmodule MyApp.SQSAcknowledger do
  @behaviour MyApp.Acknowledger

  def init(queue_url, receipt_handle) do
    {__MODULE__, queue_url, receipt_handle}
  end

  @impl true
  def ack(queue_url, successful, _failed) do
    entries = Enum.map(successful, fn msg ->
      {_, _, receipt_handle} = msg.acknowledger
      %{id: msg.metadata.id, receipt_handle: receipt_handle}
    end)

    ExAws.SQS.delete_message_batch(queue_url, entries)
    |> ExAws.request()

    :ok
  end
end
```

### Subscriber Resubscription Pattern

Auto-reconnect to producers after transient failures:

```elixir
defmodule MyApp.Subscriber do
  use GenStage

  def start_link(opts) do
    GenStage.start_link(__MODULE__, opts)
  end

  @impl true
  def init(opts) do
    producers = Keyword.fetch!(opts, :producers)
    resubscribe_delay = Keyword.get(opts, :resubscribe_delay, 1000)

    state = %{
      producers: producers,
      resubscribe_delay: resubscribe_delay,
      subscriptions: %{}
    }

    # Subscribe to all producers
    {:consumer, state, subscribe_to: producers}
  end

  @impl true
  def handle_subscribe(:producer, _opts, {pid, _ref} = from, state) do
    subscriptions = Map.put(state.subscriptions, pid, from)
    {:automatic, %{state | subscriptions: subscriptions}}
  end

  @impl true
  def handle_cancel({:down, _reason}, {pid, _ref}, state) do
    # Producer crashed - schedule resubscription
    Process.send_after(self(), {:resubscribe, pid}, state.resubscribe_delay)
    subscriptions = Map.delete(state.subscriptions, pid)
    {:noreply, [], %{state | subscriptions: subscriptions}}
  end

  def handle_cancel(_reason, {pid, _ref}, state) do
    # Normal cancellation - don't resubscribe
    subscriptions = Map.delete(state.subscriptions, pid)
    {:noreply, [], %{state | subscriptions: subscriptions}}
  end

  @impl true
  def handle_info({:resubscribe, producer}, state) do
    case GenStage.sync_subscribe(self(), to: producer) do
      {:ok, _ref} ->
        {:noreply, [], state}
      {:error, _reason} ->
        # Retry with backoff
        Process.send_after(self(), {:resubscribe, producer}, state.resubscribe_delay * 2)
        {:noreply, [], state}
    end
  end

  @impl true
  def handle_events(events, _from, state) do
    # Process events
    Enum.each(events, &process_event/1)
    {:noreply, [], state}
  end

  defp process_event(event), do: IO.inspect(event, label: "Event")
end
```

### Dynamic Pool Creation with Deduplication

Create connection pools on-demand with MD5 hash deduplication:

```elixir
defmodule MyApp.PoolManager do
  @moduledoc """
  Creates connection pools dynamically based on options.
  Same options = same pool (deduplicated via MD5 hash).
  """

  def get_or_create_pool(options) do
    # Generate unique pool name from options hash
    key = :erlang.md5(:erlang.term_to_binary(options))
    name = Module.concat(__MODULE__, "Pool_" <> Base.encode16(key, case: :lower))

    case DynamicSupervisor.start_child(
      MyApp.PoolSupervisor,
      {NimblePool, worker: {MyApp.PoolWorker, options}, name: name}
    ) do
      {:ok, _pid} -> name
      {:error, {:already_started, _pid}} -> name  # Deduplication: reuse existing
    end
  end
end

# In Application supervisor
children = [
  {DynamicSupervisor, strategy: :one_for_one, name: MyApp.PoolSupervisor}
]
```

### Batch Info with Trigger Metadata

Include why a batch was triggered for context-aware handling:

```elixir
defmodule MyApp.BatchInfo do
  @type t :: %__MODULE__{
    batcher: atom,
    batch_key: term,
    partition: non_neg_integer | nil,
    size: pos_integer,
    trigger: :size | :timeout | :flush
  }

  defstruct [:batcher, :batch_key, :partition, :size, :trigger]
end

defmodule MyApp.BatchHandler do
  def handle_batch(batcher, messages, batch_info, state) do
    case batch_info.trigger do
      :timeout ->
        # Partial batch due to timeout - might want different handling
        Logger.info("Processing partial batch of #{batch_info.size} (timeout)")
        process_messages(messages, async: false)

      :size ->
        # Full batch - can process more aggressively
        Logger.info("Processing full batch of #{batch_info.size}")
        process_messages(messages, async: true)

      :flush ->
        # Urgent flush - process immediately
        Logger.info("Flush-triggered batch of #{batch_info.size}")
        process_messages(messages, async: false, priority: :high)
    end

    {:ok, messages, state}
  end
end
```

### Telemetry Integration Pattern (extended)

```elixir
defmodule MyApp.Telemetry do
  def with_span(event_prefix, meta, fun) do
    start_time = System.monotonic_time()
    start_meta = Map.put(meta, :system_time, System.system_time())
    :telemetry.execute(event_prefix ++ [:start], %{system_time: start_meta.system_time}, start_meta)

    try do
      result = fun.()
      duration = System.monotonic_time() - start_time
      :telemetry.execute(event_prefix ++ [:stop], %{duration: duration}, Map.put(meta, :result, result))
      result
    rescue
      exception ->
        duration = System.monotonic_time() - start_time
        :telemetry.execute(event_prefix ++ [:exception], %{duration: duration},
          Map.merge(meta, %{kind: :error, reason: exception, stacktrace: __STACKTRACE__}))
        reraise exception, __STACKTRACE__
    end
  end

  def attach_default_logger(opts \\ []) do
    events = [[:my_app, :job, :start], [:my_app, :job, :stop], [:my_app, :job, :exception]]
    :telemetry.attach_many("my-app-logger", events, &handle_event/4, opts)
  end

  defp handle_event([:my_app, :job, :start], _, meta, _), do: Logger.info("[Job] Starting #{meta.worker}")
  defp handle_event([:my_app, :job, :stop], m, meta, _), do: Logger.info("[Job] Completed #{meta.worker} in #{div(m.duration, 1_000_000)}ms")
  defp handle_event([:my_app, :job, :exception], _, meta, _), do: Logger.error("[Job] Failed #{meta.worker}: #{inspect(meta.reason)}")
end
```
