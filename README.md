# ExLua 5.5.0

ExLua is an extended Lua fork based on Lua 5.5.0. It keeps Lua's runtime model and standard library, while adding a small set of syntax features aimed at cleaner data-oriented and functional-style code.

ExLua source files use `.exlua` (recommended) or `.exl` and are executed with
the `exlua` command. Legacy `.luex` files and the `luex` command remain supported:

```sh
exlua file.exlua
```

The project preserves the original Lua copyright and license. ExLua modifications are copyright (C) 2026 enso-x.

## Build and Test

```sh
make linux       # or make macosx / make mingw
make test
make install     # installs exlua, lua, luex, and luac
```

For a local installation, use `make install INSTALL_TOP="$HOME/.local"`.
The MinGW build produces `exlua.exe`, `lua.exe`, `luex.exe`, and `luac.exe`.
`make test` runs the regression suite in `tests/run.lua` and the feature examples
in `test.exlua`, and exits unsuccessfully if a check fails.

## Goals

- Stay close to Lua semantics.
- Add syntax that compiles naturally to ordinary Lua behavior.
- Keep `false` and `nil` semantics explicit.
- Make nested data access, pipelines, and callbacks less noisy.
- Keep regular `.lua` code separate from extended `.exlua` code.

## Added Syntax

### Compound Assignment

ExLua supports `+=`, `-=`, `*=`, and `/=` for assignable variables and fields.

```lua
x += 1
damage *= multiplier
player.hp -= damage
scale /= 2
```

These forms behave like:

```lua
x = x + 1
damage = damage * multiplier
player.hp = player.hp - damage
scale = scale / 2
```

The left-hand side is evaluated as an assignment target, not as a textual macro.

### Nil Coalescing

`??` returns the right-hand value only when the left-hand value is `nil`.

```lua
local name = user.name ?? "Unknown"
```

Unlike Lua conditionals, `??` does not treat `false` as missing:

```lua
local enabled = false ?? true -- false
local title = nil ?? "Untitled" -- "Untitled"
```

The right-hand expression is evaluated only when needed. The operator returns
one value and does not assign to the left-hand variable.

### Optional Chaining

`?.` safely accesses nested fields when an intermediate value may be `nil`.

```lua
local name = user?.profile?.name
```

Optional calls are supported too:

```lua
local displayName = user?.profile?.getName?.()
```

Each `?.` step returns `nil` when its receiver is `nil`. Use `?.` at each
potentially missing step; a subsequent ordinary `.` or call still follows
ordinary Lua rules. Optional calls skip their arguments when the function is
`nil`, return one value, and can be used as statements:

```lua
onUpdate?.(deltaTime)
```

Only `nil` is treated as missing; indexing `false` still raises an error.

### Pipeline Operator

`|>` passes the value on the left into the call on the right.

```lua
5 |> print
5 |> print()
```

For calls with arguments, the piped value becomes the first argument:

```lua
value |> f(a, b)
```

This is equivalent to:

```lua
f(value, a, b)
```

Method pipelines are supported with `|> :method(...)`:

```lua
object |> :normalize()
```

This is equivalent to:

```lua
object:normalize()
```

A pipeline can also be used as a statement. If the result is not assigned, it is still evaluated for its effects.

```lua
amount + bonus |> print;
-5 |> math.abs |> print;
"hello" |> string.upper |> print;
```

The left-hand value is evaluated once, before the callee and its arguments.
Ordinary pipelines pass and return one value; the final explicit argument can
expand multiple results as in a Lua call. Method pipelines retain normal Lua
method-call return behavior. `|>` and `??` have the same precedence as `or` and
associate to the left; parentheses make mixed expressions explicit.

As in Lua, a newline does not terminate an expression. Use a semicolon between
statements when the next line could continue the previous expression, for
example when it begins with `-` or `(`.

### Lambda Expressions

ExLua adds expression lambdas with `=>`.

Single-argument shorthand:

```lua
local double = x => x * 2
```

Multiple parameters:

```lua
local sum = (a, b) => a + b
```

Zero parameters and a parenthesized single parameter:

```lua
local now = () => os.time()
local double = (x) => x * 2
```

Block body:

```lua
local f = (x) => do
    local y = x * 2
    return y + 1
end
```

