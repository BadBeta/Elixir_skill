---
name: elixir
version: "0.11"
description: Elixir functional programming, OTP, and Ecto — pattern matching, pipelines, Enum/Stream, with-chains, ok/error tuples, multi-clause functions, GenServer, gen_statem, supervision, ETS, Ecto schemas/changesets/queries/migrations, configuration, telemetry, HTTP clients, and type system. ALWAYS use this skill when writing Elixir to avoid imperative anti-patterns. ALWAYS use when designing OTP supervision trees, state machines, or distributed systems. ALWAYS consult the Architecture and OTP sections when planning new Elixir projects, refactoring existing ones, or making structural decisions.
---

<!-- WORKAROUND: Uses fullwidth exclamation mark (！U+FF01) before closing backticks
     to bypass Claude Code parser bug that interprets `func！` as bash command.
     TODO: Revert ！to ! once bug is fixed. -->

# Elixir Programming Skill

## Navigation — Supporting Files

| File | Contents |
|------|----------|
| [data-structures.md](data-structures.md) | Lists, maps, tuples, keywords, MapSet, structs (constructors, pipelines, protocols, nesting), embedded schemas, binary patterns |
| [quick-references.md](quick-references.md) | Enum, Map, Keyword, List, String, Regex, File/Path, URI, Date/Time, IO, Access, Process, Range, Agent quick refs, Erlang stdlib (21 modules), JSON |
| [language-patterns.md](language-patterns.md) | Extended pattern matching, guards, comprehensions, pipelines, behaviours, protocols, streams, error handling, advanced patterns |
| [code-style.md](code-style.md) | .formatter.exs config, migration options, Credo checks catalog, readable code patterns (pipelines, guards, naming, conditionals), BAD/GOOD pairs |
| [documentation.md](documentation.md) | @moduledoc/@doc patterns, @spec/@type/@typedoc, @since/@deprecated, doctests (multi-line, exceptions, ellipsis), ExDoc config, cross-references |
| [type-system.md](type-system.md) | Set-theoretic types (1.17-1.20), notation, atoms/tuples/maps/functions as types, dynamic(), inference, compiler warnings, @spec vs gradual types, roadmap |
| [architecture-reference.md](architecture-reference.md) | Architecture layouts, Phoenix contexts, layered/pipeline architecture, production patterns, anti-patterns catalog |
| [debugging-profiling.md](debugging-profiling.md) | IO.inspect, dbg, IEx.pry, break!, Rexbug, system introspection, Logger, fprof/eprof/cprof/tprof, Benchee, memory/VM/scheduler profiling |
| [ecto-reference.md](ecto-reference.md) | Ecto field types, changeset API, query operations, migration options, Repo API, Multi |
| [ecto-examples.md](ecto-examples.md) | Complete Ecto examples: schemas, queries, migrations, Multi, soft delete, multi-tenancy |
| [otp-reference.md](otp-reference.md) | OTP callback signatures, ETS operations, process debugging, release management |
| [otp-examples.md](otp-examples.md) | Complete OTP examples: rate limiter, state machine, worker pool, cache, circuit breaker, distribution |
| [otp-advanced.md](otp-advanced.md) | GenStage, Flow, Broadway, hot code upgrades, production debugging |
| [testing-reference.md](testing-reference.md) | Testing quick refs: assertions, Mox API, LiveView/Channel helpers, StreamData |
| [testing-examples.md](testing-examples.md) | Complete testing examples: infrastructure, Mox, channels, LiveView, property testing, Oban |
| [eventsourcing-reference.md](eventsourcing-reference.md) | Commanded API, router DSL, middleware, projections, process managers |
| [eventsourcing-examples.md](eventsourcing-examples.md) | Complete event sourcing examples: aggregates, projectors, process managers, testing |
| [production.md](production.md) | Production Phoenix patterns, Edge/IoT patterns, Oban, telemetry, HTTP clients |

## The Elixir Way

### Rules for Writing Elixir (LLM)

1. **NEVER use if/else for structural dispatch.** Use multi-clause functions with pattern matching. Only use `if` for simple single-condition boolean checks with no alternative branches.
2. **NEVER use try/rescue for expected failures.** Use `{:ok, _}/{:error, _}` tuples with `case` or `with`. Reserve try/rescue for truly unexpected exceptions at system boundaries.
3. **NEVER write imperative loops.** Elixir has no for/while loops with mutable state. Use `Enum.map/2`, `Enum.filter/2`, `Enum.reduce/3`, or `for` comprehensions.
4. **ALWAYS design functions for pipe-ability.** The subject (primary data) goes as the first argument. Return the transformed data.
5. **PREFER pattern matching in function heads** over `case` in the body when dispatching on argument shape or type.
6. **PREFER Enum functions over manual recursion** for collection processing. Use recursion only for early termination, tree/graph traversal, or complex multi-accumulator state. Use `Stream` for infinite/lazy sequences.
7. **NEVER reassign to accumulate.** Rebinding a variable inside `Enum.each/2` does NOT mutate the outer variable. Use `Enum.reduce/3` or `Enum.map/2` to collect results.
8. **USE `with` for chaining** 2+ operations that return `{:ok, _}/{:error, _}`. Don't nest `case` statements.
9. **USE guard clauses** to constrain function heads rather than validating inside the body with if/else.
10. **BUILD strings with IO lists**, not repeated `<>` concatenation. Collect `[part1, ", ", part2]` and call `IO.iodata_to_binary/1` once, or pass IO lists directly to I/O functions.
11. **NEVER check for nil with if.** Pattern match on the value's presence: use separate clauses for `nil` and non-nil, or match `%{key: value}` to assert key exists.
12. **PREFER atoms over strings for internal identifiers.** Atoms are interned (fast comparison). Use strings only for user/external data.
13. **PREFER `Map.new/2` and `Enum.into/2`** over `Enum.reduce/3` when building maps or other collectables from lists.
14. **USE `Enum.reduce_while/3`** for early-exit accumulation instead of throwing or using flags. Return `{:cont, acc}` or `{:halt, acc}`.
15. **USE `map`, `reduce`, `filter`, or `for` to collect results.** Choose the right function for the transformation needed.
16. **ALWAYS use `@impl true`** on every behaviour callback implementation. It catches typos and missing callbacks at compile time.
17. **ALWAYS use `%{struct | key: val}`** for struct updates, NEVER `Map.put(struct, key, value)`. The update syntax raises on unknown keys, providing compile-time safety.
18. **ALWAYS distinguish between in-process validation and deferred external checks.** Validate data shape and rules immediately; defer uniqueness and referential checks to the database or external system. (In Ecto: `validate_*` runs immediately, `*_constraint` runs after DB write.)
19. **PREFER `Task.async_stream` with `ordered: false`** for parallel independent work. Use `Stream.run()` when consuming only for side effects.
20. **ALWAYS put `@derive` before `defstruct`/`schema`.** NEVER implement a protocol `for: Map` expecting it to match structs — structs dispatch separately.

### Thinking Functionally

**Data In, Data Out** — Every function takes data and returns new data. No side effects, no mutation. Design each function as a pure transformation:

```elixir
# Imperative thinking: "modify the order"
# Functional thinking: "create a new order with these changes"
def apply_discount(%Order{} = order, percentage) do
  discounted = order.total * (1 - percentage / 100)
  %{order | total: discounted, discount_applied: true}
end
```

**Pipelines as Data Flow** — Read left-to-right, top-to-bottom. Each step transforms data further:

```elixir
raw_input                          # Start: raw string
|> String.trim()                   # Remove whitespace
|> String.split("\n")              # Split into lines
|> Enum.reject(&(&1 == ""))       # Remove blanks
|> Enum.map(&parse_line/1)        # Parse each line
|> Enum.group_by(& &1.category)   # Group by category
|> Map.new(fn {k, v} -> {k, length(v)} end)  # Count per category
```

**Compose Small Functions** — Each function does one job, is testable alone, and is reusable:

```elixir
# BAD - one big function doing everything
def process(data) do
  # 50 lines of mixed concerns
end

# GOOD - small, focused, composable functions
def process(data) do
  data
  |> validate()
  |> normalize()
  |> enrich()
  |> persist()
end
```

**Tagged Tuples as Types** — Elixir uses tagged tuples where other languages use enums/unions:

```elixir
{:ok, value}             # Success
{:error, :not_found}     # Typed failure
{:error, %Changeset{}}   # Rich failure

# Match exhaustively
case result do
  {:ok, value} -> use(value)
  {:error, :not_found} -> default()
  {:error, reason} -> log_and_fail(reason)
end
```

### Multi-Clause Functions Over Conditionals

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

### Pattern Matching & Guards (Key Patterns)

```elixir
# Function arguments — struct + field extraction
def current_path(%Plug.Conn{query_string: ""} = conn), do: conn.request_path
def current_path(%Plug.Conn{query_string: q} = conn), do: "#{conn.request_path}?#{q}"

# Tagged tuples
def handle({:ok, value}), do: process(value)
def handle({:error, reason}), do: log_error(reason)

# Lists — head/tail
def sum([head | tail], acc), do: sum(tail, acc + head)
def sum([], acc), do: acc

# Pin operator — match against existing variable
target_id = 42
Enum.find(users, fn %{id: ^target_id} -> true; _ -> false end)

# Guards — type dispatch
def process(x) when is_integer(x), do: x * 2
def process(x) when is_binary(x), do: String.length(x)
def process(x) when is_list(x), do: length(x)

# Guard with in (ranges, lists)
def weekday?(day) when day in [:mon, :tue, :wed, :thu, :fri], do: true
def weekday?(_), do: false

# Multiple when clauses = OR (more readable than `or`)
defp escape_char(char)
     when char in 0x2061..0x2064
     when char in [0x061C, 0x200E, 0x200F]
     when char in 0x202A..0x202E do
  # matches if ANY when clause is true
end

# Guard on module attribute (inlined at compile time)
@unsent [:unset, :set]
def send_resp(%Conn{state: state}, _, _) when state not in @unsent, do: raise AlreadySentError

# Custom guards — reusable guard macros
defguard is_positive(n) when is_number(n) and n > 0
defguard is_non_empty_string(s) when is_binary(s) and byte_size(s) > 0
defguardp is_valid_age(age) when is_integer(age) and age in 0..150

# Use custom guards in function heads
import MyApp.Guards
def create(%{age: age, name: name})
    when is_valid_age(age) and is_non_empty_string(name) do
  {:ok, %User{age: age, name: name}}
end
```

**Allowed in guards:** `==`, `!=`, `===`, `!==`, `<`, `>`, `<=`, `>=`, `and`, `or`, `not`, `in`, `+`, `-`, `*`, `/`, `abs`, `div`, `rem`, `round`, `trunc`, `is_atom`, `is_binary`, `is_integer`, `is_float`, `is_list`, `is_map`, `is_tuple`, `is_nil`, `is_boolean`, `is_number`, `is_pid`, `is_struct`, `is_function`, `byte_size`, `elem`, `hd`, `tl`, `length`, `map_size`, `tuple_size`, `is_map_key`

**NOT allowed:** Custom function calls, `String.length/1`, `Enum.*` — only the built-in list above. Also allowed: `Bitwise.&&&`, `Bitwise.|||`, `Bitwise.bsl`, `Bitwise.bsr` (import `Bitwise` first).

```elixir
# Guards in case (not just function heads)
case value do
  n when is_integer(n) and n > 0 -> :positive
  n when is_integer(n) -> :non_positive
  f when is_float(f) -> :float
  s when is_binary(s) -> :string
  _ -> :other
end

# Nested destructuring — extract from nested maps/structs
def get_user_city(%{address: %{city: city}}), do: {:ok, city}
def get_user_city(_), do: {:error, :no_city}

# match?/2 — boolean pattern check without extracting values
Enum.filter(items, &match?({:ok, _}, &1))
if match?({:error, _}, result), do: log_error(result)
match?({in_type, _} when in_type in [:in, :one_of], config[:type])

# case — pattern match on single value
case fetch_user(id) do
  {:ok, user} -> process(user)
  {:error, :not_found} -> create_user(id)
  {:error, reason} -> log_error(reason)
end

# cond — multiple boolean conditions (no else-if chains!)
cond do
  x > 10 -> :large
  x > 5 -> :medium
  x > 0 -> :small
  true -> :zero_or_negative  # Always end with true ->
end
```

#### Assertive Pattern Matching

Let functions crash on unexpected input — makes bugs visible immediately:

