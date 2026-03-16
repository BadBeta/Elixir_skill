# Elixir Code Style & Formatter Reference

> Supporting reference for the [Elixir skill](SKILL.md). Contains .formatter.exs configuration, migration options, formatter enforcement rules, mix format commands, Credo style checks catalog, readable code patterns (pipelines, guards, pattern match ordering, with-chains, multi-line calls, import organization, naming, conditionals, Ecto.Multi, pure/side-effect separation), and BAD/GOOD pairs.

## Code Style & Formatter

### The Formatter (`mix format`)

#### .formatter.exs Configuration

```elixir
# .formatter.exs (project root)
[
  inputs: ["{mix,.formatter}.exs", "{config,lib,test}/**/*.{ex,exs}"],
  excludes: ["test/fixtures/**/*.{ex,exs}"],   # since v1.19

  # Target line length (soft limit — formatter may exceed when no break point exists)
  line_length: 98,  # default

  # DSL functions that skip parentheses
  locals_without_parens: [
    my_macro: 1,
    my_macro: 2,
    my_dsl: :*      # :* = all arities
  ],

  # Pull locals_without_parens from dependencies
  import_deps: [:phoenix, :ecto, :ecto_sql],

  # Convert all keyword syntax to do-end blocks (convergent — won't convert back)
  force_do_end_blocks: false,  # default

  # Formatter plugins (since v1.13)
  plugins: [Phoenix.LiveView.HTMLFormatter],

  # Subdirectories with their own .formatter.exs (configs NOT inherited)
  subdirectories: ["apps/*"]
]
```

#### Exporting for Library Authors

Libraries export their formatting config so consumers can `import_deps`:

```elixir
# In your library's .formatter.exs
[
  inputs: ["{mix,.formatter}.exs", "{config,lib,test}/**/*.{ex,exs}"],
  locals_without_parens: [my_macro: 1, my_dsl: :*],
  export: [
    locals_without_parens: [my_macro: 1, my_dsl: :*]
  ]
]
```

#### Migration Options (Since 1.18+)

The formatter can automatically migrate deprecated syntax:

```bash
mix format --migrate
```

| Migration | Version | What it does |
|-----------|---------|-------------|
| `migrate_bitstring_modifiers` | 1.18+ | `<<x::binary()>>` → `<<x::binary>>` |
| `migrate_charlists_as_sigils` | 1.18+ | `'foo'` → `~c"foo"` |
| `migrate_unless` | 1.18+ | `unless x, do: y` → `if !x, do: y` |
| `migrate_call_parens_on_pipe` | 1.19+ | `foo \|> bar` → `foo \|> bar()` |

Enable all at once with `migrate: true` in `.formatter.exs` or `--migrate` flag.

#### What the Formatter Enforces

**Spacing and indentation:**
- 2-space indentation (always)
- Spaces around binary operators: `a + b`, `x |> y`
- Space after `#` in comments: `# comment` (not `#comment`)
- No trailing whitespace
- Single trailing newline at end of file
- Consecutive blank lines squeezed to one

**Parentheses:**
- Added to all function calls except `locals_without_parens`
- Removed from `if`/`unless`/`case`/`cond` conditions
- Added when mixing operators with different precedence
- Zero-arity function definitions: no parens (`def foo do`)

**Line breaking:**
- Breaks at `line_length` boundary when possible
- Multi-line lists/maps/tuples get one element per line with trailing comma position
- Preserves user's choice of `do:` keyword vs `do/end` block (unless `force_do_end_blocks`)
- Preserves intentional blank lines within blocks (squeezed to max 1)

**Number formatting:**
- Large integers get underscores: `1000000` → `1_000_000` (6+ digits)
- Hex digits uppercased: `0xabcd` → `0xABCD`

**What the formatter does NOT enforce:**
- Variable/function naming conventions
- Module organization order
- Pipe usage patterns
- Documentation presence
- Code complexity

#### mix format Commands

```bash
mix format                     # Format all files
mix format --check-formatted   # CI check — exit 1 if unformatted
mix format --dry-run           # Check without writing
mix format --force             # Ignore cache, reformat everything
mix format lib/my_file.ex      # Format specific file
mix format --dot-formatter path/.formatter.exs  # Custom config
```