Expression lambdas implicitly return one value from their body expression.
Block lambdas use normal Lua statements and explicit `return`, including
multiple return values. Both forms use normal Lua closures and lexical scope;
`self` remains an explicit parameter.

Lua varargs are supported:

```lua
local first = (...) => (...)
local collect = (head, ...) => {head, ...}
local pair = (a, b) => do return a, b end
```

The old `fn(a, b) => ...` spelling has been replaced by `(a, b) => ...`.
`fn` is an ordinary identifier and can be called or passed as a callback.

### Named Arguments and Default Parameters

Ordinary functions and arrow functions accept arguments by parameter name,
using Python-style `name = value` syntax. Named arguments can appear in any
order, after any positional arguments:

```lua
local function rectangle(width, height = 10, filled = true)
    return width * height, filled
end

rectangle(height = 20, width = 5)  -- 100, true
rectangle(5, filled = false)      -- 50, false

local scale = (value, factor = 2) => value * factor
scale(factor = 3, value = 10)     -- 30
10 |> scale(factor = 3)          -- 30

local describe = (name = "world") => do
    return "Hello, " .. name
end
describe()                       -- "Hello, world"
```

Names are matched against the function actually called, including functions
stored in variables, tables, or returned by other functions. Methods support
named arguments too; `object:method(...)` supplies `self` positionally.
Optional calls skip all argument expressions when the function is `nil`.
Parameter names are retained even in stripped ExLua bytecode.

Argument expressions run once, in source order. Each named value supplies one
value. A final positional call or `...` expands multiple values using ordinary
Lua rules; an expression before a named argument supplies only one value.

Defaults must follow parameters without defaults. They are evaluated once in
the enclosing scope when the function is created, as in Python. They do not
refer to the new function's parameters. A mutable default is shared by calls
to that closure; a new closure gets its own defaults:

```lua
local add = (value, items = {}) => do
    items[#items + 1] = value
    return items
end
```

An explicitly supplied `nil` or `false` is a value and does not select a
default. Omitted parameters without defaults still receive `nil`, as in Lua.
Extra positional arguments keep Lua's usual behavior, including `...`.

The following are errors:

- An unknown name, a repeated named argument, or a parameter supplied both
  positionally and by name.
- A positional argument after a named argument.
- Naming a parameter whose name is duplicated in the declaration.
- Passing named arguments to a native C function without an ExLua signature,
  or to a callable table/userdata. `Class.new` and adapters returned by `wraps`
  carry signatures; other native calls remain positional.

`...` and its optional table name are not keyword parameters. Python's
`*args`/`**kwargs` unpacking, keyword-only parameters, and positional-only
parameter declarations are not part of this syntax.

### Classes

Classes are tables; instances are tables whose metatable is their class.
Methods are shared and receive an implicit first parameter, `self`:

```lua
local class Player
    constructor(name, hp = 100)
        self.name = name
        self.hp = hp
        self.inventory = {}  -- a fresh table for each instance
    end

    function damage(amount = 1)
        self.hp -= amount
        return self.hp
    end

    static kind = "player"
    static function describe(name)
        return f"Player {name}"
    end
end

local player = Player.new(name = "Alex")
player:damage(amount = 15)
print(Player.describe(name = player.name))
```

- `local class Name ... end` declares a local binding. `class Name ... end`
  assigns a name using the same scope rules as `function Name(...) ... end`.
- `constructor(...) ... end` defines `Class.__init(self, ...)`. `Class.new(...)`
  creates the instance, calls its initializer and returns the instance;
  initializer return values are ignored. Constructor errors and yields propagate.
- `function method(...) ... end` adds an instance method; call it with `:`.
  `static function` has no implicit `self`; call it with `.`.
- `static name = expression` evaluates once when the class is declared.
  These fields live on the class and are shared, including mutable tables.
  Initialize per-instance state in the constructor. Static fields are also
  visible through an instance's ordinary `__index` lookup.
- Defaults follow the same definition-time rules as other functions.
  Named arguments work on constructors, instance methods and static functions.
- A class without an initializer creates an empty instance. Call classes with
  `Class.new(...)`; the class table itself is not callable.

Single inheritance uses `extends` (an optional `do` can precede the body):