```elixir
# BAD - hides malformed input, returns wrong value
def get_value(string, key) do
  parts = String.split(string, "&")
  Enum.find_value(parts, fn pair ->
    key_value = String.split(pair, "=")
    Enum.at(key_value, 0) == key && Enum.at(key_value, 1)
  end)
end

# GOOD - pattern match asserts structure, crashes on malformed input
def get_value(string, key) do
  Enum.find_value(String.split(string, "&"), fn pair ->
    [k, value] = String.split(pair, "=")
    k == key && value
  end)
end

# GOOD - extract only what guard needs, destructure in body
def drive(%User{age: age} = user) when age >= 18 do
  %User{name: name, license: license} = user
  "#{name} with license #{license} can drive"
end
```

#### Imperative to Elixir Translation

**Collection Operations:**

| Imperative | Elixir |
|---|---|
| `for (x of list) result.push(f(x))` | `Enum.map(list, &f/1)` |
| `for (x of list) if (p(x)) result.push(x)` | `Enum.filter(list, &p/1)` |
| `let acc = init; for (...) acc = f(acc, x)` | `Enum.reduce(list, init, fn x, acc -> ... end)` |
| `list.find(x => p(x))` | `Enum.find(list, &p/1)` |
| `list.some(x => p(x))` | `Enum.any?(list, &p/1)` |
| `list.flatMap(x => f(x))` | `Enum.flat_map(list, &f/1)` |
| `[...set]` (deduplicate) | `Enum.uniq(list)` or `Enum.uniq_by(list, &key/1)` |
| `list.sort((a,b) => a.name - b.name)` | `Enum.sort_by(list, & &1.name)` |
| `Object.groupBy(list, x => x.type)` | `Enum.group_by(list, & &1.type)` |
| `_.countBy(list, f)` | `Enum.frequencies_by(list, &f/1)` |
| `_.chunk(list, 3)` | `Enum.chunk_every(list, 3)` |
| `_.partition(list, pred)` | `Enum.split_with(list, &pred/1)` |
| `list.join(", ")` | `Enum.join(list, ", ")` |
| `Math.max(...list)` | `Enum.max(list)` |
| `list.reduce((a, b) => a + b, 0)` | `Enum.sum(list)` |

**Control Flow:**

| Imperative | Elixir |
|---|---|
| `if/else if/else` | Multi-clause function with pattern matching |
| `switch (x.type)` | `case x.type do ... end` or multi-clause function |
| `if (x != null && x.active)` | `def f(%{active: true} = x)` (pattern match) |
| `try { risky() } catch(e) { ... }` | `case risky() do {:ok, v} -> v; {:error, _} -> fallback end` |
| `for (...) { if (done) break }` | `Enum.reduce_while(list, acc, fn x, acc -> {:cont/:halt, acc} end)` |
| `while (cond) { ... }` | Recursive function with guard or `Stream.iterate/2` |
| `early return` | Pattern match + multiple function clauses |

**Data Mutation:**

| Imperative | Elixir |
|---|---|
| `obj.key = value` | `%{map \| key: value}` or `Map.put(map, key, value)` |
| `obj.a.b.c = value` | `put_in(obj, [:a, :b, :c], value)` |
| `obj.count++` | `update_in(obj, [:count], & &1 + 1)` |
| `delete obj.key` | `Map.delete(map, key)` |
| `list.push(item)` | `[item \| list]` (prepend — O(1)) |
| `list.pop()` | `[head \| tail] = list` (pattern match) |
| `set.add(item)` | `MapSet.put(set, item)` |
| `str += chunk` in loop | IO list: `[chunk \| acc]`, then `IO.iodata_to_binary/1` |
| `"Hello " + name + "!"` | `"Hello #{name}!"` (interpolation) |
| `result = ""; for (x) result += f(x)` | `Enum.map_join(items, ", ", &f/1)` |
| `x ?? default` | `x \|\| default` (beware: also catches `false`) |
| `x?.y?.z` | `get_in(x, [:y, :z])` |

### The With Statement

```elixir
def create_order(user_id, product_id, qty) do
  with {:ok, user} <- Users.get(user_id),
       {:ok, product} <- Products.get(product_id),
       :ok <- validate_stock(product, qty),
       {:ok, order} <- insert_order(user, product, qty) do
    {:ok, order}
  else
    {:error, :not_found} -> {:error, "Resource not found"}
    {:error, :insufficient_stock} -> {:error, "Insufficient stock"}
    error -> error
  end
end
```

### For Comprehensions (Key Patterns)

```elixir
# Pattern matching in generators — non-matching silently skipped
for {:ok, val} <- results, do: val

# Collect into different types
for {k, v} <- [a: 1, b: 2], into: %{}, do: {k, v * 2}

# Binary comprehensions
for <<byte <- string>>, into: "", do: process_byte(byte)

# Accumulator control with reduce:
for line <- lines, reduce: %{totals: 0, count: 0} do
  acc -> %{acc | totals: acc.totals + parse(line), count: acc.count + 1}
end
```

### When to Use Each Construct

| Construct | Use When |
|-----------|----------|
| Multiple clauses | Different behaviors for different input patterns |
| Pattern matching | Extracting/matching data structure shapes |
| Guards | Constraints beyond pattern matching |
| `with` | Chaining operations that may fail |
| `for` | Iteration with filtering, multiple sources, or custom collection |
| `case` | Pattern matching on single value |
| `cond` | Multiple boolean conditions |
| `if` | Simple true/false check |

### Pipeline Best Practices

```elixir
# GOOD: Multi-step transformation — each step adds meaning
order
|> calculate_subtotal()
|> apply_discount(coupon)
|> add_tax(state)
|> round_to_cents()

# BAD: Single step — just call the function
name |> String.upcase()
# GOOD:
String.upcase(name)

# Design functions data-first so they compose in pipelines
defmodule StringHelpers do
  def normalize(string) do
    string |> String.trim() |> String.downcase() |> String.replace(~r/\s+/, " ")
  end
  def truncate(string, max) when byte_size(string) <= max, do: string
  def truncate(string, max), do: String.slice(string, 0, max - 3) <> "..."
end

input |> StringHelpers.normalize() |> StringHelpers.truncate(100)

# Break long pipelines into named private functions
def process_orders(orders) do
  orders
  |> filter_valid()
  |> calculate_totals()
  |> apply_discounts()
  |> generate_invoices()
end

defp filter_valid(orders), do: orders |> Enum.filter(&valid_order?/1) |> Enum.reject(&cancelled?/1)
defp calculate_totals(orders), do: Enum.map(orders, &%{&1 | total: calculate_order_total(&1)})

# Conditional steps — use maybe_ helpers to keep pipeline flat
data
|> transform()
|> maybe_validate(opts[:validate])
|> finalize()

defp maybe_validate(data, true), do: validate(data)
defp maybe_validate(data, _), do: data

# Or with then/1 for inline conditionals
data
|> transform()
|> then(fn d -> if opts[:validate], do: validate(d), else: d end)

# tap/1 — inspect without breaking the pipeline (returns input unchanged)
order
|> calculate_total()
|> tap(&Logger.debug("Total: #{&1}"))
|> apply_tax()

# Error pipeline with with — short-circuit on first error
with {:ok, user} <- fetch_user(id),
     {:ok, account} <- fetch_account(user),
     {:ok, balance} <- check_balance(account, amount) do
  {:ok, transfer(balance, amount)}
else
  {:error, :not_found} -> {:error, "User not found"}
  {:error, :insufficient} -> {:error, "Insufficient funds"}
end

# ok/error pipeline — chain with pattern-matching helpers
defp authorize(user, action) do
  user
  |> check_role(action)
  |> check_permissions(action)
  |> check_rate_limit()
end

defp check_role(%{role: :admin} = user, _action), do: {:ok, user}
defp check_role(%{role: :user} = user, :read), do: {:ok, user}
defp check_role(_, action), do: {:error, {:unauthorized, action}}

defp check_permissions({:ok, user}, action), do: Permissions.verify(user, action)
defp check_permissions(error, _action), do: error

defp check_rate_limit({:ok, user}), do: RateLimiter.check(user)
defp check_rate_limit(error), do: error
```

### Multi-Clause Anonymous Functions

```elixir
# Different clauses in Enum callbacks — pattern match + guards
Enum.reduce(events, %{}, fn
  %{type: :credit, amount: amt}, acc -> Map.update(acc, :total, amt, &(&1 + amt))
  %{type: :debit, amount: amt}, acc -> Map.update(acc, :total, -amt, &(&1 - amt))
  _, acc -> acc
end)

# Task result handling
|> Enum.map(fn
  {:ok, result} -> result
  {:exit, :timeout} -> :timed_out
end)
```

### cond with Variable Binding

Assign in condition, use in body — useful for priority resolution:

```elixir
cond do
  val = Map.get(overrides, key) -> val
  val = Map.get(config, key) -> val
  true -> default
end

# Range/threshold branching
cond do
  time > 1_000_000 -> "#{div(time, 1_000_000)}s"
  time > 1_000 -> "#{div(time, 1_000)}ms"
  true -> "#{time}μs"
end
```

### Pin Operator in Maps & Comprehensions

```elixir
# Pin in map pattern — match key from variable
key = :name
%{^key => value} = %{name: "Alice"}     # value = "Alice"

# Pin in comprehension generators
target = 42
for %{id: ^target, data: data} <- records, do: data

# Pin in with clauses
expected = "admin"
with %{role: ^expected} <- get_user(id) do
  :authorized
end
```

> **Deep dive:** [language-patterns.md](language-patterns.md) — nested destructuring, match?/2 edge cases,
> multi-clause anonymous functions, binary comprehensions, pin operator advanced uses, cond with variable binding.
> [code-style.md](code-style.md) — .formatter.exs configuration, Credo checks catalog, readable code patterns,
> pipeline readability, guard ordering, with-chain formatting, naming conventions.

## Application Architecture

### Rules for Application Architecture (LLM)

1. **ALWAYS consult this section, [architecture-reference.md](architecture-reference.md), and OTP Patterns when planning new Elixir projects, major features, or refactoring architecture.**
2. ALWAYS organize domain logic into context modules — public API functions that encapsulate queries, validations, and side effects.
3. ALWAYS start with a single Mix app — only split into umbrella/poncho when you have clear, independent deployment or team boundaries
4. NEVER create circular dependencies between contexts
5. ALWAYS define behaviours for integration boundaries — external APIs, payment gateways, notification services
6. ALWAYS order supervision tree children by dependency — infrastructure first, domain next, endpoints last
7. NEVER expose internal data structures outside their context
8. PREFER protocols when you need polymorphism across types from different contexts
9. ALWAYS use `:one_for_all` strategy when grouping tightly-coupled processes (Registry + DynamicSupervisor)
10. NEVER put business logic in GenServer callbacks — delegate domain logic to pure functions
11. ALWAYS design dependency direction inward — outer layers depend on inner, never reverse
12. PREFER pure functions over processes — use GenServer only when you need shared mutable state, serialized access, or scheduled work
13. ALWAYS use `in_umbrella: true` for sibling dependencies in umbrella apps
14. **ALWAYS design for replaceability.** Use behaviours at integration points.
15. **PREFER small, focused contexts over large ones.** ~500 lines max per context.

### Configuration

```elixir
# config/runtime.exs — secrets and env vars
import Config
if config_env() == :prod do
  config :my_app, MyAppWeb.Endpoint,
    url: [host: System.get_env("PHX_HOST", "example.com"), port: 443, scheme: "https"],
    secret_key_base: System.fetch_env!("SECRET_KEY_BASE")
  config :my_app, MyApp.Repo,
    url: System.fetch_env!("DATABASE_URL"),
    pool_size: String.to_integer(System.get_env("POOL_SIZE", "10"))
end
```

**Rule:** Never call `System.get_env/1` in compile-time config files for values that vary per deployment. Use `runtime.exs`.

### Layout Decision Guide

| Signal | Layout |
|--------|--------|
| Single team, one deployable | Single app with contexts |
| Need hard compile-time boundaries | Umbrella |
| Different teams, shared config OK | Umbrella |
| Different dependency versions needed | Poncho |
| "Should I split?" uncertainty | Don't split — use contexts |

### Application Supervision Tree

```elixir
defmodule MyApp.Application do
  use Application

  @impl true
  def start(_type, _args) do
    children = [
      MyAppWeb.Telemetry,          # 1. Instrumentation
      MyApp.Repo,                  # 2. Database
      {Phoenix.PubSub, name: MyApp.PubSub},  # 3. PubSub
      {Task.Supervisor, name: MyApp.TaskSupervisor},
      MyApp.WorkerRegistry,        # 5. Registry before DynamicSupervisor
      MyApp.WorkerSupervisor,      # 6. Dynamic workers
      MyAppWeb.Endpoint            # 7. HTTP last
    ]
    Supervisor.start_link(children, strategy: :one_for_one, name: MyApp.Supervisor)
  end
end
```

### Context Modules — Domain Boundaries

Group related functionality into context modules — each is a public API boundary (DDD bounded context). This applies to any Elixir application, not just Phoenix.