### Credo Style Checks

Credo enforces semantic style rules the formatter cannot. Key checks organized by impact:

#### High-Priority (Community Consensus)

| Check | Rule |
|-------|------|
| `FunctionNames` | Functions must be `snake_case` |
| `VariableNames` | Variables must be `snake_case` |
| `ModuleNames` | Modules must be `PascalCase` |
| `ModuleAttributeNames` | Attributes must be `snake_case` |
| `PredicateFunctionNames` | Public: end with `?`; Guard macros: start with `is_`, no `?` |
| `PreferUnquotedAtoms` | `:foo` not `:"foo"` |
| `ParenthesesInCondition` | No parens in `if`/`unless` conditions |
| `Semicolons` | Never use `;` — use line breaks |
| `ExceptionNames` | Consistent suffix (e.g., `Error`) |
| `SpaceAroundOperators` | `a + b` not `a+b` |
| `TabsOrSpaces` | Spaces only (2-space indent) |

#### Readability (Recommended)

| Check | Rule |
|-------|------|
| `SinglePipe` | Don't pipe single calls: `list \|> length()` → `length(list)` |
| `ModuleDoc` | Every module needs `@moduledoc` or `@moduledoc false` |
| `AliasOrder` | Alphabetize aliases within groups |
| `SeparateAliasRequire` | Group all `alias` together, all `require` together |
| `LargeNumbers` | Use underscores: `1_000_000` not `1000000` |
| `RedundantBlankLines` | Max 1 consecutive blank line |
| `PipeIntoAnonymousFunctions` | Use `then/1` instead of `\|> (fn x -> ... end).()` |
| `PreferImplicitTry` | Use implicit try in function bodies |
| `StringSigils` | Use `~s` when string has 3+ escaped quotes |
| `OnePipePerLine` | Each `\|>` on its own line |
| `OneArityFunctionInPipe` | `\|> String.downcase()` not `\|> String.downcase` |
| `WithSingleClause` | Single-clause `with` + `else` → use `case` instead |
| `ImplTrue` | `@impl MyBehaviour` over `@impl true` for clarity |

#### Consistency (Team Choice — Pick One)

| Check | Options |
|-------|---------|
| `MultiAliasImportRequireUse` | `alias Mod.{A, B}` vs separate `alias` per module |
| `ParameterPatternMatching` | `pattern = param` vs `param = pattern` |
| `UnusedVariableNames` | `_user` (meaningful) vs `_` (anonymous) |

### Readable Code Patterns

#### Pipeline Readability

```elixir
# GOOD: Multi-step transformation — pipes read left-to-right, top-to-bottom
order
|> calculate_subtotal()
|> apply_discount(coupon)
|> add_tax(state)
|> round_to_cents()

# BAD: Single step — just call the function directly
name |> String.upcase()

# GOOD: Direct call for single transformation
String.upcase(name)

# BAD: Piping to anonymous function
data |> (fn x -> x * 2 end).()

# GOOD: Use then/1 or named function
data |> then(&(&1 * 2))
data |> double()
```

#### Guard Clause Patterns

Use guards for type validation at function boundaries — they're more readable than
`if`/`cond` inside the function body and the type system can reason about them:

```elixir
# GOOD: Guards declare constraints visibly
def chunk_every(enum, count, step)
    when is_integer(count) and count > 0
    when is_integer(step) and step > 0 do
  # ...
end

# GOOD: Type-specialized clauses with guards (fast path first)
def process(items) when is_list(items) do
  Enum.map(items, &transform/1)    # List-optimized path
end

def process(items) do
  items |> Enum.to_list() |> process()   # General enumerable fallback
end

# BAD: Type check buried in function body
def process(items) do
  if is_list(items) do
    Enum.map(items, &transform/1)
  else
    items |> Enum.to_list() |> process()
  end
end
```

#### Pattern Match Ordering

Always order clauses from most specific to most general:

