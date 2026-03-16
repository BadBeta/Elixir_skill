# Elixir Data Structures Reference

> Supporting reference for the [Elixir skill](SKILL.md). Contains selection guide, lists (linked lists, IO lists, charlists), maps (access, update, nested, patterns), tuples (tagged tuples, ETS), keyword lists, MapSet, structs (define, construct, pattern match, update, pipelines, accumulator, protocols, nesting, GenServer state, anti-patterns), embedded schemas, map operations, function captures, and binary pattern matching.

## Data Structures

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
| Graph/dependencies | `:digraph` | Mutable directed graph with path algorithms |
| Sorted unique set | `:ordsets` | Sorted list, good for small ordered sets |
| Fast concurrent reads | ETS | In-memory table, cross-process access |

### Lists — Linked Lists

Lists are singly-linked — O(1) prepend, O(n) append/access by index. This shapes all idiomatic patterns:

```elixir
# Construction
[]                            # Empty
[1, 2, 3]                    # Literal
[head | tail]                 # Cons cell: head is element, tail is rest of list
[1 | [2 | [3 | []]]]        # Same as [1, 2, 3] — this IS the internal structure

# Pattern matching — the core list idiom
[first | rest] = [1, 2, 3]          # first=1, rest=[2,3]
[first, second | rest] = [1, 2, 3]  # first=1, second=2, rest=[3]
[only] = [42]                        # Matches single-element list
[] = []                              # Matches empty list

# Multi-clause with lists — the recursive pattern
def sum([]), do: 0
def sum([head | tail]), do: head + sum(tail)

# Prepend is O(1) — ALWAYS build lists by prepending
list = [new_item | existing_list]

# Append is O(n) — avoid in loops!
# BAD: O(n) on every iteration = O(n^2) total
Enum.reduce(items, [], fn item, acc -> acc ++ [item] end)
# GOOD: prepend then reverse = O(n) total
items |> Enum.reduce([], fn item, acc -> [item | acc] end) |> Enum.reverse()
# BEST: just use Enum.map if transforming
Enum.map(items, &transform/1)

# List operations
length(list)                  # O(n) — must traverse entire list
Enum.at(list, 0)             # O(n) — no index access
List.first(list)              # O(1) — just head
List.last(list)               # O(n) — must traverse
hd(list)                      # O(1) — head, raises on empty
tl(list)                      # O(1) — tail, raises on empty

# List-specific functions (not in Enum)
List.flatten([[1, [2]], 3])  # => [1, 2, 3]
List.zip([[1, 2], [:a, :b]])  # => [{1, :a}, {2, :b}]
List.foldl([1, 2, 3], 0, &+/2)  # left fold (same as Enum.reduce)
List.foldr([1, 2, 3], 0, &+/2)  # right fold (processes right-to-left)
List.insert_at(list, 2, :x)  # O(n) insert
List.delete(list, :item)     # Delete first occurrence
List.update_at(list, 0, &(&1 + 1))  # Update at index (O(n))
[1, 2, 3] -- [2]            # Difference: [1, 3]
[1, 2] ++ [3, 4]            # Concatenation: [1, 2, 3, 4] — O(length of left list)

# Charlists vs Strings
~c"hello"                     # Charlist: list of codepoints [104, 101, ...]
"hello"                       # Binary string (UTF-8)
# IEx displays [104, 101, ...] as ~c"hello" — it's still a list!
# Use charlists only for Erlang interop that requires them
:io.format(~c"~p~n", [value])  # Erlang :io expects charlists
```

**IO Lists** — the fastest way to build output:

```elixir
# IO lists are nested lists of binaries/integers — no copying until final output
iodata = ["Hello", ?\s, ["world", ?!]]
IO.iodata_to_binary(iodata)  # => "Hello world!" — single allocation

# BAD: string concatenation in loops (copies on every <>)
Enum.reduce(rows, "", fn row, acc -> acc <> format_row(row) <> "\n" end)

# GOOD: IO list — zero-copy accumulation, single flatten at the end
rows
|> Enum.map(fn row -> [format_row(row), ?\n] end)
|> IO.iodata_to_binary()

# IO functions accept IO lists directly — no need to flatten
File.write!("out.txt", Enum.map(rows, &[&1, ?\n]))
Plug.Conn.send_resp(conn, 200, ["<h1>", title, "</h1>", body])
```