```lua
local class Admin extends Player
    constructor(name, level = 1)
        Admin.super.__init(self, name = name)
        self.level = level
    end

    function damage(amount = 1)
        return Admin.super.damage(self, amount = amount / 2)
    end
end
```

The parent expression runs once. Inherited methods and static fields are found
through the class table's metatable. `Class.super` refers to the parent.
A subclass without a constructor uses the parent's initializer but still creates
an instance of the subclass. A declared constructor calls the parent explicitly
when needed. `Class.new` captures the selected initializer at declaration time;
subsequently assigning `Class.__init` does not replace that factory's initializer.
Lua's raw metamethod lookup still applies: special instance metamethods such as
`__tostring` must be declared on the concrete class to take effect.

The class body accepts constructors, methods, static functions and static fields.
Duplicate members and the reserved names `new`, `super`, `__index`, `__init`
are rejected (`constructor` is the spelling for declaring `__init`). There are
no visibility modifiers, automatic properties or multiple inheritance.
`class`, `extends`, `static` and `constructor` remain contextual identifiers.

### Decorators

Place `@expression` before a function or class declaration, or before a class
method, static function or constructor:

```lua
local function logged(fn)
    return wraps(fn, (...) => do
        print("calling function")
        return fn(...)
    end)
end

@logged
local function add(left, right = 1)
    return left + right
end

add(right = 3, left = 2)  -- 5
```

Decorator factories can accept named arguments, for example `@cached(ttl = 10)`.
The expressions are evaluated once, top to bottom, before creating the function
or class. Their results are applied bottom to top, each with the previous result
as its single positional argument: `@a @b` produces `a(b(value))`. Only one return
value from each decorator is used. The final result becomes the declared binding.
A decorator can return its input unchanged, for example to register an event
handler. Plain function assignments such as `@logged function object:method(...)`
are supported too. Decorators do not apply to arbitrary variable declarations.

`wraps(original, wrapper)` creates a forwarding adapter with the original
function's parameter names and defaults. It does not mutate either argument.
Use it when a decorator's wrapper forwards positional arguments to the original;
a bare `function(...)` has no named parameters to match. Both inputs must be
functions, and the original must have a Lua/ExLua signature. Multiple layers of
`wraps` retain the effective signature. Lambdas and `Class.new` are supported.
The adapter preserves multiple results, errors and coroutine yields. It retains
the original closure so its signature and defaults remain alive.

`wraps` and `Class.new` return C adapters. Like other C functions, these adapters
cannot themselves be passed to `string.dump`; dump/load the enclosing ExLua chunk
to recreate them. Lua methods and plain decorated Lua closures can still be dumped
under the normal upvalue/default limitations described below.

### Template Strings

Use `f"...{expression}..."` or `f'...{expression}...'`:

```lua
local message = f"Player {player.name}, HP: {player.hp}"
local status = f"Enabled: {false}, missing: {nil}"
local braces = f"{{literal braces}}, value: {1 + 2}"
```

Each hole is a full ExLua expression in the surrounding lexical scope. It is
evaluated once and immediately converted with `tostring`, in left-to-right order,
including `__tostring` metamethods. Only the first result of a hole is used.
`tostring` follows normal lexical lookup, so a local replacement is respected.
Templates always yield one string value.

Double braces `{{` and `}}` produce literal braces. Normal Lua short-string
escapes work, including Unicode, escaped newlines and embedded zero bytes.
Expressions may span lines, contain comments or table constructors, and contain
nested templates (up to 32 levels). Literal newlines in the text part require
escaping as in Lua short strings. Empty holes, unmatched braces and Python-style
format specifiers such as `{value:.2f}` are not supported; use
`f"{string.format('%.2f', value)}"` for formatting.

The `f` must touch the quote. In ordinary `.lua` sources, `f"text"` keeps its
existing meaning: calling the function `f` with a string argument.

### Destructuring

Local declarations can extract named fields with `{...}` and array positions
with `[...]`, starting at index 1. `field: binding` renames a field, and either
kind of pattern can be nested:

```lua
local {x, y: height} = position
local [first, second] = items
local {profile: {name}, coordinates: [px, py]} = user
```