```elixir
defmodule MyApp.Catalog do
  @moduledoc "Product catalog — public API for all product operations."
  alias MyApp.Catalog.{Product, PriceCalculator}

  # --- Public API (the only functions other modules should call) ---
  defdelegate get_product!(id), to: Product, as: :fetch!
  defdelegate create_product(attrs), to: Product, as: :create
  def calculate_price(product, qty), do: PriceCalculator.total(product, qty)
end
# defdelegate compiles to a direct call — zero overhead, keeps context as clean facade
# Use regular def when you need to wrap, transform args, or add logic
```

**Internal modules are private** — never call them from outside the context:

```elixir
defmodule MyApp.Catalog.PriceCalculator do
  @moduledoc false  # Internal to Catalog context
  def total(product, qty), do: Decimal.mult(product.price, qty)
end
```

**Cross-context communication** — always go through public API:

```elixir
defmodule MyApp.Ordering do
  alias MyApp.Catalog  # Depend on context, not internals

  def place_order(product_id, qty) do
    product = Catalog.get_product!(product_id)            # GOOD
    # MyApp.Catalog.PriceCalculator.total(product, qty)   # BAD — bypasses boundary
    total = Catalog.calculate_price(product, qty)
    do_place_order(product, qty, total)
  end
end
```

**When to split vs merge:** Split when different domain/team/data lifecycle. Merge when shared aggregate root or constant cross-calls. When unsure, keep separate — merging is easier than splitting.

**Behaviours as context contracts** — swap implementations without changing callers:

```elixir
defmodule MyApp.PaymentProvider do
  @callback charge(Decimal.t(), String.t()) :: {:ok, map()} | {:error, term()}
  @callback refund(String.t()) :: :ok | {:error, term()}
end

defmodule MyApp.Payments.Stripe do
  @behaviour MyApp.PaymentProvider
  @impl true
  def charge(amount, token), do: # Stripe API...
  @impl true
  def refund(charge_id), do: # Stripe refund...
end

# Context facade — callers use MyApp.Payments, never know which provider
defmodule MyApp.Payments do
  @provider Application.compile_env(:my_app, :payment_provider, MyApp.Payments.Stripe)
  defdelegate charge(amount, token), to: @provider
  defdelegate refund(charge_id), to: @provider
end
```

**Internal organization** — the context module is the boundary, internals are free-form:

```
lib/my_app/catalog/
├── product.ex            # Data + persistence (internal)
├── category.ex           # Data + persistence (internal)
├── price_calculator.ex   # Pure business logic (internal)
└── import_worker.ex      # Background processing (internal)
```

> **Deep dive — ALWAYS read for architecture planning and refactoring:**
> [architecture-reference.md](architecture-reference.md) — project layouts (single app, umbrella, poncho
> with decision guide), Phoenix contexts (public API design, cross-context communication), layered architecture
> (domain/service/web), pipeline architecture, behaviours as layer contracts, protocols as boundaries,
> configuration (compile-time vs runtime, when to use compile_env vs fetch_env!), refactoring guide, component reuse.

## Production Patterns

### Telemetry (Key Pattern)

```elixir
# Emit events
:telemetry.execute([:my_app, :orders, :created], %{count: 1}, %{order_id: order.id})

# Instrument a block
:telemetry.span([:my_app, :external_api], %{url: url}, fn ->
  result = HTTPClient.get(url)
  {result, %{status: result.status}}
end)
```

### HTTP Clients

**Req** is the modern default. Batteries included: JSON, retries, redirects, compression.

```elixir
# Simple
resp = Req.get!("https://api.example.com/data")

# Reusable client
client = Req.new(base_url: "https://api.example.com", auth: {:bearer, token}, retry: :transient)
{:ok, resp} = Req.get(client, url: "/users")
```

> **Deep dive:** [production.md](production.md) — telemetry deep-dive (attach handlers, span events, custom metrics),
> built-in events table (Phoenix, Ecto, Oban, VM), metrics definitions (counter, sum, distribution, last_value),
> HTTP client patterns (Req, Finch, middleware, retry), Req.Test mock/stub testing, reusable client construction.

## OTP Patterns

### Rules for OTP Code (LLM)

1. **ALWAYS consult this section and Application Architecture when adding processes or restructuring supervision.**
2. **ALWAYS supervise processes.** Never use bare `spawn/spawn_link` for long-running work.
3. **ALWAYS provide a client API** wrapping GenServer calls/casts.
4. **PREFER call over cast.** Use cast only for fire-and-forget where losing messages is acceptable.
5. **NEVER block GenServer callbacks** with I/O, HTTP, or DB queries. Offload to `Task` or use `handle_continue`.
6. **ALWAYS use `{:continue, _}` for post-init work** instead of crashing `init/1`.
7. **NEVER store large data (>100KB) in process state.** Use ETS for large/shared data.
8. **ALWAYS set explicit timeouts** on `GenServer.call`.
9. **PREFER Registry over `:global`** for process discovery within a single node.
10. **NEVER catch `:exit` from GenServer.call** in business logic — let supervision handle failures.
11. **PREFER DynamicSupervisor + Registry** over named GenServers for per-entity processes.
12. **ALWAYS do atomic state updates.** Compute new state fully, then return.
13. **ALWAYS implement `format_status/1`** on GenServers that hold sensitive data.

### When to Use Which OTP Construct

| Need | Use |
|------|-----|
| Stateful request/response | GenServer |
| Explicit state machine with transitions | `:gen_statem` |
| One-off parallel work | Task / Task.async_stream |
| Dynamic per-entity processes | DynamicSupervisor + Registry |
| Reduce single-process bottleneck | PartitionSupervisor |
| Pub/sub within node | Registry with `:duplicate` keys |
| High-read shared data | ETS (`:public`, `read_concurrency: true`) |
| Rarely-changing global config | `:persistent_term` |
| Atomic counters/gauges | `:counters` / `:atomics` |
| Backpressure data pipeline | GenStage / Broadway |
| Periodic scheduled work | `Process.send_after` loop or Oban |

### GenServer

```elixir
defmodule Counter do
  use GenServer

  def start_link(initial \\ 0), do: GenServer.start_link(__MODULE__, initial, name: __MODULE__)
  def increment, do: GenServer.call(__MODULE__, :increment)
  def get, do: GenServer.call(__MODULE__, :get)

  @impl true
  def init(initial), do: {:ok, initial}
  @impl true
  def handle_call(:increment, _from, state), do: {:reply, state + 1, state + 1}
  def handle_call(:get, _from, state), do: {:reply, state, state}
end
```

### Supervisor Strategies

| Strategy | Restarts | Use when |
|----------|----------|----------|
| `:one_for_one` | Only failed child | Children are independent (most common) |
| `:rest_for_one` | Failed + all started after it | Later children depend on earlier ones |
| `:one_for_all` | All children | Children are tightly coupled |

```elixir
# :one_for_one — independent workers (web endpoints, job processors)
children = [
  MyApp.Repo,
  MyApp.Cache,
  {MyApp.Worker, name: :worker_a},
  {MyApp.Worker, name: :worker_b}
]
Supervisor.start_link(children, strategy: :one_for_one)

# :rest_for_one — ordered dependencies (Registry before DynamicSupervisor)
children = [
  {Registry, keys: :unique, name: MyApp.Registry},       # 1st: must exist
  {DynamicSupervisor, name: MyApp.WorkerSup},             # 2nd: uses Registry
  {MyApp.WorkerStarter, registry: MyApp.Registry}         # 3rd: uses both
]
Supervisor.start_link(children, strategy: :rest_for_one)

# :one_for_all — tightly coupled (producer + consumer, writer + reader)
children = [
  {MyApp.EventBus, name: :event_bus},
  {MyApp.EventLogger, bus: :event_bus},
  {MyApp.EventNotifier, bus: :event_bus}
]
Supervisor.start_link(children, strategy: :one_for_all)

# Restart intensity — max 3 restarts in 5 seconds before supervisor gives up
Supervisor.start_link(children,
  strategy: :one_for_one,
  max_restarts: 3,
  max_seconds: 5
)
```

### DynamicSupervisor + Registry

```elixir
defmodule MyApp.WorkerRegistry do
  def child_spec(_), do: Registry.child_spec(keys: :unique, name: __MODULE__)
  def via(id), do: {:via, Registry, {__MODULE__, id}}
end

defmodule MyApp.Worker do
  use GenServer
  def start_link({id, opts}), do: GenServer.start_link(__MODULE__, opts, name: MyApp.WorkerRegistry.via(id))
  def call(id, msg), do: GenServer.call(MyApp.WorkerRegistry.via(id), msg)
end

# Registry MUST start before DynamicSupervisor
children = [MyApp.WorkerRegistry, MyApp.WorkerSupervisor]
```

### gen_statem (State Machines)

```elixir
defmodule MyApp.Connection do
  @behaviour :gen_statem

  def start_link(opts), do: :gen_statem.start_link(__MODULE__, opts, [])
  def connect(pid), do: :gen_statem.call(pid, :connect)

  @impl true
  def init(opts), do: {:ok, :disconnected, %{host: opts[:host], retries: 0}}
  @impl true
  def callback_mode, do: [:state_functions, :state_enter]

  def disconnected(:enter, _old, data), do: {:keep_state, %{data | retries: 0}}
  def disconnected({:call, from}, :connect, data), do: {:next_state, :connecting, data, [{:reply, from, :ok}]}

  def connecting(:enter, _old, data) do
    send(self(), :do_connect)
    {:keep_state_and_data, [{:state_timeout, 5000, :connect_timeout}]}
  end
end
```

**Timeout types:** `{:timeout, ms, event}` (any event cancels), `{:state_timeout, ms, event}` (state change cancels), `{{:timeout, name}, ms, event}` (named, cross-state).

### Task.Supervisor Patterns

```elixir
# async: LINKED — task crash kills caller
Task.Supervisor.async(MyApp.TaskSupervisor, fn -> work() end)

# async_nolink: NOT linked — handle :DOWN in GenServer
task = Task.Supervisor.async_nolink(MyApp.TaskSupervisor, fn -> work() end)

# Handle task result in GenServer
def handle_info({ref, result}, %{task_ref: ref} = state) do
  Process.demonitor(ref, [:flush])
  {:noreply, %{state | task_ref: nil, result: result}}
end
def handle_info({:DOWN, ref, :process, _pid, reason}, %{task_ref: ref} = state) do
  {:noreply, %{state | task_ref: nil}}
end
```

### ETS for Fast Reads

```elixir
:ets.new(:cache, [:named_table, :public, read_concurrency: true])
:ets.insert(:cache, {key, value})
:ets.lookup(:cache, key)  # [{key, value}] or []

# Atomic counters — no race conditions
:ets.new(:stats, [:named_table, :public, write_concurrency: true])
:ets.update_counter(:stats, :requests, {2, 1}, {:requests, 0})  # increment by 1, default 0

# Match specs — server-side filtering (faster than lookup + Enum.filter)
# Find all users with age > 30: table has {name, age, email}
:ets.select(:users, [{{:"$1", :"$2", :"$3"}, [{:>, :"$2", 30}], [{{:"$1", :"$3"}}]}])
# Returns [{name, email}] for matching rows

# Simpler: match_object with partial tuple
:ets.match_object(:cache, {:_, :active, :_})  # all tuples with :active in position 2

# Delete matching entries
:ets.select_delete(:cache, [{{:_, :"$1", :_}, [{:<, :"$1", expired_at}], [true]}])
```

Use ETS instead of GenServer for read-heavy workloads to avoid bottlenecks. Use `write_concurrency: true` when multiple processes write to different keys.

### Graceful Shutdown

```elixir
defmodule MyApp.Worker do
  use GenServer

  def init(opts) do
    Process.flag(:trap_exit, true)  # Required to get terminate/2 callback
    {:ok, %{conn: connect(opts)}}
  end

  @impl true
  def terminate(_reason, %{conn: conn}) do
    # Called on shutdown — drain connections, flush queues, close resources
    Logger.info("Shutting down, closing connection...")
    close(conn)
    :ok  # Return value ignored
  end
end
```

### Process Bottleneck Detection

```elixir
# Quick check — find processes with large mailboxes
for pid <- Process.list(),
    {:message_queue_len, len} = Process.info(pid, :message_queue_len),
    len > 1000,
    do: {pid, len, Process.info(pid, :registered_name)}
```

### Call vs Cast Decision

| Use Call when... | Use Cast when... |
|---|---|
| Need response or confirmation | Fire-and-forget (logging, metrics) |
| Data consistency critical | Notifications, broadcasts |
| Want natural backpressure (caller blocks) | High throughput needed |
| Failures should propagate to caller | Failures can be handled internally |