### Maps — Hash Maps

```elixir
# Creation
%{}                                    # Empty
%{key: "value"}                        # Atom keys
%{"key" => "value"}                    # String keys (external data)
Map.new([{:a, 1}, {:b, 2}])          # From key-value pairs
Map.new(users, &{&1.id, &1})          # From enumerable with transform

# Access patterns — choose based on intent
map.key                                # Atom key: raises KeyError if missing (assertive)
map[:key]                              # Atom key: returns nil if missing (lenient)
map["key"]                             # String key: returns nil if missing
Map.get(map, key, default)             # With explicit default
Map.fetch(map, key)                    # {:ok, val} or :error (pattern matchable)
Map.fetch!(map, key)                   # Value or raise

# Update (returns new map — never mutates)
Map.put(map, key, value)               # Set (create or overwrite)
Map.put_new(map, key, value)           # Set only if key missing
Map.put_new_lazy(map, key, fun)        # Lazy default (avoid expensive computation)
%{map | key: new_value}                # Update EXISTING key only (raises if missing!)
Map.update(map, key, default, fun)     # Update with function or insert default
Map.update!(map, key, fun)             # Update with function (raises if missing)
Map.merge(map1, map2)                  # Merge (map2 wins on conflict)
Map.merge(map1, map2, fn _k, v1, v2 -> v1 + v2 end)  # Merge with resolver

# Delete & filter
Map.delete(map, key)                   # Remove key (no error if missing)
Map.drop(map, [:a, :b])               # Remove multiple keys
Map.take(map, [:a, :b])               # Keep only these keys
Map.pop(map, key)                      # {value, rest_map}
Map.reject(map, fn {_k, v} -> is_nil(v) end)  # Filter out entries
Map.filter(map, fn {_k, v} -> v > 0 end)      # Keep matching entries

# Nested access — works on maps, keyword lists, structs
get_in(data, [:users, 0, :name])                    # Nested get
put_in(data, [:settings, :theme], "dark")            # Nested put
update_in(data, [:stats, :count], & &1 + 1)          # Nested update
pop_in(data, [:temp, :cache])                        # Nested pop
get_in(data, [:users, Access.all(), :name])           # All user names
get_in(data, [:items, Access.filter(&(&1.active))])   # Filtered access

# Transform — build new map
for {k, v} <- map, into: %{}, do: {k, transform(v)}
Map.new(map, fn {k, v} -> {k, transform(v)} end)   # Same, functional style
```

**Map patterns in functions:**

```elixir
# Match on specific keys (partial match — extra keys are OK)
def greet(%{name: name}), do: "Hello #{name}"

# Match on key presence + guard
def process(%{status: status} = record) when status in [:pending, :active] do
  %{record | processed_at: DateTime.utc_now()}
end

# Atom keys vs string keys — be consistent per context
# Internal data: atom keys (fast comparison, dot access)
%{user_id: 1, role: :admin}
# External/JSON data: string keys (safe, no atom exhaustion)
%{"user_id" => 1, "role" => "admin"}

# Converting between key types
Map.new(string_map, fn {k, v} -> {String.to_existing_atom(k), v} end)
```

### Tuples — Fixed-Size Containers

Tuples are stored contiguously — O(1) element access, but updating creates a full copy. Use for small, fixed-size groupings (typically 2-4 elements).

```elixir
# Tagged tuples — Elixir's primary result/control flow mechanism
{:ok, value}                  # Success
{:error, reason}              # Failure
{:error, :not_found}          # Typed failure
{:error, %Ecto.Changeset{}}   # Rich failure with details
{:noreply, state}             # GenServer continue
{:reply, response, state}     # GenServer respond
{:halt, acc}                  # Stop reduce_while/Stream.transform
{:cont, acc}                  # Continue reduce_while

# Pattern matching — the primary way to work with tuples
case File.read("config.json") do
  {:ok, contents} -> Jason.decode!(contents)
  {:error, :enoent} -> %{}       # File not found — use defaults
  {:error, reason} -> raise "Config error: #{inspect(reason)}"
end

# In function heads
def handle_result({:ok, value}), do: value
def handle_result({:error, reason}), do: raise "Failed: #{reason}"

# Tuples as lightweight records (when a struct is overkill)
# Common in Erlang interop and internal data
point = {10, 20}                    # {x, y}
color = {255, 128, 0, 1.0}         # {r, g, b, alpha}
range = {start, finish}             # {from, to}

# ETS stores tuples — first element is typically the key
:ets.insert(table, {user_id, username, email})
:ets.lookup(table, user_id)  # => [{user_id, username, email}]

# Tuple operations
elem(tuple, 0)                # O(1) access by index
put_elem(tuple, 1, :new_val)  # Returns new tuple (copies all elements)
tuple_size(tuple)              # O(1) size
Tuple.append(tuple, :val)     # Add at end (returns new tuple)
Tuple.insert_at(tuple, 1, :v) # Insert at position
Tuple.delete_at(tuple, 0)     # Remove at position

# NEVER use tuples as variable-length collections
# BAD: tuple as list replacement
items = {:a, :b, :c, :d, :e}  # Can't pattern match head/tail, no Enum support
# GOOD: use a list
items = [:a, :b, :c, :d, :e]  # Supports Enum, pattern matching, recursion
```

