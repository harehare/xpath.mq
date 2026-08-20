<h1 align="center">xpath.mq</h1>

An abbreviated [XPath](https://www.w3.org/TR/1999/REC-xpath-19991116/) query engine implemented as an [mq](https://github.com/harehare/mq) module, for querying the value tree produced by [xml.mq](https://github.com/harehare/mq/blob/main/crates/mq-lang/modules/xml.mq)'s `xml_parse`.

## Features

- Child (`a/b`), absolute (`/a/b`), self (`.`) and parent (`..`) steps
- Descendant search at any depth (`//a`, `a//b`)
- Wildcards for elements (`*`) and attributes (`@*`)
- Attribute access (`@name`)
- `text()` and `name()` node-test functions
- Predicates: positional (`[1]`), attribute/child existence and equality,
  `and` / `or` / `not(...)`, `contains(...)`, `starts-with(...)`,
  `last()` / `position()`

## Installation

Copy `xpath.mq` to your mq module directory, or place it anywhere and reference it with `-L`.

```sh
cp xpath.mq ~/.local/mq/config/
```

## Usage

```sh
mq -L /path/to/modules -I xml \
  'import "xpath" | xpath::xpath_query(., "//user[@id=\"2\"]/name/text()")' input.xml
```

If you copied it to the mq built-in module directory:

```sh
mq -I xml 'import "xpath" | xpath::xpath_query(., "//user/@id")' input.xml
```

### HTTP Import (no local installation needed)

If `mq` was built with the `http-import` feature, you can import directly from GitHub without any local setup. This requires the `--allow-http-import` flag, which is disabled by default:

```sh
mq --allow-http-import -I xml 'import "github.com/harehare/xpath.mq" | xpath::xpath_query(., "//user/@id")' input.xml
```

Pin to a specific release with `@vX.Y.Z`:

```sh
mq --allow-http-import -I xml 'import "github.com/harehare/xpath.mq@v1.0.0" | xpath::xpath_query(., "//user/@id")' input.xml
```

## API

### `xpath_query(node, path)`

Runs an XPath-like `path` expression against `node` (a value produced by `xml.mq`'s `xml_parse`) and returns an array of matches.

| Step result | Value returned |
|---|---|
| Element step (`a`, `*`, `//a`, `.`, `..`) | The matched element dict |
| `@name` / `@*` | The attribute's string value |
| `text()` | The element's text (or `""` if it has none) |
| `name()` | The element's tag name |

Returns `[]` when nothing matches. Raises an error (via `miette`) for a malformed `path`.

### `xpath_first(node, path)`

Convenience wrapper around `xpath_query` that returns the first match, or `None` if there are no matches.

## Supported syntax

| Syntax | Meaning |
|---|---|
| `a`, `a/b` | Child element(s) named `a` (then `b`) |
| `/a`, `/a/b` | Absolute path, starting from `node` itself |
| `*` | Any child element |
| `.` | Self |
| `..` | Parent |
| `//a` | `a` at any depth below the context (also matches the context node itself when `//` starts the whole expression) |
| `a//b` | `b` at any depth below `a`'s matches |
| `@name` | Value of attribute `name` |
| `@*` | Every attribute value |
| `text()` | Text content of the context node |
| `name()` | Tag name of the context node |
| `[n]` | The `n`-th (1-based) match |
| `[@a]` | Attribute `a` exists |
| `[@a='v']`, `[a='v']` | Attribute/child text equals `v` |
| `[a]` | Child element `a` exists |
| `[expr and expr]`, `[expr or expr]`, `[not(expr)]` | Boolean combinators |
| `[contains(@a,'v')]`, `[starts-with(@a,'v')]` | Substring/prefix tests |
| `[last()]`, `[position()=n]` | Position-aware predicates |

Comparisons (`=`, `!=`) compare values as strings; `<`, `<=`, `>`, `>=` coerce both sides to numbers and evaluate to `false` (rather than erroring) when either side isn't numeric.

## Simplifications versus full XPath

These follow directly from the shape of the value tree that `xml.mq`'s `xml_parse` produces:

- There is no separate document node — the value you pass to `xpath_query` is both the context node and, for a leading `/`, the root.
- The "string value" of an element is just its own `text` field (the text immediately inside it), not the concatenation of all descendant text.
- Mixed content ordering (text interleaved with child elements) isn't recoverable from `xml_parse`'s output, so it isn't reconstructed here either.
- `position()` / `last()` for a `//` step are computed over the flattened list of matches found under each context node, not grouped per ancestor as in strict XPath semantics.
- No union operator (`|`) and no unabbreviated axes (`child::`, `parent::`, `following-sibling::`, ...).

## Example

Given `users.xml`:

```xml
<users>
  <user id="1" active="true">
    <name>Alice</name>
    <roles><role>admin</role><role>user</role></roles>
  </user>
  <user id="2">
    <name>Bob</name>
    <roles><role>user</role></roles>
  </user>
</users>
```

```sh
mq -L . -I xml 'import "xpath" | xpath::xpath_query(., "user[@active]/name/text()")' users.xml
# => ["Alice"]

mq -L . -I xml 'import "xpath" | xpath::xpath_query(., "//role/text()")' users.xml
# => ["admin", "user", "user"]

mq -L . -I xml 'import "xpath" | xpath::xpath_first(., "user[@id=\"2\"]/name/text()")' users.xml
# => Bob
```

## Compatibility

Requires [mq](https://github.com/harehare/mq) v0.6 or later.

## License

MIT