> **Deep dive:** [otp-reference.md](otp-reference.md) — GenServer/gen_statem callback signatures, child specs,
> supervisor strategies, ETS match specs, process registry patterns, :persistent_term, :counters/:atomics,
> :sys debugging, node operations, RPC/ERPC, release commands.
> [otp-examples.md](otp-examples.md) — rate limiter, connection state machine, worker pool, cache, circuit breaker,
> graceful shutdown, distribution patterns. [otp-advanced.md](otp-advanced.md) — GenStage, Flow, Broadway,
> hot code upgrades.

## Code Organization

### Module Structure

```elixir
defmodule MyApp.User do
  @moduledoc "User management"

  use Ecto.Schema
  import Ecto.Changeset
  alias MyApp.Repo

  @derive {Jason.Encoder, only: [:id, :email]}

  @type t :: %__MODULE__{}

  schema "users" do
    field :email, :string
  end

  @doc "Creates a user"
  @spec create(map()) :: {:ok, t()} | {:error, Ecto.Changeset.t()}
  def create(attrs), do: # ...

  defp validate(changeset), do: # ...
end
```

Order: @moduledoc, use/import/alias/require, module attributes, types, schema/struct, public functions with @doc/@spec, private functions.

### Public vs Private Functions

**Default to `defp`** — only promote to `def` when external callers need it.

| | `def` (public) | `defp` (private) |
|---|---|---|
| **Visibility** | Callable from other modules | Only within defining module |
| **Contract** | Part of module API — add `@doc` + `@spec` | Implementation detail — refactor freely |
| **Testing** | Test directly | Test through public API only |
| **Stability** | Changing breaks callers | Changing is safe |

```elixir
# Public API — small, stable surface
defmodule MyApp.Accounts do
  @doc "Registers a new user, sends welcome email."
  @spec register(map()) :: {:ok, User.t()} | {:error, Changeset.t()}
  def register(attrs) do
    attrs
    |> build_user()
    |> validate_uniqueness()
    |> insert_and_notify()
  end

  # Private — all implementation details
  defp build_user(attrs), do: User.changeset(%User{}, attrs)
  defp validate_uniqueness(changeset), do: unique_constraint(changeset, :email)
  defp insert_and_notify(changeset) do
    with {:ok, user} <- Repo.insert(changeset) do
      Mailer.send_welcome(user)
      {:ok, user}
    end
  end
end
```

**Guidelines:**
- A module's public API should be **as small as possible** — fewer public functions = easier to understand, test, and maintain
- **Extract private helpers** when logic is reused within the module or when a function does more than one thing
- **`do_` prefix** for recursive private helpers of a public function: `def transform(list)` → `defp do_transform(list, acc)`
- **`maybe_` prefix** for conditional operations: `defp maybe_notify(user, true)`, `defp maybe_notify(_user, false)`
- Functions called from other modules **must** be `def` — if you find yourself wanting to call `defp` from outside, rethink the module boundary
- **Context modules** (Phoenix contexts) are the public API for a domain — keep controller-facing functions `def`, keep query-building and validation helpers `defp`

### Naming

- **Modules**: PascalCase (`MyApp.UserController`)
- **Functions/Variables**: snake_case (`find_user_by_email`)
- **Predicates**: End with `?` (`valid?`, `empty?`)
- **Dangerous functions**: End with exclamation mark (`delete！`, `fetch！`)
- **Atoms**: snake_case (`:user_not_found`)

## Code Style, Formatter & Readability

### Rules for Code Style (LLM)

1. ALWAYS run `mix format` — never hand-format code the formatter handles (spacing, indentation, parens, line breaks)
2. ALWAYS configure `.formatter.exs` with `import_deps` for libraries you use — this pulls their `locals_without_parens` settings
3. NEVER fight the formatter — if it reformats your code unexpectedly, restructure your code, don't add workarounds
4. ALWAYS use pipes for 2+ transformation steps — single function calls don't need pipes
5. NEVER pipe into anonymous functions — use `then/1` or extract a named function
6. ALWAYS order pattern match clauses specific → general — guards and literal matches before wildcards
7. ALWAYS place the success/happy path first in case/with clauses — error handling after
8. PREFER guards over if/cond for type and value validation at function boundaries
9. ALWAYS use `@doc false` (not omitting `@doc`) to explicitly mark internal public functions
10. NEVER use semicolons to separate expressions — use line breaks
11. ALWAYS write one pipe per line in multi-step pipelines — never multiple pipes on one line
12. PREFER direct function calls over single-element pipes: `String.upcase(name)` not `name |> String.upcase()`

### Key BAD/GOOD

```elixir
# BAD: Single step pipe
name |> String.upcase()
# GOOD: Direct call
String.upcase(name)

# BAD: Piping to anonymous function
data |> (fn x -> x * 2 end).()
# GOOD: Use then/1
data |> then(&(&1 * 2))

# BAD: Missing import_deps
locals_without_parens: [get: 3, post: 3, field: 2]
# GOOD: Let libraries provide their own config
import_deps: [:phoenix, :ecto, :ecto_sql, :plug]

# BAD: import/alias scattered and ungrouped
import Ecto.Query
alias MyApp.Repo
import Ecto.Changeset
alias MyApp.User

# GOOD: Grouped and alphabetized within groups
import Ecto.Changeset
import Ecto.Query
alias MyApp.{Repo, User}

# BAD: Deeply nested conditionals
if valid?(data) do
  if authorized?(user) do
    process(data)
  else
    {:error, :unauthorized}
  end
else
  {:error, :invalid}
end

# GOOD: with-chain flattens the nesting
with :ok <- validate(data),
     :ok <- authorize(user) do
  process(data)
end
```

> **Deep dive:** [code-style.md](code-style.md) — .formatter.exs configuration (line_length, locals_without_parens,
> plugins, migration options since 1.18+), Credo check catalog (high-priority, readability, consistency),
> pipeline readability (when to break, single-step anti-pattern), guard clause ordering (specific→general),
> pattern match ordering, with-chain formatting, variable naming, pure logic separation, Ecto.Multi.

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

### Key Patterns

```elixir
# Defining a behaviour
defmodule MyApp.Storage do
  @callback fetch(key :: String.t()) :: {:ok, term()} | {:error, :not_found}
  @callback store(key :: String.t(), value :: term()) :: :ok | {:error, term()}
  @optional_callbacks [store: 2]
end

# Implementing with @impl
defmodule MyApp.Storage.ETS do
  @behaviour MyApp.Storage
  @impl true
  def fetch(key), do: ...
  @impl true
  def store(key, value), do: ...
end

# Adapter pattern — runtime dispatch via config
defmodule MyApp.Mailer.Dispatcher do
  def send(to, subject, body) do
    impl = Application.get_env(:my_app, :mailer, MyApp.Mailer.SMTP)
    impl.send_email(to, subject, body)
  end
end
```

### Behaviour + `use` Macro — Inject Defaults

```elixir
defmodule MyApp.Plugin do
  @callback handle(term()) :: {:ok, term()} | {:error, term()}
  @callback name() :: String.t()

  defmacro __using__(_opts) do
    quote do
      @behaviour MyApp.Plugin
      @impl true
      def name, do: __MODULE__ |> Module.split() |> List.last()
      defoverridable name: 0  # Implementers MAY override
    end
  end
end

defmodule MyApp.Plugins.CSV do
  use MyApp.Plugin  # Gets default name/0, must implement handle/1
  @impl true
  def handle(data), do: {:ok, CSV.encode(data)}
end
```

### Behaviour vs Protocol Decision

| | Behaviour | Protocol |
|---|---|---|
| **Dispatch on** | Module identity (passed as config) | Data type of first argument |
| **Testing** | Works with Mox | Does not work with Mox |
| **When to use** | External services, adapters, strategies | Type-specific formatting, encoding, iteration |
| **Example** | `HTTPClient`, `Mailer`, `Storage` | `Jason.Encoder`, `Enumerable`, `Inspect` |

> **Deep dive:** [language-patterns.md](language-patterns.md) — multi-behaviour composition (Ecto adapter pattern),
> dynamic dispatch patterns, DSL recipe (__using__ + accumulated attributes + @before_compile),
> behaviour introspection (module_info, __info__), @optional_callbacks with function_exported?/3,
> defoverridable patterns, testing behaviours with Mox.

## Protocols

### Rules for Protocols (LLM)

1. **PREFER single-function protocols** — the vast majority of stdlib/library protocols define exactly 1 function
2. **ALWAYS put `@derive` BEFORE `defstruct`** (or `schema`) — the compiler warns if it comes after
3. **NEVER implement `for: Map` expecting it to match structs** — structs dispatch through `struct_impl_for/1`, not the Map implementation
4. **Use `@fallback_to_any true`** only when there IS a sensible default
5. **ALWAYS implement all 3 commands** in Collectable: `{:cont, elem}`, `:done`, `:halt`
6. **For Enumerable, return `{:error, __MODULE__}`** from `count/1`, `member?/2`, `slice/1` when O(1) isn't possible
7. **Use `Protocol.derive/3`** for structs you don't own
8. **Guard `for: BitString` implementations** with `is_binary/1`

### Key Patterns

```elixir
# Define protocol
defprotocol MyApp.Renderable do
  @spec render(t()) :: iodata()
  def render(term)
end

# Implement for struct
defimpl MyApp.Renderable, for: MyApp.Widget do
  def render(%{html: html}), do: html
end

# @derive for common protocols
defmodule User do
  @derive {Jason.Encoder, only: [:id, :name, :email]}
  @derive {Inspect, only: [:id, :name]}
  defstruct [:id, :name, :email, :password_hash]
end

# @fallback_to_any — sensible default for all types
defprotocol MyApp.Blank do
  @fallback_to_any true
  def blank?(term)
end

defimpl MyApp.Blank, for: Any do
  def blank?(_), do: false  # Default: nothing is blank
end

defimpl MyApp.Blank, for: [BitString, List] do
  def blank?(""), do: true
  def blank?([]), do: true
  def blank?(_), do: false
end
```

> **Deep dive:** [language-patterns.md](language-patterns.md) — making derivable protocols, Enumerable/Collectable
> implementation, struct dispatch precedence (struct impl beats Map impl), protocol introspection
> (Protocol.consolidated?/1, impl_for/1), consolidation behavior differences in dev vs prod,
> Protocol.derive/3 for structs you don't own, multi-type implementation syntax.

## Documentation & Doctests

### Rules for Documentation (LLM)

1. **ALWAYS add `@moduledoc`** to every public module — `@moduledoc false` for internal/private modules
2. **ALWAYS add `@doc` and `@spec`** to every public function — `@doc false` for intentionally undocumented helpers
3. **ALWAYS start `@moduledoc` and `@doc` with a concise summary line** — ExDoc uses the first paragraph for search/index
4. **ALWAYS include `## Examples` with `iex>` doctests** for pure functions
5. **ALWAYS place `@doc` before the FIRST clause** of multi-clause functions
6. **ALWAYS pair `@deprecated` with `@doc false`** — hide deprecated functions from docs
7. **NEVER write docs that just repeat the function name**
8. **ALWAYS use backtick cross-references** — `` `Module` ``, `` `Module.function/arity` ``, `` `t:type/0` ``
9. **ALWAYS document `:ok/:error` return shapes** explicitly
10. **PREFER `@typedoc`** for complex or important types

### Key Pattern

```elixir
@doc """
Fetches a user by ID, preloading their organization.

Returns `{:ok, user}` if found, `{:error, :not_found}` otherwise.

## Examples

    iex> MyApp.Accounts.get_user(123)
    {:ok, %User{id: 123}}

"""
@spec get_user(pos_integer()) :: {:ok, User.t()} | {:error, :not_found}
def get_user(id), do: ...
```

### Common @spec Patterns

```elixir
@spec process(String.t()) :: {:ok, map()} | {:error, atom()}
@spec transform(list(A)) :: list(A) when A: term()          # Parametric
@spec callback(term(), keyword()) :: :ok | no_return()       # Side-effectful
@type result :: {:ok, t()} | {:error, reason()}              # Reusable type
@type reason :: :not_found | :unauthorized | :invalid_input
```

> **Deep dive:** [documentation.md](documentation.md) — @moduledoc/@doc full patterns, @typedoc/@opaque types,
> @since/@deprecated annotations, doctest syntax (multi-line, exceptions, opaque values, ellipsis matching
> in 1.19+, omitting output), ExDoc configuration (groups_for_modules, extras, source_url, groups_for_docs),
> cross-reference syntax (`t:MyMod.t/0`, `c:callback/1`, `m:Module`), documentation BAD/GOOD pairs.

## Data Structures & Access Patterns

### Data Structure Selection Guide

