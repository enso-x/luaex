# LuaEx 5.5.0

LuaEx is an extended Lua fork based on Lua 5.5.0. It keeps Lua's runtime model and standard library, while adding a small set of syntax features aimed at cleaner data-oriented and functional-style code.

LuaEx source files use the `.luex` extension and are executed with the `luex` command:

```sh
luex file.luex
```

The project preserves the original Lua copyright and license. LuaEx modifications are copyright (C) 2026 enso-x.

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

### Optional Chaining

`?.` safely accesses nested fields when an intermediate value may be `nil`.

```lua
local name = user?.profile?.name
```

Optional calls are supported too:

```lua
local displayName = user?.profile?.getName?.()
```

If any optional step receives `nil`, the whole chain returns `nil` instead of throwing an indexing error.

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

### Lambda Expressions

LuaEx adds expression lambdas with `=>`.

Single-argument shorthand:

```lua
local double = x => x * 2
```

Multiple parameters:

```lua
local sum = fn(a, b) => a + b
```

Block body:

```lua
local f = fn(x) => do
    local y = x * 2
    return y + 1
end
```

Expression lambdas implicitly return their body expression. Block lambdas use normal Lua statements and explicit `return`.

## File Extension

LuaEx code should be stored in `.luex` files.

The extension is intentional:

- `.lua` remains regular Lua.
- `.luex` enables extended LuaEx syntax.
- Tooling can distinguish Lua and LuaEx files without guessing.

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

## License

LuaEx is based on Lua 5.5.0 and follows the Lua license. The original Lua copyright is preserved. LuaEx modifications are copyright (C) 2026 enso-x.
