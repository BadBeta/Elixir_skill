# Elixir Type System Reference (1.17-1.20)

> Supporting reference for the [Elixir skill](SKILL.md). Contains the full set-theoretic type system notation, inference, dynamic(), map key domains, typespecs, compiler configuration, roadmap, and BAD/GOOD pairs.

## Type System (Elixir 1.17-1.20)

Elixir has a **gradual set-theoretic type system** built into the compiler. It performs
automatic type inference and checking without requiring type annotations. The system
is sound, gradual (via `dynamic()`), and uses set operations (union, intersection, negation).

### Type System Evolution

| Version | Capabilities |
|---------|-------------|
| 1.17 | Type checking of patterns, guards, and basic expressions. Struct-aware typing. Initial warnings for type mismatches |
| 1.18 | Full type inference of patterns. Type checking across function clauses. Improved map and struct tracking |
| 1.19 | Expanded expression type checking. Better error messages. More stdlib functions typed |
| 1.20 | Full inference of function definitions and guards. Cross-clause type narrowing. Complete map key domain typing. Typed Map module operations. Redundant clause detection |

### Set-Theoretic Type Notation

The compiler uses set-theoretic types in warnings. This notation differs from `@spec` typespecs.

#### Basic Types

```
atom()          # all atoms
binary()        # all binaries (bitstring divisible by 8)
bitstring()     # all bitstrings (binary() is subtype)
integer()       # all integers
float()         # all floats
pid()           # all process identifiers
port()          # all ports
reference()     # all references
empty_list()    # the empty list []
function()      # all functions
map()           # all maps (same as %{...})
tuple()         # all tuples
```

#### Special Types

```
term()          # all types (top type, universal set)
none()          # empty set (no values, e.g. atom() and integer())
dynamic()       # gradual type — range of possible runtime types
not_set()       # marker: map key is absent
if_set(type)    # marker: map key is optional
```

#### Set Operations

```elixir
type1 or type2          # union: either type
type1 and type2         # intersection: both types
type1 and not type2     # difference: first but not second
not type                # negation: everything except type
```

#### Convenience Aliases

```elixir
boolean()       # = true or false
number()        # = integer() or float()
list()          # = empty_list() or non_empty_list(term())
list(a)         # = empty_list() or non_empty_list(a)
```

### Atoms in the Type System

Individual atoms are distinct types:

```elixir
:ok             # the specific atom :ok
:error          # the specific atom :error
nil             # the atom nil
true            # the atom true
false           # the atom false
SomeModule      # module atoms are also types
```

### Tuple Types

```elixir
tuple()                     # all tuples
{:ok, binary()}             # exact 2-element tuple
{:error, binary(), term()}  # exact 3-element tuple
{:ok, binary(), ...}        # at least 2 elements (open tuple)
```

### List Types

```elixir
list(integer())                          # [] or [1, 2, 3]
non_empty_list(integer())                # [1, 2, 3] (not empty)
non_empty_list(integer(), integer())     # improper: [1, 2 | 3]
```

### Map Types

Maps are the most sophisticated part of the type system:

```elixir
# Closed maps — exactly these keys, no more
%{name: binary(), age: integer()}

# Open maps — these keys required, others allowed
%{..., name: binary(), age: integer()}

# Optional keys — may or may not be present
%{name: binary(), age: if_set(integer())}

# Absent keys — key is definitely not present
%{..., age: not_set()}

# Domain keys — non-atom key types (always optional)
%{binary() => integer()}
%{atom() => binary(), integer() => integer()}

# Mixed — domain + specific atom keys (specific overrides domain)
%{atom() => binary(), root: integer()}

# Empty map
empty_map()
```

**Map operation type tracking (1.20+):**

```elixir
Map.put(map, :key, 123)        #=> %{..., key: integer()}
Map.delete(map, :key)          #=> %{..., key: not_set()}
Map.replace(map, :key, 123)    #=> %{..., key: if_set(integer())}

# Bang functions propagate key requirements across modules:
Map.fetch!(map, :name)         # caller must provide %{..., name: term()}
```

### Function Types

```elixir
function()                      # all functions
(integer() -> boolean())        # int argument, bool return
(binary(), binary() -> binary())  # two args

# Multiple clauses use intersection (NOT union):
(integer() -> integer()) and (boolean() -> boolean())
# = function that handles both integer→integer AND boolean→boolean
```

**Why intersection, not union?** A function with multiple clauses belongs to
*both* sets simultaneously — it can handle integers AND booleans. Union would
mean it handles integers OR booleans (not necessarily both).

### The `dynamic()` Type — Gradual Typing

`dynamic()` is the key innovation that lets the type system check existing untyped code.

**How it works:** `dynamic()` represents a *range* of possible runtime types. Unlike `any` in
TypeScript or similar, `dynamic()` still provides meaningful type checking — the compiler
warns only when ALL possible types in the range will fail.