**When to use tuples vs other structures:**

| Size/Use | Data structure |
|---|---|
| 2-4 elements, fixed shape | Tuple `{:ok, value}` |
| Named fields, compile-time | Struct `%User{}` |
| Variable length, ordered | List `[1, 2, 3]` |
| Key-value lookup | Map `%{key: val}` |

### Keyword Lists & Options

```elixir
# Keyword lists are lists of {atom, value} tuples
# Used primarily for function options
[name: "Jo", age: 30] == [{:name, "Jo"}, {:age, 30}]

# Allow duplicate keys (unlike maps)
[a: 1, a: 2]  # Valid! First match wins with Keyword.get

# Common Operations
Keyword.get(opts, :key, default)     # Get with default
Keyword.fetch!(opts, :key)           # Get or raise
Keyword.put(opts, :key, value)       # Set (replaces first)
Keyword.merge(opts1, opts2)          # Merge
Keyword.take(opts, [:a, :b])         # Keep subset
Keyword.validate!(opts, [:a, :b])    # Validate allowed keys (Elixir 1.13+)
Keyword.pop(opts, :key, default)     # Extract and return rest

# Options pattern — last argument keyword list drops brackets
def start(opts \\ []) do
  host = Keyword.get(opts, :host, "localhost")
  port = Keyword.get(opts, :port, 4000)
  # ...
end
# Callers:
start(host: "example.com", port: 8080)
start([{:host, "example.com"}])  # Equivalent, explicit

# Keyword vs Map for options:
# Use Keyword when: order matters, duplicate keys needed, function options
# Use Map when: fast lookup needed, no duplicates, data (not config)
```

### MapSet — Unique Value Collections

```elixir
set = MapSet.new([1, 2, 3, 2])     # => MapSet.new([1, 2, 3])
MapSet.member?(set, 2)              # => true
MapSet.put(set, 4)                  # => MapSet.new([1, 2, 3, 4])
MapSet.delete(set, 1)               # => MapSet.new([2, 3])

# Set operations
MapSet.union(set_a, set_b)          # All elements from both
MapSet.intersection(set_a, set_b)   # Elements in both
MapSet.difference(set_a, set_b)     # Elements in A but not B
MapSet.disjoint?(set_a, set_b)      # True if no overlap
MapSet.subset?(small, large)        # True if small ⊆ large
MapSet.equal?(set_a, set_b)         # True if same elements

# Common patterns
# Deduplicate while preserving type
tags = MapSet.new(user_tags)
MapSet.to_list(tags)

# Fast membership testing in guards/conditions
allowed = MapSet.new([:admin, :moderator])
if MapSet.member?(allowed, user.role), do: :granted, else: :denied

# Build set from pipeline
active_ids =
  users
  |> Enum.filter(& &1.active)
  |> Enum.map(& &1.id)
  |> MapSet.new()
```

### Structs

Structs are named maps with compile-time guarantees. They are central to idiomatic Elixir — used as function arguments, pipeline state, GenServer state, and domain models.

#### Defining Structs

```elixir
defmodule User do
  @enforce_keys [:email]  # Required at creation time
  defstruct [:name, :email, age: 18]  # With default value

  @type t :: %__MODULE__{
    name: String.t() | nil,
    email: String.t(),
    age: non_neg_integer()
  }
end

# Creation - email required, age defaults to 18
%User{email: "a@b.com"}  # => %User{name: nil, email: "a@b.com", age: 18}
%User{name: "Jo"}        # => ArgumentError: :email is required
```

