# LuaEx 5.5.0

LuaEx is an extended Lua fork based on Lua 5.5.0. It keeps Lua's runtime model and standard library, while adding a small set of syntax features aimed at cleaner data-oriented and functional-style code.

LuaEx source files use the `.luex` extension and are executed with the `luex` command:

```sh
luex file.luex
```

The project preserves the original Lua copyright and license. LuaEx modifications are copyright (C) 2026 enso-x.

## Build and Test

```sh
make linux       # or make macosx / make mingw
make test
make install     # installs lua, luex, and luac; may require a writable INSTALL_TOP
```

For a local installation, use `make install INSTALL_TOP="$HOME/.local"`.
The MinGW build produces `lua.exe`, `luex.exe`, and `luac.exe`.
`make test` runs the regression suite in `tests/run.lua` and the feature examples
in `test.luex`, and exits unsuccessfully if a check fails.

## Goals

- Stay close to Lua semantics.
- Add syntax that compiles naturally to ordinary Lua behavior.
- Keep `false` and `nil` semantics explicit.
- Make nested data access, pipelines, and callbacks less noisy.
- Keep regular `.lua` code separate from extended `.luex` code.

## Added Syntax

### Compound Assignment

LuaEx supports `+=`, `-=`, `*=`, and `/=` for assignable variables and fields.

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

LuaEx adds expression lambdas with `=>`.

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

## File Extension

LuaEx code should be stored in `.luex` files.

The extension is intentional:

- `.lua` remains regular Lua.
- `.luex` enables extended LuaEx syntax.
- Tooling can distinguish Lua and LuaEx files without guessing.

File loading selects the dialect from the filename, independently of whether
the executable is named `lua` or `luex`. Only `.luex` files enable extensions;
ordinary `.lua` files reject them.

Command-line snippets (`-e`), stdin, the REPL, and anonymous `load` strings
support LuaEx syntax. Embedders can select the dialect with the chunk name:

```lua
local extended = assert(load("return (x => x * 2)(21)", "@example.luex"))
local ordinary = assert(load("return 21 * 2", "@example.lua"))
```

A non-file chunk name ending in `.lua` also requests ordinary Lua syntax.

## Tooling

The LuaEx ecosystem currently includes:

- `luaex`: the language/interpreter repository.
- `tree-sitter-luaex`: a Tree-sitter grammar for LuaEx syntax.
- `luex-language-server`: LuexLS, a LuaLS fork with LuaEx parser support.
- `luaex-zed`: a Zed extension that wires LuaEx highlighting and LuexLS into Zed.

## Compatibility Notes

LuaEx is source-compatible with ordinary Lua code where extended syntax is not used. The runtime remains Lua-like; the added features are syntactic conveniences layered on top of Lua's model.

Current limitations:

- LuaEx is a language fork, not a Lua standard feature.
- Tooling support is provided by LuaEx-specific packages.
- Some semantic analysis for new operators is still intentionally conservative in LuexLS.
- Companion grammars and editor integrations must support the new `(...) =>`
  spelling; their repositories are separate from this interpreter change.

## License

LuaEx is based on Lua 5.5.0 and follows the Lua license. The original Lua copyright is preserved. LuaEx modifications are copyright (C) 2026 enso-x.