```elixir
# GOOD: Specific → general
def handle_response({:ok, %{status: 200, body: body}}), do: {:ok, decode(body)}
def handle_response({:ok, %{status: 404}}), do: {:error, :not_found}
def handle_response({:ok, %{status: status}}), do: {:error, {:http_error, status}}
def handle_response({:error, reason}), do: {:error, reason}

# GOOD: Pin operator signals "must match exactly"
case acc do
  [^current | _] -> acc       # Duplicate — skip
  _ -> [current | acc]         # New value — prepend
end

# GOOD: Success path first, then errors
case Repo.insert(changeset) do
  {:ok, record} -> {:ok, record}
  {:error, changeset} -> {:error, format_errors(changeset)}
end
```

#### with-Chain Formatting

```elixir
# GOOD: Each clause on its own line, aligned
with {:ok, user} <- Accounts.get_user(id),
     {:ok, token} <- Tokens.generate(user),
     :ok <- Mailer.send_reset(user, token) do
  {:ok, :email_sent}
else
  {:error, :not_found} -> {:error, :user_not_found}
  {:error, :rate_limited} -> {:error, :try_later}
  {:error, reason} -> {:error, reason}
end

# BAD: with for single clause — use case instead
with {:ok, user} <- Accounts.get_user(id) do
  {:ok, user.name}
else
  {:error, _} -> {:error, :not_found}
end

# GOOD: case for single pattern
case Accounts.get_user(id) do
  {:ok, user} -> {:ok, user.name}
  {:error, _} -> {:error, :not_found}
end
```

#### Multi-Line Function Calls

```elixir
# GOOD: One argument per line when many parameters
defp process_param(
       key,
       params,
       types,
       data,
       defaults,
       {changes, errors, valid?}
     ) do
  # ...
end

# GOOD: Keyword list on separate lines when long
changeset
|> validate_required([:name, :email])
|> validate_format(:email, ~r/@/,
  message: "must contain @"
)
|> validate_length(:name,
  min: 2,
  max: 100,
  message: "must be between 2 and 100 characters"
)
```

#### Import/Alias/Require Organization

```elixir
defmodule MyApp.Catalog do
  @moduledoc "Product catalog management."

  # 1. use — changes module behavior
  use Ecto.Schema

  # 2. import — brings functions into scope
  import Ecto.Changeset
  import Ecto.Query, warn: false

  # 3. alias — creates shortcuts (alphabetized)
  alias MyApp.Accounts.User
  alias MyApp.Catalog.{Category, Product}
  alias MyApp.Repo

  # 4. require — compile-time macros
  require Logger

  # ... module attributes, types, functions
end
```

#### Variable Naming for Readability

```elixir
# GOOD: Descriptive names reveal intent
def transfer(from_account, to_account, amount) do
  # ...
end

# BAD: Single letters obscure meaning (except in short lambdas/comprehensions)
def transfer(a, b, c) do
  # ...
end

# GOOD: Single letters OK in lambdas and comprehensions
Enum.map(users, &(&1.name))
for u <- users, u.active?, do: u.email

# GOOD: Underscore prefix for unused but documented
def handle_info(_message, state), do: {:noreply, state}

# GOOD: Meaningful underscore names when debugging might need them
def handle_call(_request, _from, state), do: {:reply, :ok, state}
```

#### Private Function Naming

Follow the stdlib convention for private helper naming:

```elixir
# Optimized variant for specific type
defp process_list(items) when is_list(items), do: ...
defp process_map(items) when is_map(items), do: ...

# Recursive helper for public function
defp do_transform([], acc), do: Enum.reverse(acc)
defp do_transform([h | t], acc), do: do_transform(t, [convert(h) | acc])

# Reducer functions
defp reduce_totals(item, acc), do: ...
```

#### Readable Conditionals

```elixir
# GOOD: Pattern match in function head — clearest
def status(%Order{paid: true}), do: :paid
def status(%Order{shipped: true}), do: :shipped
def status(%Order{}), do: :pending

# GOOD: case for multiple patterns on same value
case order.status do
  :pending -> process_payment(order)
  :paid -> ship_order(order)
  :shipped -> track_delivery(order)
end

# GOOD: cond for unrelated boolean conditions
cond do
  order.total > 100 -> apply_bulk_discount(order)
  order.coupon != nil -> apply_coupon(order)
  true -> order
end

# BAD: Nested if/else — hard to read
if order.paid do
  if order.shipped do
    :delivered
  else
    :processing
  end
else
  :pending
end
```