| Need | Use | Why |
|---|---|---|
| Key-value, known keys | Struct (`%Mod{}`) | Compile-time checks, dot access |
| Key-value, dynamic keys | Map (`%{}`) | O(log n) lookup, any key type |
| Ordered collection | List (`[]`) | Linked list, O(1) prepend, pattern match head |
| Fixed-size grouping | Tuple (`{}`) | O(1) element access, commonly 2-4 elements |
| Function options | Keyword list (`[k: v]`) | Ordered, duplicate keys allowed, last-arg sugar |
| Unique values | MapSet | Set operations (union, intersection, difference) |
| FIFO queue | `:queue` | O(1) amortized enqueue/dequeue |
| Fast concurrent reads | ETS | In-memory table, cross-process access |

### Structs — Key Pattern

```elixir
defmodule User do
  @enforce_keys [:email]
  defstruct [:name, :email, age: 18, role: :user]
  @type t :: %__MODULE__{name: String.t() | nil, email: String.t(), age: non_neg_integer(), role: atom()}

  # Constructor — validate and set defaults in one place
  def new(attrs) when is_map(attrs) do
    struct!(__MODULE__, attrs)   # Raises on unknown keys
  end

  # Named update functions — express intent, hide field names
  def promote(%__MODULE__{} = user), do: %{user | role: :admin}
  def rename(%__MODULE__{} = user, name), do: %{user | name: name}
end

# Update syntax — raises on unknown keys (compile-time safety)
%{user | name: "New Name"}

# Pattern match on struct type — dispatch or validate
def greet(%User{name: name}), do: "Hello, #{name}"
def greet(%Admin{name: name}), do: "Welcome back, #{name}"

# Pipeline of struct transforms
order
|> Order.validate_inventory()
|> Order.apply_discounts()
|> Order.calculate_tax()
|> Order.confirm()

# Struct as behaviour contract — all fields documented in one place
# Functions that take/return the struct live in the same module

# Nested structs — compose domain models
defmodule Customer do
  @enforce_keys [:name, :email]
  defstruct [:name, :email, address: %Address{}]

  # Deep update with put_in (structs support Access via Map)
  def update_city(%__MODULE__{} = c, city), do: put_in(c.address.city, city)
end

# Deriving protocols — declare at struct definition
defmodule Event do
  @derive {Jason.Encoder, only: [:type, :data, :timestamp]}
  @derive {Inspect, only: [:type, :timestamp]}  # Hide sensitive data from logs
  defstruct [:type, :data, :timestamp, :internal_id]
end

# Constructor with ok/error — validate before creating
def new(attrs) do
  case Map.fetch(attrs, :email) do
    {:ok, email} when is_binary(email) -> {:ok, struct!(__MODULE__, attrs)}
    _ -> {:error, :email_required}
  end
end
```

### Lists — Linked Lists

```elixir
# O(1) prepend — ALWAYS build lists by prepending
list = [new_item | existing_list]

# Head/tail pattern matching
[first | rest] = [1, 2, 3]       # first=1, rest=[2,3]
[a, b | rest] = [1, 2, 3]       # a=1, b=2, rest=[3]

# BAD: O(n) append in loops = O(n^2)
Enum.reduce(items, [], fn item, acc -> acc ++ [item] end)
# GOOD: prepend then reverse = O(n)
items |> Enum.reduce([], fn item, acc -> [item | acc] end) |> Enum.reverse()

# IO lists — fastest way to build output (zero-copy accumulation)
rows |> Enum.map(fn row -> [format(row), ?\n] end) |> IO.iodata_to_binary()
File.write!("out.txt", Enum.map(rows, &[&1, ?\n]))  # IO functions accept IO lists directly
```

### Maps — Hash Maps

```elixir
# Creation
%{key: "value"}                        # Atom keys (internal data)
%{"key" => "value"}                    # String keys (external/JSON data)
Map.new(users, &{&1.id, &1})          # From enumerable with transform

# Access — choose based on intent
map.key                                # Raises KeyError if missing (assertive)
map[:key]                              # Returns nil if missing (lenient)
Map.get(map, key, default)             # With explicit default
Map.fetch(map, key)                    # {:ok, val} or :error (pattern matchable)

# Update (returns new map — never mutates)
%{map | key: new_value}                # Update EXISTING key only (raises if missing!)
Map.put(map, key, value)               # Set (create or overwrite)
Map.merge(map1, map2)                  # Merge (map2 wins on conflict)
Map.merge(m1, m2, fn _k, v1, v2 -> v1 + v2 end)  # Custom conflict resolution
```

### Keyword Lists

```elixir
# Lists of {atom, value} tuples — used for function options
[a: 1, a: 2]                            # Duplicate keys allowed (unlike maps)
# Last-arg sugar: brackets dropped
start(host: "localhost", port: 4000)     # Same as start([host: "localhost", ...])
Keyword.get(opts, :key, default)
Keyword.validate!(opts, [:name, :timeout, pool_size: 10])
```

### Tuples — Fixed-Size Containers

```elixir
# O(1) element access, but copying on update — use for 2-4 elements
{:ok, value}                  # Tagged result tuples
{:error, :not_found}          # Typed failure
elem(tuple, 0)                # O(1) access by index
put_elem(tuple, 1, :new_val)  # Returns new tuple (copies all)
```

### Binary Pattern Matching

```elixir
# Parse binary protocols/headers
<<header::16, payload_len::32, payload::binary-size(payload_len), rest::binary>> = data

# Fixed-width fields
<<id::binary-size(8), _sep::8, name::binary-size(20), _::binary>> = record

# UTF-8 character extraction
<<char::utf8, rest::binary>> = "hello"  # char=104, rest="ello"

# Integer extraction with endianness
<<port::16-big>> = <<0x1F, 0x90>>  # port=8080
<<value::32-little-signed>> = <<0xFF, 0xFF, 0xFF, 0xFF>>  # value=-1
```

> **Deep dive:** [data-structures.md](data-structures.md) — list operations (flatten, zip, foldl/foldr, charlist interop,
> IO lists), maps (access, update, nested, patterns), tuples (tagged, ETS), keywords, MapSet (union,
> intersection, difference), structs (constructors, pipelines, accumulator pattern, protocols, nesting,
> GenServer state, anti-patterns), embedded schemas, function captures, binary pattern matching.

## Quick References

### Enum — Top Functions

```elixir
Enum.map(enum, fun)              # Transform each element
Enum.filter(enum, fun)           # Keep where fun returns truthy
Enum.reject(enum, fun)           # Remove where fun returns truthy
Enum.reduce(enum, acc, fun)      # Fold into single value
Enum.find(enum, fun)             # First match or nil
Enum.any?(enum, fun)             # At least one truthy?
Enum.all?(enum, fun)             # All truthy?
Enum.count(enum)                 # Count all
Enum.count(enum, fun)            # Count matching
Enum.flat_map(enum, fun)         # Map then flatten one level
Enum.group_by(enum, key_fun)     # %{key => [elements]}
Enum.sort_by(enum, fun)          # Sort by derived key
Enum.sort_by(enum, fun, :desc)   # Sort descending
Enum.zip(enum1, enum2)           # [{a1, a2}, {b1, b2}]
Enum.chunk_every(enum, n)        # [[a,b], [c,d], [e]]
Enum.take(enum, n)               # First n elements
Enum.into(enum, collectable)     # Into map, MapSet, etc.
Enum.reduce_while(enum, acc, fun) # {:cont, acc} or {:halt, acc}
Enum.with_index(enum)            # [{elem, 0}, {elem, 1}, ...]
Enum.with_index(enum, 1)         # Start from 1
Enum.frequencies(enum)           # %{elem => count}
Enum.frequencies_by(enum, fun)   # %{key => count}
Enum.map_reduce(enum, acc, fun)  # {mapped_list, final_acc}
Enum.uniq_by(enum, fun)          # Deduplicate by key function
Enum.each(enum, fun)             # Invoke fun on each element (returns :ok)
Enum.zip_with([a, b], fn [x, y] -> {x, y} end)  # Zip with transform
Enum.min_max_by(enum, fun)       # {min, max} by key
Enum.slide(enum, index, to)      # Move element (1.13+)
```

### Map — Top Functions

```elixir
Map.get(map, key, default)       # Value or default
Map.put(map, key, value)         # Add or replace
Map.merge(map1, map2)            # Combine (map2 wins conflicts)
Map.update(map, key, default, fun) # Update or set default
Map.delete(map, key)             # Remove key
Map.keys(map)                    # List of keys
Map.values(map)                  # List of values
Map.new(enum, fn x -> {k, v} end) # Build from enumerable
```

### String — Top Functions

```elixir
String.split("a,b,c", ",")              # ["a", "b", "c"]
String.trim("  hi  ")                   # "hi"
String.replace("hello", "l", "r")       # "herro"
String.starts_with?("hello", "hel")     # true
String.contains?("hello", "ell")        # true
String.downcase("Hello")                # "hello"
String.slice("hello", 1..3)             # "ell"
String.to_integer("42")                 # 42
```

### File, Path & System — Top Functions

```elixir
# Reading/writing
File.read!("path")                      # binary (raises on error)
File.write!("path", content)            # :ok (raises on error)
File.read("path")                       # {:ok, binary} | {:error, reason}
File.stream!("path")                    # lazy line stream (large files)

# Checking
File.exists?("path")                    # boolean
File.dir?("path")                       # boolean
File.regular?("path")                   # boolean (is regular file?)
File.stat!("path")                      # %File.Stat{size: ..., type: ...}

# Filesystem operations
File.mkdir_p!("a/b/c")                  # create dirs recursively
File.cp!("src", "dst")                  # copy file
File.cp_r!("src_dir", "dst_dir")        # copy recursively
File.rm_rf("dir")                       # {:ok, files} — remove recursively
File.rename("old", "new")               # rename/move
File.ls("dir")                          # {:ok, [filenames]}
File.touch!("path")                     # create or update timestamp

# Path manipulation
Path.join("a", "b")                     # "a/b"
Path.join(["a", "b", "c"])              # "a/b/c"
Path.expand("../file", __DIR__)         # absolute path
Path.basename("/a/b/c.ex")              # "c.ex"
Path.dirname("/a/b/c.ex")               # "/a/b"
Path.extname("file.ex")                 # ".ex"
Path.rootname("file.ex")                # "file"
Path.wildcard("lib/**/*.ex")            # glob matching
Path.relative_to("a/b/c", "a")         # "b/c"

# System
System.get_env("HOME")                  # value or nil
System.fetch_env!("DATABASE_URL")       # value or raise
System.cmd("git", ["status"])           # {output, exit_code}
System.monotonic_time(:millisecond)     # for measuring durations
System.tmp_dir!()                       # temp directory path
```

> **Deep dive:** [quick-references.md](quick-references.md) — full Enum (transform, filter, reduce, search, group,
> collect, access), Map (read, write, transform, pop, nested update), Keyword (read, write, split, validate),
> List (operations, tuple-list), String (split/join, trim/pad, search/test, transform, convert, parsing),
> Regex, URI/Encoding, Date/Time, IO/Inspect, Access/Nested Data, Process, Range, Agent, Erlang stdlib (21 modules).

## Stream, Enum, and the Enumerable Protocol

### When to Use Stream vs Enum

| Use `Stream` when | Use `Enum` when |
|---|---|
| Large/infinite data | Small collections (< 10K) |
| File processing line-by-line | Result needed immediately |
| Multiple transformations on large data | Simple map/filter/reduce |
| Need to limit (take first N) | Need all results |

```elixir
# Stream for large files — lazy, processes one line at a time
File.stream!("huge.csv")
|> Stream.map(&String.trim/1)
|> Stream.reject(&(&1 == ""))
|> Stream.take(1000)
|> Enum.to_list()

# Stream.iterate — infinite sequence from seed
Stream.iterate(1, &(&1 * 2)) |> Enum.take(10)  # [1, 2, 4, 8, 16, ...]

# Stream.unfold — generate from state, stop with nil
Stream.unfold(10, fn
  0 -> nil                        # Stop
  n -> {n, n - 1}                 # {emit, next_state}
end) |> Enum.to_list()            # [10, 9, 8, ..., 1]

# Stream.resource — acquire/generate/cleanup (DB cursors, API pagination)
Stream.resource(
  fn -> fetch_page(1) end,                          # init: first page
  fn
    {[], _page} -> {:halt, nil}                     # no more items → stop
    {[h | t], page} -> {[h], {t, page}}             # emit one item
    {_, page} -> {[], fetch_page(page + 1)}         # fetch next page (not shown as practical)
  end,
  fn _ -> :ok end                                   # cleanup
)

# Endless generators — infinite streams consumed lazily
random_floats = Stream.repeatedly(fn -> :rand.uniform() end)
Enum.take(random_floats, 5)           # [0.234, 0.891, 0.112, ...]

ids = Stream.iterate(1, &(&1 + 1))   # 1, 2, 3, 4, ... forever
timestamps = Stream.repeatedly(fn -> DateTime.utc_now() end)

# Combine infinite streams with data
orders
|> Stream.zip(ids)                    # {order, id} pairs
|> Enum.take(100)                     # materialize only what you need

# Pipeline: chain Stream, terminate with Enum
orders
|> Stream.filter(&(&1.status == :pending))
|> Stream.map(&calculate_total/1)
|> Stream.reject(&(&1.total == 0))
|> Enum.sum()                     # Enum call triggers the lazy pipeline

# Stream.chunk_while — variable-size chunks with custom logic
# Group log lines into multi-line entries (entry starts with timestamp)
File.stream!("app.log")
|> Stream.chunk_while([], fn
  line, [] -> {:cont, [line]}
  <<d, _::binary>> = line, acc when d in ?0..?9 -> {:cont, Enum.reverse(acc), [line]}
  line, acc -> {:cont, [line | acc]}
end, fn acc -> {:cont, Enum.reverse(acc), []} end)

# Stream.transform — stateful stream transformation
# Rate-limit: emit at most 10 items per second
Stream.transform(items, fn -> :ok end, fn item, acc ->
  Process.sleep(100)
  {[item], acc}
end, fn _acc -> :ok end)
```