```elixir
# Without dynamic — static typing:
# atom() or integer() given to Integer.to_string/1 → WARNING (atoms fail)

# With dynamic — gradual typing:
# dynamic(atom() or integer()) given to Integer.to_string/1 → no warning
# (at least integer() succeeds, so it might work at runtime)

# But dynamic(atom()) given to Integer.to_string/1 → WARNING
# (no type in the range works)
```

**Dynamic is always at the root:** Writing `{:ok, dynamic()}` gets rewritten to
`dynamic({:ok, term()})`. You cannot make only part of a data structure gradual.

**Modes of type checking:**

| Mode | When | Behavior |
|------|------|----------|
| `:static` | Function has explicit type signature | Full static checking, dynamic() args use subtyping |
| `:dynamic` | Function has no type signature | Gradual checking, wraps returns in dynamic() |
| `:infer` | During inference pass | No warnings, only learns types |

### Type Inference — How It Works

The compiler performs **module-level type inference** automatically:

1. **Pattern inference:** Patterns and guards narrow variable types
2. **Expression inference:** Operators and function calls refine types further
3. **Cross-clause inference:** Each clause's type excludes what previous clauses matched
4. **Cross-module:** Calls to other project modules are typed as `dynamic()` (stdlib and deps are fully typed)

```elixir
# 1. Guard inference
def example(x) when is_integer(x)
# x inferred as: integer()

# 2. Pattern + guard combination
def example({:ok, x} = y) when is_binary(x) or is_integer(x)
# x: binary() or integer()
# y: {:ok, binary() or integer()}

# 3. Map key inference
def example(x) when is_map_key(x, :foo)
# x: %{..., foo: dynamic()}

# 4. Negated guards
def example(x) when not is_map_key(x, :foo)
# x: %{..., foo: not_set()}  — accessing x.foo would be a type violation

# 5. Tuple size tracking
def example(x) when tuple_size(x) < 3
# x has at most 2 elements — elem(x, 3) is a type violation

# 6. Cross-clause narrowing (1.20+)
case System.get_env("VAR") do
  nil -> :not_found           # first clause handles nil
  value -> String.upcase(value) # value narrowed to binary() only
end

# 7. Function body inference (1.20+)
def sum_to_string(a, b) do
  Integer.to_string(a + b)
end
# a and b inferred as integer() (not float!) because
# Integer.to_string/1 only accepts integers
```

### Type Warnings — Reading Compiler Output

The compiler emits structured warnings with types, traces, and locations:

```text
warning: incompatible types given to User.name/1:

    User.name(%{})

given types:

    %{name: not_set()}

but expected one of:

    dynamic(%{..., name: term()})

type warning found at:
│
16 │     User.name(%{})
   │         ~
│
└─ lib/calls_user.ex:7:5: CallsUser.calls_name/0
```

**Common warning categories:**

| Warning | Meaning |
|---------|---------|
| `incompatible types given to` | Function called with wrong argument types |
| `incompatible types in pattern` | Pattern can never match the expression's type |
| `redundant clause` | Previous clauses already cover all cases — dead code |
| `mismatched comparison` | Comparing disjoint types (always true or always false) |
| `struct_comparison` | Using `<`/`>` on structs (structural, not semantic comparison) |
| `unknown_struct_field` | Accessing a field that doesn't exist on the struct |
| `badmap` | Map access on a non-map type |
| `badkey` | Key doesn't exist in the known map/struct shape |
| `badindex` | Tuple element access out of bounds |

### Traditional Typespecs (`@spec` / `@type`)

Typespecs use Erlang-based notation and are **separate from the set-theoretic type system**.
They serve as documentation, are shown in ExDoc, and are used by Dialyzer. They will eventually
be replaced by set-theoretic type signatures in a future Elixir version.

**Typespec notation uses `|` for unions, `::` for type assignment:**

```elixir
# Type definitions
@type status :: :pending | :active | :archived
@type result :: {:ok, term()} | {:error, String.t()}
@type user_id :: pos_integer()

# Struct type (convention: always define t/0)
@type t :: %__MODULE__{
  id: pos_integer(),
  email: String.t(),
  role: :admin | :member | :guest,
  inserted_at: DateTime.t()
}

# Opaque type — callers cannot inspect internals
@opaque t :: %__MODULE__{data: map()}

# Private type — only visible in this module
@typep internal_state :: {atom(), map()}

# Parameterized type
@type tree(value) :: :leaf | {:node, value, tree(value), tree(value)}
```

**Function specs:**

```elixir
@spec create(map()) :: {:ok, t()} | {:error, Ecto.Changeset.t()}
@spec get(pos_integer()) :: t() | nil
@spec list(keyword()) :: [t()]
@spec delete(t()) :: :ok

# Multiple clauses (different arities need separate specs)
@spec parse(binary()) :: {:ok, t()} | {:error, binary()}

# Callback specs with named params
@callback handle_event(event :: term(), state :: t()) ::
  {:ok, t()} | {:error, reason :: term()}

# No return (raises or loops forever)
@spec raise_error(binary()) :: no_return()
```