The source expression runs once and supplies one value. Fields are read in
source order using ordinary indexing, including `__index`. A nested field is
read once before its children. Missing leaves become `nil`; destructuring a
missing nested object raises the usual indexing error. `false` is preserved.
Bindings become visible after the whole declaration, so `local {x} = x` reads
the outer `x`. Captures, block scope and later assignment work like regular locals.
Trailing commas are accepted; duplicate binding names are rejected.

This version supports local binding patterns. Reassignment patterns, parameter
patterns, rest/spread and defaults inside a pattern are not implemented. Use
`??` on the source or on an extracted value when a fallback is needed.

## File Extension

Use `.exlua` for new code, or `.exl` for a shorter filename.

The extension is intentional:

- `.lua` remains regular Lua.
- `.exlua` and `.exl` enable extended ExLua syntax.
- `.luex` is accepted for existing projects.
- Tooling can distinguish Lua and ExLua files without guessing.

File loading selects the dialect from the filename, independently of whether
the executable is named `exlua`, `lua`, or `luex`. Ordinary `.lua` files reject
extensions. The default `package.path` searches `.exlua` and `.exl` modules
(including `init` files) as well as legacy `.luex` and ordinary `.lua` modules.

Command-line snippets (`-e`), stdin, the REPL, and anonymous `load` strings
support ExLua syntax. Embedders can select the dialect with the chunk name:

```lua
local extended = assert(load("return (x => x * 2)(21)", "@example.exlua"))
local ordinary = assert(load("return 21 * 2", "@example.lua"))
```

A non-file chunk name ending in `.lua` also requests ordinary Lua syntax.

## Tooling

Companion repositories retain their existing names:

- `luaex`: the language/interpreter repository.
- `tree-sitter-luaex`: a Tree-sitter grammar for ExLua syntax.
- `luex-language-server`: LuexLS, a LuaLS fork with ExLua parser support.
- `luaex-zed`: a Zed extension that wires ExLua highlighting and LuexLS into Zed.

## Compatibility Notes

ExLua retains ordinary Lua source behavior when extended syntax is not used.
The C API names, library names, and Lua 5.5 module directories are preserved.
`_VERSION` and the interpreter banner identify the language as ExLua.

Named arguments and default values add runtime metadata and bytecode
instructions. **Recompile existing bytecode:** this version uses ExLua binary
format 1 and rejects stock Lua / earlier format 0 chunks. Stock Lua cannot
load the new chunks. This does not affect loading ordinary `.lua` source.

`string.dump` preserves parameter names, but does not serialize captured
default values (which may be arbitrary objects), just as it does not preserve
captured upvalue values. Dump/load the enclosing chunk to recreate functions
with their defaults. A separately dumped function can still be called with
all defaulted parameters supplied explicitly; attempting to use a lost default
raises an explanatory error.

Classes, decorators, template strings and destructuring introduce no new VM
instructions or binary format changes beyond format 1. Classes lower to member
tables plus `exlua.class(table, parent)`; templates lower to `tostring` calls and
concatenation. `exlua.class` and `wraps` are installed by the base library
(`luaopen_base` / `luaL_openlibs`). An embedder using a custom `_ENV` must provide
`exlua` for class declarations and `tostring` for template holes. Ordinary method
calls and destructuring use the existing VM operations.

The additive C API `lua_setsignature(L, adapter_index, source_index, skip)` attaches
a Lua signature to a C closure. `skip` is a nonnegative count of leading source
parameters supplied by the adapter; it is added to an existing adapter's offset.
The destination must be a C closure with upvalues, not a light C function. It
returns 1 on success, or 0 (without changing the destination) if the source has no
signature or the offset exceeds its parameter count. This affects named argument
binding only; the C function remains responsible for forwarding positional calls.
All existing C API entry points remain available. Rebuild native modules against
the ExLua headers/library; internal closure layouts are not stock-Lua compatible.

Current limitations:

- ExLua is a language fork, not a Lua standard feature.
- Tooling support is provided by ExLua-specific packages.
- Some semantic analysis for new operators is still intentionally conservative in LuexLS.
- Companion grammars and editor integrations need updates for the new file
  extensions, named arguments, defaults, classes, decorators, template strings,
  destructuring, and `(...) =>` spelling; their
  repositories are separate from this interpreter change.

## License

ExLua is based on Lua 5.5.0 and follows the Lua license. The original Lua copyright is preserved. ExLua modifications are copyright (C) 2026 enso-x.