> **Deep dive:** [language-patterns.md](language-patterns.md) — Enumerable protocol implementation (reduce/3,
> count/1, member?/2, slice/1), stream creators (iterate, unfold, resource), stream transforms (chunk_while,
> transform), consuming streams safely, practical stream patterns (file processing, pagination, rate limiting),
> Collectable protocol, lazy evaluation gotchas.

## Recursion Patterns

**Rule: Prefer Enum functions.** Use recursion only when you need early termination with complex conditions, multiple accumulators, or tree/graph traversal. Use `Stream` for infinite/generative sequences.

**Tail call optimization (TCO):** When a function's last expression is a call to itself, the BEAM reuses the stack frame — constant memory, no stack overflow regardless of depth.

- Operations after the call break TCO — `[h | func(t)]` is NOT tail-recursive (cons happens after return). Accumulate and reverse instead.
- `try/rescue/catch` blocks prevent TCO — the BEAM keeps the frame for exception handling.
- Stack traces lose intermediate frames — reused frames mean you won't see every recursion step in crash traces.
- `case`, `if`, `with` around the call are fine — TCO applies as long as the recursive call is last in whichever branch executes.

```elixir
# Accumulator pattern (tail-recursive)
def sum(list), do: sum(list, 0)
defp sum([], acc), do: acc
defp sum([head | tail], acc), do: sum(tail, acc + head)

# Build-and-reverse (building a list)
# Prepending [new | list] is O(1). Appending list ++ [new] is O(n).
# Build reversed, reverse once at the end.
def transform(list), do: do_transform(list, [])
defp do_transform([], acc), do: Enum.reverse(acc)
defp do_transform([h | t], acc), do: do_transform(t, [process(h) | acc])

# Multiple accumulators
def partition(list), do: partition(list, [], [])
defp partition([], evens, odds), do: {Enum.reverse(evens), Enum.reverse(odds)}
defp partition([h | t], evens, odds) do
  if rem(h, 2) == 0,
    do: partition(t, [h | evens], odds),
    else: partition(t, evens, [h | odds])
end
# But prefer: Enum.split_with(list, &(rem(&1, 2) == 0))

# Tree traversal (recursion is the right tool)
def flatten_tree(%{children: children, value: value}) do
  [value | Enum.flat_map(children, &flatten_tree/1)]
end
def flatten_tree(%{value: value}), do: [value]

# Infinite generation - use Stream, not raw recursion
def fibonacci do
  Stream.unfold({0, 1}, fn {a, b} -> {a, {b, a + b}} end)
end
# Enum.take(fibonacci(), 10) => [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

## Error Handling

### ok/error Tuples

The standard Elixir convention for results. Use atoms for error types, structs/maps for rich errors.

```elixir
# Return conventions — be consistent within a context
{:ok, value}                        # Success with data
:ok                                 # Success, no data (side-effect confirmation)
{:error, :not_found}                # Typed failure (atom)
{:error, %Changeset{}}              # Rich failure (struct with details)
{:error, {reason, details}}         # Compound failure

# Pattern match with case
case Repo.fetch(User, id) do
  {:ok, user} -> process(user)
  {:error, :not_found} -> create_default(id)
  {:error, reason} -> log_and_fail(reason)
end

# Bang (!) variants — raise on error, used when failure is unexpected
user = Repo.get!(User, id)         # Raises Ecto.NoResultsError
file = File.read!(path)            # Raises File.Error

# Writing bang/non-bang pairs
def fetch_config(key) do
  case lookup(key) do
    nil -> {:error, :not_found}
    val -> {:ok, val}
  end
end

def fetch_config!(key) do
  case fetch_config(key) do
    {:ok, val} -> val
    {:error, reason} -> raise "Config #{key} failed: #{reason}"
  end
end

# Wrapping external results
with {:ok, resp} <- HTTPClient.get(url),
     {:ok, body} <- Jason.decode(resp.body) do
  {:ok, body}
end
# Returns the first {:error, _} from the chain automatically

# Multi-clause functions — match directly on ok/error
def handle_result({:ok, user}), do: send_welcome(user)
def handle_result({:error, :not_found}), do: redirect_to_signup()
def handle_result({:error, _reason}), do: show_generic_error()

# Tagged tuples in with — label each step for targeted error handling
with {:user, {:ok, user}} <- {:user, fetch_user(id)},
     {:auth, :ok} <- {:auth, authorize(user, action)},
     {:save, {:ok, result}} <- {:save, save(user)} do
  {:ok, result}
else
  {:user, {:error, _}} -> {:error, :user_not_found}
  {:auth, {:error, _}} -> {:error, :unauthorized}
  {:save, {:error, changeset}} -> {:error, changeset}
end

# Filtering ok results from a list
results = Enum.map(items, &process/1)
successes = for {:ok, val} <- results, do: val
failures = for {:error, reason} <- results, do: reason

# ok/error in GenServer — pass through the tuple
def handle_call(:get, _from, state) do
  case compute(state) do
    {:ok, result} -> {:reply, {:ok, result}, state}
    {:error, _} = err -> {:reply, err, state}  # Capture and forward
  end
end
```

**When to use which:**
- Non-bang (`fetch/1`) — caller decides how to handle failure
- Bang (`fetch!/1`) — failure is a bug, crash early (scripts, seeds, known-good paths)
- `:ok` atom — fire-and-forget side effects (logging, cache writes, sending messages)

### Let It Crash

Don't rescue unknown errors — let supervision handle it. Reserve `try/rescue` for system boundaries only.

### Error Kernel Design

Keep critical state in stable processes, volatile work in expendable ones:

```elixir
children = [
  MyApp.ConfigStore,    # Stable kernel — rarely crashes
  MyApp.Repo,
  {DynamicSupervisor, name: MyApp.WorkerSupervisor}  # Volatile workers
]
Supervisor.start_link(children, strategy: :rest_for_one)
```

### defexception Patterns

```elixir
defmodule MyApp.NotFoundError do
  defexception [:message, :resource, :id]

  @impl true
  def exception(opts) do
    resource = Keyword.fetch!(opts, :resource)
    id = Keyword.fetch!(opts, :id)
    %__MODULE__{message: "#{resource} #{id} not found", resource: resource, id: id}
  end
end
```

> **Deep dive:** [language-patterns.md](language-patterns.md) — defexception patterns (message/1, custom fields),
> ok/error tuple conventions, let it crash philosophy, error kernel design (separate error-prone from critical
> state), exit reason classification (:normal, :shutdown, {:shutdown, term}), with-chain error handling,
> rescue vs catch, reraise/3.

## Advanced Patterns

> Extended patterns from Req, Broadway, and Absinthe including bidirectional step pipelines, option registration, private data namespaces, error delegation, atomics for rate limiting, persistent term namespacing, status as tagged exception, collectable with streaming hash, caller acknowledger for testing, coordinated shutdown, telemetry integration, Plug halt semantics, Task.async_stream patterns, AST traversal, changeset semantics, and phase pipeline pattern.

> **Deep dive:** [language-patterns.md](language-patterns.md) — sigils (~r, ~w, ~s, custom), comprehensions
> (generators, filters, into:, reduce:, binary), Access module (all(), at(), filter(), key()),
> dynamic dispatch, metaprogramming basics (quote/unquote, __ENV__, Module attributes),
> Req bidirectional pipelines, Broadway patterns, option registration.

## Ecto & Database

> **Supporting files:** Quick-reference tables in [ecto-reference.md](ecto-reference.md). Complete working examples in [ecto-examples.md](ecto-examples.md).

### Rules for Writing Ecto Code (LLM)

1. **ALWAYS use changesets for external data.** Use `cast/4` for user/API input (string keys, type casting). Use `change/2` only for trusted internal data (atom keys, no casting).
2. **NEVER call Repo directly from boundary modules** (controllers, LiveViews, CLI commands, GenServer callbacks). Always go through context modules (e.g., `Blog.create_post/1`). Contexts are the public API.
3. **NEVER expose Ecto schemas outside their context.** Return structs, maps, or dedicated types from context functions.
4. **ALWAYS use `Repo.transact/2` for transactions**, not the legacy `Repo.transaction/2`.
5. **ALWAYS separate validations from constraints.** Validations (`validate_*`) run immediately. Constraints (`*_constraint`) are deferred to the database.
6. **NEVER use `unsafe_validate_unique/4` as a security measure.** Use it only for UX hints.
7. **PREFER multiple changeset functions over one god changeset.**
8. **ALWAYS use `Ecto.Multi` for multi-step operations** that must succeed or fail together.
9. **NEVER preload inside `Enum.map`.** That's N+1. Use `Repo.preload(list, [:assoc])` to batch.
10. **PREFER composable query functions** over monolithic queries.
11. **ALWAYS use `on_conflict` for upserts**, not get-then-insert which has race conditions.
12. **NEVER call `Repo.stream/2` outside a transaction.**
13. **ALWAYS use `apply_action/2` for changeset validation without persistence.**
14. **PREFER migration-level `references(on_delete:)` over schema-level `:on_delete`.**
15. **ALWAYS use concurrent indexes** for production tables with `@disable_ddl_transaction true` and `@disable_migration_lock true`.

### Schema & Changeset

```elixir
defmodule MyApp.Blog.Post do
  use Ecto.Schema
  import Ecto.Changeset

  schema "posts" do
    field :title, :string
    field :body, :text
    field :status, Ecto.Enum, values: [:draft, :published, :archived]
    belongs_to :author, MyApp.Accounts.User
    has_many :comments, MyApp.Blog.Comment
    timestamps(type: :utc_datetime_usec)
  end

  def changeset(post, attrs) do
    post
    |> cast(attrs, [:title, :body, :status, :author_id])
    |> validate_required([:title, :body, :author_id])
    |> validate_length(:title, min: 3, max: 200)
    |> unique_constraint(:slug)
    |> foreign_key_constraint(:author_id)
  end
end
```

### Composable Queries

```elixir
import Ecto.Query
def published(query \\ Post), do: from(p in query, where: p.status == :published)
def by_author(query, id), do: from(p in query, where: p.author_id == ^id)
def recent(query), do: from(p in query, order_by: [desc: p.published_at])
Post |> published() |> by_author(user_id) |> recent() |> Repo.all()

# Joins and preloads
from(p in Post,
  join: a in assoc(p, :author),
  where: a.active == true,
  preload: [author: a],
  order_by: [desc: p.inserted_at])
|> Repo.all()
```

### Migration & Multi — see [ecto-reference.md](ecto-reference.md) and [ecto-examples.md](ecto-examples.md)

### Common Mistakes (BAD/GOOD)

```elixir
# BAD: N+1 queries
Enum.map(posts, fn p -> Repo.preload(p, :comments) end)
# GOOD: Batch preload
posts = Repo.all(Post) |> Repo.preload([:comments, :author])

# BAD: TOCTOU race
case Repo.get_by(Post, slug: slug) do
  nil -> Repo.insert(%Post{slug: slug})
  post -> Repo.update(Post.changeset(post, attrs))
end
# GOOD: Atomic upsert
Repo.insert(%Post{slug: slug, title: title},
  on_conflict: [set: [title: title]], conflict_target: :slug)