#### Ecto.Multi for Readable Transactions

```elixir
# GOOD: Named steps read as a story
def transfer_funds(from, to, amount) do
  Ecto.Multi.new()
  |> Ecto.Multi.update(:debit, debit_changeset(from, amount))
  |> Ecto.Multi.update(:credit, credit_changeset(to, amount))
  |> Ecto.Multi.insert(:log, transfer_log(from, to, amount))
  |> Repo.transaction()
  |> case do
    {:ok, %{debit: debit, credit: credit, log: log}} ->
      {:ok, log}
    {:error, :debit, changeset, _changes} ->
      {:error, :insufficient_funds}
    {:error, failed_step, changeset, _changes} ->
      {:error, {failed_step, changeset}}
  end
end
```

#### Separation of Pure Logic from Side Effects

```elixir
# GOOD: Pure calculation function — easy to test, easy to read
defmodule MyApp.Pricing do
  def calculate_total(items, tax_rate) do
    subtotal = Enum.reduce(items, Decimal.new(0), &Decimal.add(&2, &1.price))
    tax = Decimal.mult(subtotal, tax_rate)
    Decimal.add(subtotal, tax)
  end
end

# Side effects isolated in context
defmodule MyApp.Orders do
  def create_order(items, params) do
    total = Pricing.calculate_total(items, tax_rate())
    %Order{}
    |> Order.changeset(Map.put(params, :total, total))
    |> Repo.insert()
  end
end
```

### Code Style BAD/GOOD

**Fighting the formatter with escape hatches:**
```elixir
# BAD: Manual alignment that formatter will destroy
result = some_function(arg1,
                       arg2,
                       arg3)

# GOOD: Let formatter decide — it uses 2-space nesting
result =
  some_function(
    arg1,
    arg2,
    arg3
  )
```

**Missing import_deps:**
```elixir
# BAD: Duplicating library DSL config in your .formatter.exs
locals_without_parens: [
  get: 3, post: 3, put: 3, patch: 3,  # Phoenix
  field: 2, belongs_to: 3,             # Ecto
  plug: 1, plug: 2                     # Plug
]

# GOOD: Let libraries provide their own config
import_deps: [:phoenix, :ecto, :ecto_sql, :plug]
```

**Pipe into block construct:**
```elixir
# BAD: Piping into case/if/with is hard to follow
data
|> transform()
|> case do
  {:ok, result} -> result
  {:error, _} -> nil
end

# GOOD: Assign, then match
result = data |> transform()
case result do
  {:ok, value} -> value
  {:error, _} -> nil
end
```

**Inconsistent do: keyword vs do/end block:**
```elixir
# BAD: Long expression crammed into keyword syntax
if user.admin?, do: render(conn, :admin_dashboard, users: list_users(), stats: get_stats()), else: redirect(conn, to: ~p"/")

# GOOD: Use do/end when expressions are complex
if user.admin? do
  render(conn, :admin_dashboard, users: list_users(), stats: get_stats())
else
  redirect(conn, to: ~p"/")
end

# GOOD: Keyword syntax for short, simple expressions
if user.admin?, do: :admin, else: :user
```

**Deeply nested data access:**
```elixir
# BAD: Chain of Map.get for nested access
city = Map.get(Map.get(Map.get(user, :address, %{}), :location, %{}), :city)

# GOOD: get_in with Access path
city = get_in(user, [:address, :location, :city])

# GOOD: Pattern match when structure is known
%{address: %{location: %{city: city}}} = user
```

**Magic numbers and unnamed constants:**
```elixir
# BAD: What does 86400 mean?
Process.send_after(self(), :cleanup, 86400 * 1000)

# GOOD: Module attribute gives it a name
@cleanup_interval :timer.hours(24)
Process.send_after(self(), :cleanup, @cleanup_interval)
```