**Key typespec types:**

| Type | Meaning |
|------|---------|
| `term()` | Any type (same as `any()`) |
| `String.t()` | UTF-8 binary (preferred over `binary()` for text) |
| `boolean()` | `true \| false` |
| `number()` | `integer() \| float()` |
| `keyword()` | `[{atom(), term()}]` |
| `keyword(t)` | `[{atom(), t}]` |
| `charlist()` | `[char()]` |
| `iodata()` | `binary() \| iolist()` |
| `iolist()` | Nested lists of bytes/binaries |
| `pos_integer()` | Positive integers (1, 2, 3...) |
| `non_neg_integer()` | Zero and positive (0, 1, 2...) |
| `neg_integer()` | Negative integers (..., -2, -1) |
| `module()` | Atom representing a module |
| `mfa()` | `{module(), atom(), arity()}` |
| `as_boolean(t)` | Returns `t` but treated as truthy/falsy |
| `no_return()` | Function never returns |

### Typespecs vs Set-Theoretic Types — Notation Comparison

| Concept | Typespec (`@spec`) | Set-theoretic (compiler) |
|---------|-------------------|--------------------------|
| Union | `atom() \| integer()` | `atom() or integer()` |
| All terms | `any()` / `term()` | `term()` |
| Empty | `none()` | `none()` |
| Intersection | *(not supported)* | `type1 and type2` |
| Negation | *(not supported)* | `not type` |
| Difference | *(not supported)* | `type1 and not type2` |
| Gradual | *(not available)* | `dynamic()` / `dynamic(type)` |
| Optional map key | *(not expressible)* | `if_set(type)` |
| Absent map key | *(not expressible)* | `not_set()` |
| Open map | `map()` | `%{..., key: type}` |
| Function | `(a, b -> c)` | `(a, b -> c)` |
| Multi-clause fn | *(multiple @spec)* | `(a -> b) and (c -> d)` |

### Compiler Configuration

```elixir
# In mix.exs or config — control type inference scope
Code.put_compiler_option(:infer_signatures, true)    # default: true (= [:elixir])
Code.put_compiler_option(:infer_signatures, false)   # disable module-local inference
Code.put_compiler_option(:infer_signatures, [:elixir, :my_app])  # specific apps

# Type checking still runs regardless of :infer_signatures setting
# The option only controls whether function signatures are inferred
```

### Roadmap — What's Coming

The type system is under active development by the Elixir core team with CNRS:

1. **Current (1.20):** Full inference of existing codebases — no code changes needed
2. **Next milestone:** Typed structs — field types propagated through pattern matching
3. **Future:** Set-theoretic type signatures replace `@spec`/Erlang typespecs entirely

Until type signatures arrive, `@spec` remains the primary way to document function types
for humans and tools (ExDoc, Dialyzer). Write `@spec` for public APIs today — the knowledge
transfers when set-theoretic signatures become available.

### Type System BAD/GOOD

**Ignoring type warnings:**
```elixir
# BAD: Suppressing with a wrapper
def get_name(user), do: user |> Map.from_struct() |> Map.get(:name)

# GOOD: Fix the underlying type issue
@spec get_name(User.t()) :: binary()
def get_name(%User{name: name}), do: name
```

**Overly broad types in specs:**
```elixir
# BAD: map() tells nothing
@spec process(map()) :: map()

# GOOD: Specific struct or key requirements
@spec process(Order.t()) :: {:ok, Invoice.t()} | {:error, binary()}
```

**Using `any()` as escape hatch:**
```elixir
# BAD: Defeats type checking
@spec handle(any()) :: any()

# GOOD: Use term() if truly polymorphic, or specific types
@spec handle(Event.t()) :: :ok | {:error, reason :: binary()}
```

**Not leveraging bang functions for type propagation:**
```elixir
# BAD: Type system can't verify key exists
def get_name(map) do
  case Map.fetch(map, :name) do
    {:ok, name} -> name
    :error -> raise "missing name"
  end
end

# GOOD: Bang function propagates key requirement to callers
def get_name(map), do: Map.fetch!(map, :name)
# Callers must pass %{..., name: term()} — type system enforces this
```

**Wrong set operation for function types:**
```elixir
# BAD: Union means "one or the other" — wrong for multi-clause
# (integer() -> integer()) or (boolean() -> boolean())

# GOOD: Intersection means "handles both" — correct for multi-clause
# (integer() -> integer()) and (boolean() -> boolean())
```

**Redundant type guards when compiler already infers:**
```elixir
# BAD: Unnecessary guard — compiler already knows from pattern
def process(%User{} = user) when is_map(user) do
  user.name
end

# GOOD: Pattern match is sufficient — type system tracks struct shape
def process(%User{} = user) do
  user.name
end
```