# BAD: God changeset
def changeset(user, attrs), do: cast(user, attrs, [:name, :email, :password, :role, :admin_notes])
# GOOD: Separate changesets per action
def registration_changeset(user, attrs), do: cast(user, attrs, [:name, :email, :password]) |> ...
def profile_changeset(user, attrs), do: cast(user, attrs, [:name]) |> ...
```

## Type System (Elixir 1.17–1.20)

### Rules for Working with the Type System (LLM)

1. **ALWAYS trust compiler type warnings** — they indicate real type mismatches. Fix the code, don't suppress.
2. **Understand `dynamic()`** — it opts a value out of type checking. Use at system boundaries (JSON parsing, external data).
3. **NEVER add type annotations just to silence warnings** — fix the underlying type issue instead.
4. **PREFER pattern matching over type guards** when possible — the type system tracks patterns automatically.
5. **ALWAYS read warnings as set operations** — "expected `integer()`, got `integer() or atom()`" means your value's type is a superset of what's expected.
6. **ALWAYS use `@spec` on public functions** — even though the compiler infers types, specs serve as documentation and enable Dialyzer.
7. **ALWAYS check what version introduced a feature** — Elixir 1.17 started type inference, 1.18 added pattern/guard typing, 1.19 adds map/list types, 1.20 adds function signatures.

### Key Concepts

- **Set-theoretic types:** Types are sets. `integer() or atom()` is a union set. `integer() and atom()` is empty (no value is both).
- **`dynamic()`:** Gradual typing escape hatch. `dynamic()` intersected with any type passes. Used for external data.
- **Inference:** The compiler infers types from patterns, guards, and usage. No annotations needed for internal code.
- **Warnings, not errors:** Type mismatches produce warnings. Code still compiles. Enable `--warnings-as-errors` in CI.

> **Deep dive:** [type-system.md](type-system.md) — set-theoretic notation (union, intersection, negation),
> how atoms/tuples/lists/maps/functions are modeled as types, dynamic() deep-dive (when to use, interaction
> with inference), type inference mechanics (cross-clause narrowing, function body inference), reading compiler
> warnings as set operations, @spec/@type vs gradual types, notation comparison table, roadmap (1.17-1.20).

## More Quick References

### Keyword — Top Functions

```elixir
Keyword.get(kw, :key, default)       # Value or default
Keyword.fetch!(kw, :key)             # Value or raise
Keyword.put(kw, :key, value)         # Add/replace (keeps last)
Keyword.merge(kw1, kw2)              # Combine (kw2 wins)
Keyword.take(kw, [:a, :b])           # Keep only these keys
Keyword.drop(kw, [:a, :b])           # Remove these keys
Keyword.validate!(kw, [:a, :b, c: 3]) # Validate keys, set defaults (1.13+)
Keyword.pop(kw, :key, default)       # {value, rest}
Keyword.split(kw, [:a])              # {[a: 1], [b: 2]}
```

### Regex Quick Reference

```elixir
Regex.match?(~r/^\d+$/, "123")          # true
Regex.run(~r/(\w+)@(\w+)/, "a@b")       # ["a@b", "a", "b"]
Regex.scan(~r/\d+/, "a1b2c3")           # [["1"], ["2"], ["3"]]
Regex.replace(~r/\d/, "a1b2", "*")       # "a*b*"
Regex.split(~r/[,;]/, "a,b;c")          # ["a", "b", "c"]
Regex.named_captures(~r/(?<y>\d{4})-(?<m>\d{2})/, "2025-03")  # %{"y" => "2025", "m" => "03"}
```

### Date, Time & NaiveDateTime

```elixir
Date.utc_today()                         # ~D[2025-03-16]
Date.diff(~D[2025-03-16], ~D[2025-01-01]) # 74
DateTime.utc_now()                       # ~U[2025-03-16 12:00:00Z]
NaiveDateTime.utc_now()                  # ~N[2025-03-16 12:00:00]
DateTime.diff(dt1, dt2, :second)         # Difference in seconds
Calendar.strftime(datetime, "%Y-%m-%d")  # "2025-03-16"
```

### Process Essentials

```elixir
self()                                   # Current process PID
Process.send_after(pid, :msg, 5_000)     # Send message after 5s
Process.alive?(pid)                      # Is process running?
Process.monitor(pid)                     # Monitor for :DOWN messages
Process.link(pid)                        # Bidirectional link (crash together)
spawn(fn -> work() end)                  # Fire-and-forget (no linking!)
spawn_link(fn -> work() end)             # Linked — crashes propagate
Task.async(fn -> work() end) |> Task.await()  # Async with result
```

## Erlang Standard Library

```elixir
# :timer — delays and periodic work
:timer.send_after(5_000, self(), :tick)     # Send message after 5s
:timer.apply_after(1_000, Module, :fun, []) # Call function after 1s
{time_us, result} = :timer.tc(fn -> work() end)  # Measure execution

# :queue — efficient FIFO (O(1) amortized in/out)
q = :queue.new() |> :queue.in(:a) |> :queue.in(:b)
{{:value, :a}, q} = :queue.out(q)

# :persistent_term — fast reads, expensive writes (global config)
:persistent_term.put({MyApp, :config}, %{max_retries: 3})
:persistent_term.get({MyApp, :config})

# :ets — in-memory concurrent storage (see OTP section for patterns)
:ets.new(:cache, [:named_table, :public, read_concurrency: true])
:ets.insert(:cache, {key, value, System.monotonic_time()})
:ets.lookup(:cache, key)  # [{key, value, timestamp}]
```

> **Deep dive:** [quick-references.md](quick-references.md) — :queue (in_r, out_r, peek, to_list), :persistent_term (erase, info), :atomics/:counters (lock-free increment/add/sub, compare_exchange), :ordsets (sorted unique sets), :digraph (directed graphs, shortest path), :gb_trees (balanced trees), :array (functional arrays), :math/:rand (uniform, normal), :binary (split, match, copy, referenced_byte_size), :erlang (system_info, memory, monotonic_time, process_info), :lists (keyfind, keystore, usort, foldl), :io_lib (format strings), :calendar (date/time conversion), :unicode (characters_to_binary), :zlib (gzip/gunzip), :os (cmd, env), :sys (process debugging), :dets (disk-based ETS), :file (low-level file ops), Erlang data structure selection guide

## JSON Encoding

```elixir
# Built-in JSON module (Elixir 1.18+)
JSON.encode!(%{name: "test", age: 30})
JSON.decode!(~s({"name":"test"}))
```

## Anti-Patterns to Avoid

### Imperative Habits (Most Common LLM Mistakes)

```elixir
# BAD: Enum.each to build result (returns :ok, not accumulated value)
result = []
Enum.each(items, fn item -> result = [process(item) | result] end)
# result is still [] — rebinding doesn't work!

# GOOD: Use Enum.map
result = Enum.map(items, &process/1)

# BAD: if/else chain for structural dispatch
def handle(msg) do
  if is_map(msg) and Map.has_key?(msg, :type) do
    if msg.type == :error, do: handle_error(msg), else: handle_ok(msg)
  end
end

# GOOD: Multi-clause functions
def handle(%{type: :error} = msg), do: handle_error(msg)
def handle(%{type: _} = msg), do: handle_ok(msg)

# BAD: Mutable accumulator thinking
count = 0
Enum.each(items, fn _ -> count = count + 1 end)
# count is still 0!

# GOOD: Enum.count or Enum.reduce
count = Enum.count(items)
count = Enum.reduce(items, 0, fn _, acc -> acc + 1 end)

# BAD: String concatenation in loops
Enum.reduce(rows, "", fn row, acc -> acc <> format(row) <> "\n" end)

# GOOD: IO lists
rows |> Enum.map(fn row -> [format(row), ?\n] end) |> IO.iodata_to_binary()

# BAD: try/rescue for expected failures
try do
  user = Repo.get!(User, id)
rescue
  Ecto.NoResultsError -> nil
end

# GOOD: ok/error pattern
case Repo.get(User, id) do
  nil -> {:error, :not_found}
  user -> {:ok, user}
end
```

### Process & OTP Anti-Patterns

```elixir
# BAD: GenServer as bottleneck for reads
def get(key), do: GenServer.call(__MODULE__, {:get, key})
# GOOD: Direct ETS access
def get(key), do: :ets.lookup(__MODULE__, key)

# BAD: Partial state update (crash between steps corrupts state)
def handle_call(:transfer, _from, state) do
  state = update_in(state.account_a, &(&1 - 100))
  external_api_call()  # May crash here!
  state = update_in(state.account_b, &(&1 + 100))
  {:reply, :ok, state}
end
# GOOD: Atomic state update
def handle_call(:transfer, _from, state) do
  :ok = external_api_call()
  new_state = state |> update_in([:account_a], &(&1 - 100)) |> update_in([:account_b], &(&1 + 100))
  {:reply, :ok, new_state}
end
```

### Control Flow Anti-Patterns

```elixir
# BAD: if/else instead of pattern matching
def status(user), do: if user.active, do: :active, else: :inactive
# GOOD
def status(%{active: true}), do: :active
def status(%{active: false}), do: :inactive

# BAD: Boolean parameters obscure intent
fetch_users(true)
# GOOD: Separate functions with clear names
fetch_active_users()
```

### Pattern Matching Gotchas

```elixir
# BAD: %{} matches ANY map, not just empty maps
def handle(%{}), do: :empty       # Matches %{a: 1} too!
# GOOD: Guard for empty map
def handle(map) when map_size(map) == 0, do: :empty
def handle(map), do: :has_keys

# BAD: Atom keys don't match string keys (common with JSON/params)
%{name: name} = %{"name" => "Jo"}  # MatchError!
# GOOD: Match with the correct key type
%{"name" => name} = params          # External data uses string keys
%{name: name} = internal_map        # Internal data uses atom keys

# BAD: Forgot pin — variable rebinds instead of matching
expected = :ok
case result do
  expected -> :matched        # ALWAYS matches! expected rebinds to result
end
# GOOD: Pin to match against existing value
case result do
  ^expected -> :matched       # Only matches if result == :ok
end
```

### Data Structure Anti-Patterns

```elixir
# DANGEROUS: Atoms from user input (exhausts atom table ~1M limit)
String.to_atom(user_input)
Jason.decode!(json, keys: :atoms)
# SAFE: to_existing_atom or explicit mapping
String.to_existing_atom(user_input)
Jason.decode!(json, keys: :strings)     # Default, safe

# BAD: String concatenation in loops (O(n^2) — copies on every <>)
Enum.reduce(items, "", fn i, acc -> acc <> "#{i}\n" end)
# GOOD: IO lists (zero-copy accumulation)
items |> Enum.map(&["Item: ", &1, "\n"]) |> IO.iodata_to_binary()
```

> **Deep dive:** [architecture-reference.md](architecture-reference.md) — full anti-patterns catalog with BAD/GOOD pairs for control flow (if/else chains, boolean params), pattern matching gotchas (empty maps, atom/string keys, keyword list order, integer/float, pin operator, IEEE 754 -0.0), cross-type comparisons (term ordering surprises), data structures (atom exhaustion, string concat), processes & OTP (GenServer bottleneck, blocking callbacks, unbounded mailbox, unsupervised processes, Task.async in GenServer), performance (N+1 queries, list as lookup table)

## State Machines

> For comprehensive guidance, use the `state-machine` skill.

**Use when:** lifecycle management (orders, subscriptions, documents), protocol implementation (TCP, auth flows, 2PC), resource management (circuit breakers, connection pools), hardware/device control (motors, sensors). **Skip when:** simple CRUD, boolean flags, no meaningful transitions.

| Approach | Best For | Trade-off |
|----------|----------|-----------|
| **gen_statem / GenStateMachine** | Process-based FSM, timeouts, concurrent state | Full OTP power, more boilerplate |
| **AshStateMachine** | Ash Framework resources | Declarative, integrates with Ash policies |
| **fsmx / Machinery** | Ecto schemas with state field | Lightweight, database-backed |
| **Pure pattern matching** | Simple state transitions | Minimal deps, manual validation |

```elixir
# BAD: State logic scattered across conditionals
def process(order, action) do
  cond do
    order.state == :pending and action == :confirm -> confirm(order)
    order.state == :confirmed and action == :ship -> ship(order)
    true -> {:error, :invalid_action}
  end
end

# GOOD: Pure function state machine (no process needed)
defmodule Order do
  defstruct state: :pending, id: nil
  def transition(%{state: :pending} = o, :confirm), do: {:ok, %{o | state: :confirmed}}
  def transition(%{state: :pending} = o, :cancel),  do: {:ok, %{o | state: :cancelled}}
  def transition(%{state: :confirmed} = o, :ship),  do: {:ok, %{o | state: :shipped}}
  def transition(_, _), do: {:error, :invalid_transition}
end

