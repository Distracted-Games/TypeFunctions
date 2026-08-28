# TypeFunctions

A collection of Luau type functions for working with dictionary and enum-like types in the modern Roblox Luau Type Solver.

`TypeFunctions` provides utilities for deriving string-literal unions from dictionaries, constructing strongly typed enum dictionaries from those unions, and adding correctly typed indexers to existing dictionary types.

The library exists primarily to address a change in the Luau Type Solver: iterating over a dictionary with `for` requires the table type to have an indexer. Previously, developers could work around this by adding a generic `[string]: any` indexer to their type, but that sacrifices type safety and weakens the type information provided by the dictionary (not to mention it ruined autocomplete).

With `TypeFunctions.AddKeyIndexer`, the indexer is derived automatically from the dictionary's existing properties, preserving the actual key and value types.

## Features

* Derive an enum type from a dictionary with `EnumOf<T>`
* Construct a strongly typed enum dictionary with `FromEnum<T>`
* Add a type-safe indexer to an existing dictionary with `AddKeyIndexer<T>`
* Avoid unnecessary `[string]: any` indexers
* Designed for `--!strict` Luau code and the modern Roblox Luau Type Solver

## Installation

Download the latest release from the [latest release page](https://github.com/Distracted-Games/TypeFunctions/releases/latest).

Copy `TypeFunctions.luau` into your project and require it wherever you need the type functions.

```lua
local TypeFunctions = require(path.to.TypeFunctions)
```

> **Note:** Type functions are a Luau type-system feature and must be used in a context where the Luau Type Solver can evaluate them. Also, if you are not familiar with Luau type functions, I recommend giving a quick read about them [here](https://luau.org/types/type-functions/). The main thing to note is that you do not call these functions with parentheses `()` but with angle brackets `<>` instead.

## Usage

### `EnumOf<T>`

`EnumOf<T>` produces a string-literal union containing the keys of a dictionary type.

Given:

```lua
local Currency = {
	Coins = "Coins",
	Gems = "Gems",
}
```

You can derive the corresponding enum type directly from the dictionary:

```lua
type CurrencyEnum = TypeFunctions.EnumOf<typeof(Currency)>
```

`CurrencyEnum` resolves to:

```lua
"Coins" | "Gems"
```

This is useful for maintaining a single source of truth: adding a new entry to the dictionary automatically adds the corresponding key to the derived type.

---

### `FromEnum<T>`

If you prefer to write out your custom Enum type first and then ensure the Enum table adheres to that type, `FromEnum<T>` creates a table type from a string-literal union.

```lua
type CurrencyEnum =
	"Coins"
	| "Gems"

local Currency: TypeFunctions.FromEnum<CurrencyEnum> = {
	Coins = "Coins",
	Gems = "Gems",
}
```

The resulting type contains properties for each member of the union and an indexer matching the enum:

```text
"Coins" -> "Coins"
"Gems"  -> "Gems"
```

This makes it useful for creating enum-like tables where both explicitly named properties and indexed access are type-safe.

For example:

```lua
local Currency: TypeFunctions.FromEnum<CurrencyEnum> = {
	Coins = "Coins",
	Gems = "Gems",
}

local currency: CurrencyEnum = "Coins"

print(Currency[currency])
```

`FromEnum<T>` expects `T` to be either a string literal or a union of string literals.

---

### `AddKeyIndexer<T>`

`AddKeyIndexer<T>` adds an automatically derived indexer to an existing dictionary type.

Consider a normal dictionary:

```lua
local Currency = {
	Coins = 100,
	Gems = 50,
}
```

Its inferred type contains the individual properties:

```text
Coins: number
Gems: number
```

However, iterating over the dictionary can produce a Type Solver error because the table does not have an indexer.

A common workaround is to manually add something like:

```lua
type Currency = {
	Coins: number,
	Gems: number,

	[string]: any,
}
```

or:

```lua
type Currency: { [string]: number } = {
	Coins: number,
	Gems: number,
}
```

This satisfies the Type Solver, but it also throws away useful type information, including the explicit key Enum (`"Coins" | "Gems"`).

Instead, use `AddKeyIndexer`:

```lua
for currency, amount in Currency :: TypeFunctions.AddKeyIndexer<typeof(Currency)> do
	print(`{currency} = {amount}`)
end
```

The type function derives the indexer from the dictionary's existing properties.

Conceptually, the resulting type is equivalent to:

```lua
{
	Coins: number,
	Gems: number,

	["Coins" | "Gems"]: number,
}
```

The key and value types are inferred from the original dictionary, so no `any` is necessary.

### Example with different value types

The indexer is derived from both the keys and values present in the original type:

```lua
local Values = {
	Coins = 100,
	Gems = 50,
	Keys = 10,
}

type ValuesWithIndexer = TypeFunctions.AddKeyIndexer<typeof(Values)>

for key, value in Values :: ValuesWithIndexer do
	print(key, value)
end
```

The resulting indexer effectively represents:

```lua
["Coins" | "Gems" | "Keys"]: number
```

This preserves the information already known by the Type Solver rather than replacing it with a broad `string`/`any` indexer.

## Complete Example

The three functions can also be used together to build a strongly typed enum system.

```lua
local TypeFunctions = require(path.to.TypeFunctions)

local Currency = {
	Coins = "Coins",
	Gems = "Gems",
}

type CurrencyEnum = TypeFunctions.EnumOf<typeof(Currency)>

local CurrencyLookup: TypeFunctions.FromEnum<CurrencyEnum> = {
	Coins = "Coins",
	Gems = "Gems",
}

for currency, value in CurrencyLookup :: TypeFunctions.AddKeyIndexer<typeof(CurrencyLookup)> do
	print(`{currency} = {value}`)
end
```

Granted, creating an Enum type from the dictionary, then the dictionary type from the Enum like that is overkill but is shown for demonstration only.

## Why not `[string]: any`?

The Type Solver's indexer requirement is useful because it prevents arbitrary indexing of tables whose types do not support it. However, when iterating over a statically known dictionary, you already have enough information to construct a precise indexer.

For example:

```lua
local Items = {
	Sword = 10,
	Shield = 20,
	Potion = 5,
}
```

Adding:

```lua
[string]: any
```

technically makes the table iterable, but communicates that **any string can be used as a key and the result can be anything**.

`AddKeyIndexer` instead derives:

```text
"Sword" | "Shield" | "Potion"
```

for the keys and:

```text
number
```

for the values.

This keeps the Type Solver's type information intact while satisfying its indexer requirements.

Now, if you were to try:

```lua
local item = "Staff"
local amount = Items[item] -- Type Solver throws an error here
```

Similarly, autocomplete remains intact, so `Items.` will reveal all strictly typed keys in the dictionary.

## API

### `EnumOf<T>`

```lua
TypeFunctions.EnumOf<T>
```

Returns a union containing the keys of a table type.

**Requirements:**

* `T` must be a table type (e.g., `typeof(table)`).

---

### `FromEnum<T>`

```lua
TypeFunctions.FromEnum<T>
```

Creates a table type whose properties and indexer correspond to the members of a string-literal union.

**Requirements:**

* `T` must be a string literal or a union of string literals.

---

### `AddKeyIndexer<T>`

```lua
TypeFunctions.AddKeyIndexer<T>
```

Creates a new table type containing the original properties of `T` plus an indexer whose key and value types are derived from those properties.

**Requirements:**

* `T` must be a table type (e.g., `typeof(table)`).

## License

See the repository's `LICENSE` file for licensing information.