**ALWAYS include `@type t`** for every struct. Use `@enforce_keys` sparingly — only for fields that truly must be set at creation. For complex structs, provide a constructor function instead.

**Struct vs Map:**
- Structs enforce keys at compile time (typos caught early)
- No `Access` protocol — use `user.field` not `user[:field]`
- Pattern matching verifies struct type (`%User{}` won't match a plain map)
- Include `__struct__` key for type identification

#### Constructor Functions (new/build)

Don't rely on bare `%MyStruct{}` — provide constructor functions that validate, compute derived fields, and return consistent results:

```elixir
defmodule Order do
  defstruct [:id, :items, :total, :status, created_at: nil]

  @type t :: %__MODULE__{
    id: String.t(),
    items: [item()],
    total: Decimal.t(),
    status: :pending | :confirmed | :shipped,
    created_at: DateTime.t()
  }

  # Simple constructor — computes derived fields
  def new(items) when is_list(items) do
    %__MODULE__{
      id: generate_id(),
      items: items,
      total: Enum.reduce(items, Decimal.new(0), &Decimal.add(&2, &1.price)),
      status: :pending,
      created_at: DateTime.utc_now()
    }
  end

  # Constructor from external input — returns ok/error tuple
  def build(params) when is_map(params) do
    with {:ok, items} <- validate_items(params["items"]),
         {:ok, _} <- validate_address(params["address"]) do
      {:ok, new(items)}
    end
  end
end
```

**Constructor from keyword options** — use `struct!/2` to parse keywords with enforcement:

```elixir
# Pattern from Phoenix: parse keyword opts into struct fields
def new!(opts) when is_list(opts) do
  scope = struct!(__MODULE__, opts)
  alias_name = scope.module |> Module.split() |> List.last() |> String.to_atom()
  %{scope | alias: alias_name}
end
```

**When to use which pattern:**
| Pattern | Use when |
|---|---|
| `%Mod{field: val}` | Internal code, all fields known at compile time |
| `new/1` | Computing derived fields, setting defaults beyond `defstruct` |
| `build/1` returning `{:ok, t} \| {:error, term}` | External/user input that may be invalid |
| `struct!/2` from keywords | Parsing option lists (raises on unknown keys) |

#### Pattern Matching on Structs

```elixir
# Match struct type in function head
def process(%User{age: age}) when age >= 18, do: :adult
def process(%User{}), do: :minor

# Extract only what you need for the guard, bind rest
def drive(%User{age: age} = user) when age >= 18 do
  Logger.info("#{user.name} is driving")
  {:ok, user}
end

# Match multiple struct types
def handle(%mod{name: name}) when mod in [User, Admin] do
  "#{mod}: #{name}"
end

# Assertive matching — crash on unexpected struct type
def send_email(%User{email: email}) do
  # If called with %Admin{}, this crashes immediately — good!
  Mailer.send(email)
end
```

#### Struct Updates (Immutable Transforms)

The update syntax `%{struct | field: value}` creates a new struct — it never mutates:

```elixir
# Single field update
user = %{user | name: "New Name"}

# Multiple field update
order = %{order | status: :shipped, shipped_at: DateTime.utc_now()}

# Nested map field — use Map.merge or Map.put for map-valued fields
socket = %{socket | assigns: Map.merge(socket.assigns, %{user: user, flash: "Done"})}

# IMPORTANT: update syntax only works for EXISTING keys
%User{} |> Map.put(:nonexistent, "val")  # Works (plain map operation, loses struct)
%{%User{email: "a"} | nonexistent: "v"}  # ** KeyError — good, catches typos!
```

#### Struct Transformation Pipelines

The most idiomatic Elixir pattern: thread a struct through a series of functions, each returning an updated struct.

```elixir
# Pipeline of struct transforms — each function takes and returns the struct
defmodule OrderProcessor do
  def process(%Order{} = order) do
    order
    |> validate_inventory()
    |> apply_discounts()
    |> calculate_tax()
    |> set_status(:confirmed)
  end

  defp validate_inventory(%Order{items: items} = order) do
    # Check stock, raise if unavailable
    Enum.each(items, &check_stock!/1)
    order
  end

  defp apply_discounts(%Order{items: items, total: total} = order) do
    discount = calculate_discount(items)
    %{order | total: Decimal.sub(total, discount)}
  end

  defp calculate_tax(%Order{total: total} = order) do
    tax = Decimal.mult(total, Decimal.new("0.25"))
    %{order | total: Decimal.add(total, tax)}
  end

  defp set_status(%Order{} = order, status) do
    %{order | status: status}
  end
end
```

**Pipeline with error handling** — when steps can fail, use `with` or result tuples:

```elixir
# with-chain for fallible pipeline
def checkout(%Cart{} = cart) do
  with {:ok, order} <- Cart.to_order(cart),
       {:ok, order} <- validate(order),
       {:ok, order} <- charge_payment(order),
       {:ok, order} <- send_confirmation(order) do
    {:ok, %{order | status: :completed}}
  end
end

# Or: thread {:ok, struct} through pipeline functions
def process(order) do
  {:ok, order}
  |> then_ok(&validate/1)
  |> then_ok(&charge/1)
  |> then_ok(&confirm/1)
end

defp then_ok({:ok, val}, fun), do: fun.(val)
defp then_ok({:error, _} = err, _fun), do: err
```

#### Struct as Accumulator (Credo/Phoenix Pattern)

Use a struct to accumulate state across pipeline stages — this is how Credo's `Execution` and Phoenix's `Conn` work:

```elixir
defmodule Pipeline do
  defstruct [
    :input,
    :result,
    assigns: %{},     # extensible key-value storage
    private: %{},      # internal framework state (not user-facing)
    halted: false,
    errors: []
  ]

  @type t :: %__MODULE__{
    input: term(),
    result: term() | nil,
    assigns: map(),
    private: map(),
    halted: boolean(),
    errors: [term()]
  }

  def new(input), do: %__MODULE__{input: input}

  # Assign arbitrary key-value pairs (Plug.Conn pattern)
  def assign(%__MODULE__{} = pipeline, key, value) do
    %{pipeline | assigns: Map.put(pipeline.assigns, key, value)}
  end

  # Halt pipeline processing
  def halt(%__MODULE__{} = pipeline, reason) do
    %{pipeline | halted: true, errors: [reason | pipeline.errors]}
  end

  # Run a pipeline stage, skip if halted
  def run(%__MODULE__{halted: true} = pipeline, _fun), do: pipeline
  def run(%__MODULE__{} = pipeline, fun) when is_function(fun, 1), do: fun.(pipeline)
end

# Usage — thread struct through stages
Pipeline.new(raw_data)
|> Pipeline.run(&parse/1)
|> Pipeline.run(&validate/1)
|> Pipeline.run(&transform/1)
|> Pipeline.assign(:processed_at, DateTime.utc_now())
```

#### Structs with Protocols

> See also: **Protocols** section in [SKILL.md](SKILL.md) for full protocol definition, implementation, @derive, and dispatch details.

```elixir
# @derive for common protocols — declare before defstruct
defmodule User do
  @derive {Jason.Encoder, only: [:id, :name, :email]}  # selective JSON encoding
  @derive {Phoenix.Param, key: :username}               # URL param generation
  @derive {Inspect, only: [:id, :name]}                 # hide sensitive fields in logs
  defstruct [:id, :name, :email, :password_hash, :username]
end

# Custom Inspect for sensitive structs (more control than @derive)
defimpl Inspect, for: Credentials do
  def inspect(%Credentials{username: username}, opts) do
    Inspect.Algebra.concat(["#Credentials<", username, ", password: [REDACTED]>"])
  end
end

# Custom String.Chars for struct-to-string conversion
defimpl String.Chars, for: User do
  def to_string(%User{name: name, email: email}), do: "#{name} <#{email}>"
end
```

#### Nested Structs and Deep Updates

```elixir
defmodule Address do
  defstruct [:street, :city, :zip]
  @type t :: %__MODULE__{street: String.t(), city: String.t(), zip: String.t()}
end

defmodule Company do
  defstruct [:name, :address, employees: []]
  @type t :: %__MODULE__{name: String.t(), address: Address.t(), employees: [User.t()]}
end

# Update nested struct — use put_in with Access-style paths
company = %Company{name: "Acme", address: %Address{street: "1st", city: "Oslo", zip: "0001"}}

# These work because put_in/update_in work on maps (structs are maps):
put_in(company, [Access.key(:address), Access.key(:city)], "Bergen")
update_in(company, [Access.key(:address), Access.key(:zip)], &String.upcase/1)

# Or use direct struct update (clearer for shallow nesting):
%{company | address: %{company.address | city: "Bergen"}}

# For deep nesting, write explicit helper functions:
defmodule Company do
  def update_address(%__MODULE__{} = company, fun) when is_function(fun, 1) do
    %{company | address: fun.(company.address)}
  end
end

company |> Company.update_address(&%{&1 | city: "Bergen"})
```

#### GenServer State as Struct

```elixir
defmodule CacheServer do
  use GenServer

  defstruct [
    :name,
    store: %{},
    stats: %{hits: 0, misses: 0},
    max_size: 1000,
    ttl: :timer.minutes(5)
  ]

  @type t :: %__MODULE__{
    name: atom(),
    store: %{term() => {term(), integer()}},
    stats: %{hits: non_neg_integer(), misses: non_neg_integer()},
    max_size: pos_integer(),
    ttl: pos_integer()
  }

  # Constructor validates and sets defaults
  def start_link(opts) do
    name = Keyword.fetch!(opts, :name)
    state = struct!(__MODULE__, opts)
    GenServer.start_link(__MODULE__, state, name: name)
  end

  @impl true
  def init(%__MODULE__{} = state), do: {:ok, state}

  @impl true
  def handle_call({:get, key}, _from, %__MODULE__{} = state) do
    case Map.get(state.store, key) do
      {value, expires} when expires > System.monotonic_time(:millisecond) ->
        {:reply, {:ok, value}, %{state | stats: bump_hits(state.stats)}}
      _ ->
        {:reply, :miss, %{state | stats: bump_misses(state.stats)}}
    end
  end

  defp bump_hits(stats), do: %{stats | hits: stats.hits + 1}
  defp bump_misses(stats), do: %{stats | misses: stats.misses + 1}
end
```

#### Common Anti-Patterns with Structs

```elixir
# BAD: Using Map functions that strip struct type
map = Map.put(%User{email: "a"}, :name, "Jo")  # Returns plain map, not %User{}!

# GOOD: Use struct update syntax to preserve type
user = %{%User{email: "a"} | name: "Jo"}

# BAD: Dynamic field access on structs
user[:name]  # UndefinedFunctionError — structs don't implement Access

# GOOD: Use dot notation
user.name

# BAD: Creating structs with string keys from external input
%User{"name" => "Jo"}  # Won't work — struct keys are atoms

# GOOD: Use a constructor that casts
User.build(%{"name" => "Jo"})  # or use Ecto changeset for casting

# BAD: Huge struct with 30+ fields in one module
defstruct [:a, :b, :c, :d, ...]  # Hard to reason about

# GOOD: Compose smaller structs or use map fields for extensibility
defstruct [:core_field, config: %Config{}, assigns: %{}, private: %{}]
```

### Embedded Schemas (Validation Without Database)

Use `embedded_schema` for data validation without database tables:

```elixir
defmodule CreateUserCommand do
  use Ecto.Schema
  import Ecto.Changeset

  @primary_key false
  embedded_schema do
    field :email, :string
    field :name, :string
    field :age, :integer
  end

  @doc "Validate params, return struct or error changeset"
  def validate(params) do
    %__MODULE__{}
    |> cast(params, [:email, :name, :age])
    |> validate_required([:email])
    |> validate_format(:email, ~r/@/)
    |> validate_number(:age, greater_than: 0)
    |> apply_action(:validate)  # Returns {:ok, struct} or {:error, changeset}
  end
end

# Usage - fail fast before expensive operations
case CreateUserCommand.validate(params) do
  {:ok, command} -> Accounts.create_user(command)
  {:error, changeset} -> {:error, changeset}
end
```

**Use Cases:**
- API input validation (command pattern)
- LiveView forms without database backing
- Configuration validation
- External API response parsing

**Key Point:** Call `apply_action/2` to get `{:ok, struct}` or `{:error, changeset}`. Without it, `changeset.valid?` works but errors won't display in forms.

### Map Operations

```elixir
# Create
%{}                                    # Empty
%{key: "value"}                        # Atom keys
%{"key" => "value"}                    # String keys
Map.new([{:a, 1}, {:b, 2}])           # From key-value pairs
Map.new(users, &{&1.id, &1})          # From enumerable with function

# Read
map.key                                # Atom key (raises if missing)
map[:key]                              # Atom key (nil if missing)
map["key"]                             # String key (nil if missing)
Map.get(map, key, default)             # With default
Map.fetch(map, key)                    # {:ok, val} or :error
Map.fetch!(map, key)                   # Value or raise
Map.has_key?(map, key)                 # Boolean
Map.keys(map)                          # List of keys
Map.values(map)                        # List of values

# Update (returns new map)
Map.put(map, key, value)               # Set (create or overwrite)
Map.put_new(map, key, value)           # Set only if key missing
Map.put_new_lazy(map, key, fun)        # Lazy default
%{map | key: new_value}                # Update existing (raises if missing!)
Map.update(map, key, default, fun)     # Update with function or insert default
Map.update!(map, key, fun)             # Update with function (raises if missing)
Map.merge(map1, map2)                  # Merge (map2 wins on conflict)
Map.merge(map1, map2, fn _k, v1, v2 -> v1 + v2 end)  # Merge with resolver

# Delete
Map.delete(map, key)                   # Remove key (no error if missing)
Map.drop(map, [:a, :b])               # Remove multiple keys
Map.take(map, [:a, :b])               # Keep only these keys
Map.pop(map, key)                      # {value, rest_map}

# Nested Access (works on maps, keyword lists, structs)
get_in(data, [:users, 0, :name])                    # Nested get
put_in(data, [:settings, :theme], "dark")            # Nested put
update_in(data, [:stats, :count], & &1 + 1)          # Nested update
pop_in(data, [:temp, :cache])                        # Nested pop
get_in(data, [:users, Access.all(), :name])           # All user names
get_in(data, [:items, Access.filter(&(&1.active))])   # Filtered access

# Transform - build new map
for {k, v} <- map, into: %{}, do: {k, transform(v)}
```

### Function References & Capture

```elixir
# Capture operator (&)
&Module.function/arity          # Reference to named function
&(&1 + 1)                      # Short anonymous function (one arg)
&(&1 + &2)                     # Multiple args
&{&1, &2}                      # Returns tuple

# When to use which:
# Named function reference - prefer when function already exists
Enum.map(list, &String.upcase/1)
Enum.filter(list, &valid?/1)

# Short capture - prefer for simple one-liners
Enum.map(list, & &1 * 2)
Enum.filter(list, & &1 > 0)

# Full anonymous function - prefer for multi-line or pattern matching
Enum.map(list, fn %{name: name, age: age} ->
  "#{name} is #{age}"
end)

# Common captures
&to_string/1                    # Kernel.to_string
&is_nil/1                       # Kernel.is_nil
&elem(&1, 0)                    # Get first tuple element
&(&1)                           # Identity function
```

### Binary Pattern Matching

Parse binary protocols, file formats, network packets:

```elixir
# Fixed-size fields
<<version::8, type::8, length::16, payload::binary>> = packet

# Bit-level parsing
<<flag::1, reserved::3, value::4>> = <<0b1010_0111>>

# String matching — prefix only
<<"HTTP/", version::binary-size(3), rest::binary>> = "HTTP/1.1 200 OK"

# UTF-8 codepoint extraction
<<first::utf8, rest::binary>> = "héllo"  # first = 104 (h)

# Common modifiers
<<x::big-integer-size(32)>>          # 32-bit big-endian integer
<<x::little-float-size(64)>>         # 64-bit little-endian float
<<data::binary-size(10)>>            # Exactly 10 bytes
<<head::binary-size(4), _::binary>>  # First 4 bytes
```

**Advanced binary patterns:**

```elixir
# Pinned size — dynamic segment length from variable
size = byte_size(prefix)
<<prefix::binary-size(^size), rest::binary>> = data

# Multi-clause binary dispatch — UTF-8 cascade (from Elixir stdlib)
defp process(<<char, rest::binary>>) when char < 128, do: ...      # ASCII byte
defp process(<<char::utf8, rest::binary>>), do: ...                 # Multi-byte UTF-8
defp process(<<a::4, b::4, rest::binary>>), do: ...                 # Nibble parsing
defp process(<<>>), do: ...                                          # Empty binary

# Reinterpret bytes as nibbles (for hex encoding)
<<hi::4, lo::4>> = <<byte>>

# Binary comprehension — iterate over bytes
for <<byte <- binary>>, into: <<>>, do: <<byte + 1>>
```