# GOOD: GenStateMachine (when you need process, timeouts, concurrent access)
defmodule OrderFSM do
  use GenStateMachine, callback_mode: :state_functions
  def start_link(id), do: GenStateMachine.start_link(__MODULE__, id)
  def confirm(pid), do: GenStateMachine.call(pid, :confirm)

  @impl true
  def init(id), do: {:ok, :pending, %{id: id}}
  def pending({:call, from}, :confirm, data),
    do: {:next_state, :confirmed, data, [{:reply, from, :ok}]}
  def pending({:call, from}, :cancel, data),
    do: {:next_state, :cancelled, data, [{:reply, from, :ok}]}
end
```

## Event Sourcing & CQRS

> **ALWAYS read** [eventsourcing-reference.md](eventsourcing-reference.md) for rules, architecture, BAD/GOOD patterns, Commanded API, router DSL, middleware, projections, process managers, and mix tasks. Complete working examples in [eventsourcing-examples.md](eventsourcing-examples.md).

**Use when:** complex domains with audit trails, undo/replay, event-driven integration. **Don't use for:** simple CRUD, static data, OLAP. **Hybrid:** event source only critical bounded contexts.

## Testing

> **Supporting files:** Quick-reference in [testing-reference.md](testing-reference.md). Complete examples in [testing-examples.md](testing-examples.md).

### Rules for Writing Elixir Tests (LLM)

1. **ALWAYS use `async: true`** unless the test modifies global state (ETS named tables, Application env, named GenServers).
2. **ALWAYS use `Ecto.Adapters.SQL.Sandbox`** for database tests.
3. **ALWAYS mock at system boundaries only** (HTTP clients, email, payment gateways). Never mock modules you own.
4. **ALWAYS use `assert_receive`/`refute_receive` with explicit timeouts** instead of `Process.sleep`.
5. **ALWAYS use `setup :verify_on_exit!`** with Mox.
6. **ALWAYS use `errors_on/1` helper** to assert changeset errors.
7. **NEVER test private functions directly.** Test through the public API.
8. **ALWAYS use ExMachina factories** (or equivalent) for test data.
9. **ALWAYS explicitly allow sandbox access** for spawned processes.
10. **ALWAYS test LiveView through `live/2`** and assert with `has_element?/3`, `render_click/2`, `render_submit/2`.
11. **ALWAYS use `describe` blocks** to group related tests.
12. **NEVER leave flaky tests.** Fix the root cause.

### ExUnit Fundamentals

```elixir
defmodule MyApp.UserTest do
  use MyApp.DataCase, async: true

  alias MyApp.Accounts
  import MyApp.Factory

  describe "create_user/1" do
    setup do
      %{valid_attrs: %{email: "test@example.com", name: "Test"}}
    end

    test "with valid attrs creates user", %{valid_attrs: attrs} do
      assert {:ok, user} = Accounts.create_user(attrs)
      assert user.email == attrs.email
    end

    test "with invalid attrs returns error changeset" do
      assert {:error, changeset} = Accounts.create_user(%{})
      assert %{email: ["can't be blank"]} = errors_on(changeset)
    end
  end
end
```

### Key Assertions

```elixir
assert value                              # Truthy
assert {:ok, result} = function()         # Pattern match + extract
assert_raise RuntimeError, fn -> raise "oops" end
assert_receive {:msg, payload}, 1000      # With timeout
refute_receive :unexpected, 100
assert %{email: ["can't be blank"]} = errors_on(changeset)
```

### Setup Patterns

```elixir
# Basic setup
setup do
  user = insert(:user)
  %{user: user}
end

# Named setup
setup :create_user
defp create_user(_), do: %{user: insert(:user)}

# Multiple setups
setup [:create_user, :create_project, :verify_on_exit!]

# start_supervised! — auto-stopped after test
setup do
  pid = start_supervised!({MyWorker, initial_state: []})
  %{worker: pid}
end
```

### Mox Pattern

```elixir
# Define behaviour → mock → test
defmodule MyApp.HTTPClient do
  @callback get(String.t()) :: {:ok, map()} | {:error, term()}
end

# test_helper.exs
Mox.defmock(MyApp.HTTPClient.Mock, for: MyApp.HTTPClient)

# In test
import Mox
setup :verify_on_exit!

test "handles API response" do
  expect(MyApp.HTTPClient.Mock, :get, fn url ->
    assert url == "https://api.example.com/data"
    {:ok, %{status: 200, body: %{"result" => "ok"}}}
  end)
  assert {:ok, _} = MyModule.fetch_data()
end
```

> **Deep dive:** [testing-examples.md](testing-examples.md) — Mox patterns (expect, stub, verify_on_exit!),
> Ecto sandbox (ownership, async mode, allowances), factory patterns (build/insert helpers), LiveView testing
> (render, click, submit), channel testing (join, push, assert_push), Oban testing (perform_job, assert_enqueued),
> OTP process testing (start_supervised!, GenServer testing), property-based testing with StreamData, BAD/GOOD pairs.

## Debugging

### IO.inspect/2 (Pipeline Debugging)

```elixir
data
|> step_one()
|> IO.inspect(label: "after step_one")
|> step_two()
```

### dbg/2 (Elixir 1.14+)

```elixir
# Prints expression + result for each pipeline step
data |> step_one() |> step_two() |> dbg()
```

### IEx.pry

```elixir
require IEx
def problematic_function(data) do
  intermediate = process(data)
  IEx.pry()  # Execution pauses here
  finalize(intermediate)
end
# Run: iex --dbg pry -S mix
```

> **Deep dive:** [debugging-profiling.md](debugging-profiling.md) — break!/1 (IEx breakpoints without code changes),
> Rexbug tracing (safe production tracing with limits), system introspection (memory breakdown, top processes
> by memory/msgq, ETS tables, scheduler utilization, process/atom/port limits), Logger config,
> common diagnostic patterns (message queue buildup, binary memory leaks, GenServer bottlenecks).

## Profiling & Benchmarking

```elixir
# Quick timing
{time_us, result} = :timer.tc(fn -> work() end)

# Benchee
Benchee.run(%{"impl_a" => fn -> a() end, "impl_b" => fn -> b() end})
```

| Profiler | Use | Overhead |
|----------|-----|----------|
| `mix profile.fprof` | Function call detail | High |
| `mix profile.eprof` | Time per function | Moderate |
| `mix profile.cprof` | Call counts | Low |
| `mix profile.tprof` | Unified (OTP 27+) | Low-Moderate |

> **Deep dive:** [debugging-profiling.md](debugging-profiling.md) — mix profile.fprof/eprof/cprof/tprof (decision
> table, options, when to use each), Benchee (inputs, memory, reduction tracking, setup per run), memory
> profiling (VM breakdown, per-process, ETS, binary leak detection), VM/scheduler profiling (utilization,
> run queue, reductions, GC, system limits), :sys tracing, telemetry-based profiling with :telemetry.span.

## Quality Tools

### Credo

```bash
mix credo --strict
mix credo --explain
```

### Dialyzer

```bash
# Add {:dialyxir, "~> 1.4", only: [:dev, :test], runtime: false}
mix dialyzer
```

### Sobelow (Security Analysis)

```bash
# Add {:sobelow, "~> 0.13", only: [:dev, :test], runtime: false}
mix sobelow                    # scan for security vulnerabilities
mix sobelow --config           # generate .sobelow-conf (suppress false positives)
mix sobelow --strict           # exit non-zero on any finding (for CI)
mix sobelow --skip             # list skippable check modules
mix sobelow --details Config   # explain a specific category
```

**What Sobelow catches:** SQL injection, XSS, directory traversal, unsafe deserialization, hardcoded secrets, insecure HTTP, missing CSRF protection, unsafe Ecto raw queries, RCE via `Code.eval_string`, insecure cookie settings, missing security headers. Always run on Phoenix projects before deployment.

### mix format

```elixir
# .formatter.exs
[
  import_deps: [:ecto, :phoenix],
  inputs: ["*.{ex,exs}", "{config,lib,test}/**/*.{ex,exs}"],
  line_length: 120
]
```

## Mix Essentials

### Common Tasks

```bash
mix compile --warnings-as-errors
mix deps.get / mix deps.update --all / mix deps.tree
mix test / mix test path:line / mix test --failed
mix ecto.create / mix ecto.migrate / mix ecto.rollback
mix format / mix format --check-formatted
MIX_ENV=prod mix release
```

## IEx Essentials

### Helpers

```elixir
h Enum.map/2      # Documentation
i value           # Type info
v() / v(3)        # Previous results
r MyModule        # Recompile module
recompile         # Recompile project
s Enum.map/2      # Show @spec
t String          # Show @type
exports Module    # List public functions
```

### .iex.exs Configuration

```elixir
import Ecto.Query
alias MyApp.{Repo, User}

defmodule H do
  def u(id), do: Repo.get(User, id)
end

import H
IO.puts("Helpers: u(id)")
```

### Production Remote Shell

```bash
iex --name debug@localhost --cookie secret --remsh myapp@localhost
```

## Generators

### Ecto

```bash
mix ecto.gen.migration create_users
mix phx.gen.schema Blog.Post posts title:string body:text
```

## Related Skills

### Native Code (NIFs)

| Skill | Scope | Key Features |
|-------|-------|-------------|
| **zigler** | Zig + C NIFs | Inline Zig (`~Z` sigil), automatic type marshalling, `@cImport`/Easy-C for C wrapping, BEAM allocator, dirty/threaded concurrency, resources, Nerves cross-compilation. v0.15.2 (`E-xyza/zigler`). |
| **[rust-nif](../rust-nif/SKILL.md)** | Rust + C++ NIFs | Rustler library, separate Rust crate, `#[rustler::nif]` macros, Encoder/Decoder traits, `ResourceArc`, precompiled NIFs, tokio async. |

**When to use NIFs:** CPU-intensive operations >1 microsecond, wrapping existing C/Rust libraries, binary manipulation, tight compute loops. **When NOT to:** I/O-bound work, fault-tolerant processing, simple data transforms.

**Decision:** Use **zigler** for Zig code or wrapping C libraries (best-in-class C integration). Use **rust-nif** for Rust code or wrapping C++ libraries (mature ecosystem, memory safety guarantees).

### Web Frameworks

- **[phoenix](../phoenix/SKILL.md)** — Phoenix Framework architecture, Plug, contexts, channels, PubSub, security, forms, Tailwind, and deployment.
  Key: use contexts as API boundaries, never put business logic in controllers, use PubSub for real-time broadcasts.
- **[phoenix-liveview](../phoenix-liveview/SKILL.md)** — LiveView lifecycle, components, forms, streams, uploads, hooks, and real-time patterns.
  Key: assign all variables in `mount/3`, use streams for large collections, minimize socket assigns.
- **[ash](../ash/SKILL.md)** — Ash Framework resources, domains, actions, policies, and extensions (AshPhoenix, AshAuthentication, AshPostgres, AshStateMachine, AshGraphQL).
  Key: declarative resource definitions, policy-based authorization, domain-driven contexts.

### Architecture & Patterns

- **[state-machine](../state-machine/SKILL.md)** — gen_statem, GenStateMachine, AshStateMachine, FSM design patterns and testing.
  Key: use gen_statem for complex state machines, AshStateMachine for resource lifecycle, test state transitions exhaustively.
- **[igniter](../igniter/SKILL.md)** — Code generation, project patching, installers, AST manipulation, upgrade tasks.
  Key: use for creating generators/installers, AST-safe code modification, composable project patches.

### Embedded & IoT

- **[nerves](../nerves/SKILL.md)** — Nerves firmware, VintageNet networking, OTA updates, hardware access, device trees, Buildroot customization.
  Key: use VintageNet for networking, validate firmware before confirming, always configure heart monitoring.
- **[modbus](../modbus/SKILL.md)** — Modbus RTU/TCP industrial protocol with Elixir, Rust, C implementations.
  Key: RS-485 wiring, timing constraints, Elixir GenServer-based Modbus master/slave patterns.

### Desktop & Deployment

- **[tauri-elixir](../tauri-elixir/SKILL.md)** — Desktop apps combining Tauri (Rust/WebView) with Phoenix LiveView, packaged as single binaries via Burrito.
  Key: Rust backend + LiveView UI, native system access through Tauri commands.
- **[elixir-deployment](../elixir-deployment/SKILL.md)** — Mix releases, Docker, cloud providers, Kubernetes, security, observability, and production patterns.
  Key: always use Mix releases for production, configure runtime.exs for env vars, use health check endpoints.
### Multimedia & Notebooks

- **[membrane](../membrane/SKILL.md)** — Multimedia streaming pipelines, WebRTC, HLS, RTMP, camera capture, video encoding, Phoenix integration.
  Key: pipeline-based architecture, elements connected by pads, backpressure-aware processing.
- **[livebook](../livebook/SKILL.md)** — Interactive notebooks, smart cells, Kino visualizations, VegaLite charts, OTP introspection, Nerves integration.
  Key: use smart cells for databases/charts, Kino for interactive widgets, great for prototyping and documentation.
