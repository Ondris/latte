# Compilation pipeline

Source → `TemplateLexer` → `TemplateParser`(+`TemplateParserHtml`, `TagParser`) →
AST → passes → `TemplateGenerator` → PHP class.

## The lexer is a state machine the *parser* drives

`TemplateLexer` is a stack of states, each a `state<Name>()` method with a single
PCRE; `states[0]` is current and dispatch is by dynamic method name. **The lexer does
not switch state by content — the parser does**, via `setState`/`pushState`/
`popState` (mostly in `TemplateParserHtml`: opening a tag → `StateHtmlTag`, a quoted
attribute → `StateHtmlQuotedValue`, a comment → `StateHtmlComment`).

Two facts worth pinning down:

- **Raw-text at the lexer level is only `script` and `style`** (`ElementNode::isRawText`
  = HTML content type and `script`/`style`). `textarea`/`pre` are **not** lexer
  raw-text — they only matter to the runtime whitespace minifier. Do not write
  "script/style/textarea" for the lexer.
- **UTF-8 is assumed and enforced.** Every tokenizing PCRE carries the `u` modifier
  (only `normalize()`'s own validation patterns are deliberately byte-oriented), and
  `normalize()` strips the BOM, converts `\r\n` to `\n`, and throws on invalid UTF-8
  or a control character.

## Code generation: `PrintContext::format`

`format()` assembles PHP from placeholders: `%node` (calls the node's `print()`,
parenthesizing sub-assignment-precedence operators), `%dump` (export a value),
`%raw` (verbatim string), `%args` (an `ArrayNode` as call arguments), `%line`
(a `/* pos X:Y */` comment), and `%escape(...)`/`%modify(...)`/`%modifyContent(...)`.
Positional references (`%0.node`, `%2.raw`) select arguments by index; note `%escape`
is special-cased to always mean `%0.escape`.

Temporary PHP variables use the **`$ʟ_` prefix** (a Unicode char users won't collide
with), and `generateId()` returns a per-`PrintContext` counter for unique names when
tags nest.

## `getIterator()` must yield references — a hard invariant

`Node::&getIterator(): \Generator` declares the **reference return in the abstract
signature**, so every node must `yield` its children by reference. `NodeTraverser`
relies on it:

```php
foreach ($node as &$subnode) {
    $subnode = $this->traverseNode($subnode);
}
```

The write-back replaces or removes a child *only* because the child was yielded by
reference. **A node that yields by value makes every replace/remove pass silently a
no-op on its children** — including the sandbox pass. This is the single easiest way
to introduce a security-relevant hole while writing a custom tag.

## `AuxiliaryNode` hides generated code from passes

`AuxiliaryNode` (statement and expression variants) carries a **closure** as its
`print`, whose body is opaque to compiler passes — passes cannot see or rewrite the
PHP it emits. Its input nodes are passed **separately** and *are* traversable via
`&getIterator`. The sandbox itself uses this to wrap already-checked call arguments
so the generated calling code is not re-visited by its own pass. The trap: anything
you build inside an `AuxiliaryNode` closure is invisible to security passes, so route
any user expression through the separate `$nodes` list, never bake it into the
closure body.

## Grammar limitation: `{include expr (args)}`

Because Latte expressions are full PHP expressions, `{include $name (args)}` parses
`$name (args)` as a **first-class-callable / invocation**, gluing the arguments into
the expression so the include's own parameters vanish. This is not a bug but a direct
consequence of the grammar; the comma before include parameters is optional, so the
rule is: when the template name is an expression, **always** separate parameters with
a comma (`{include $name, arg: 1}`). For new tags,
`TagParser::consumeCommaBeforeArguments()` is the canonical way to require (or
soft-deprecate the absence of) that comma.

## Feature flags act at three different stages

The `Feature` enum (`StrictTypes`, `StrictParsing`, `MigrationWarnings`,
`ScopedLoopVariables`, `Dedent`, toggled via `Engine::setFeature`) is the one channel
for opt-in behavior changes, but each flag hooks into a *different* stage:

- **parse time** — `Engine` copies `StrictParsing`/`Dedent` onto the parser
  (`$parser->strict`, `$parser->dedent`); `MigrationWarnings` is also consulted
  during parsing/passes via the engine.
- **print time** — nodes ask `PrintContext::hasFeature()` (e.g. `ForeachNode` checks
  `ScopedLoopVariables`); the flags travel into `PrintContext` as a constructor
  argument.
- **class generation** — `StrictTypes` is consumed by `generateCode()` (emits
  `declare(strict_types=1)`).

Every flag is part of the configuration signature (below), so toggling one
automatically compiles into a different cache file — no manual invalidation needed.

## The compiled-template cache: two signatures with different jobs

`Runtime/Cache.php` + `Engine` implement caching with **two separate signatures**;
confusing them is the main trap:

- The **configuration hash** (`Engine::generateConfigurationSignature`: contentType,
  features, syntax, and each extension's `getCacheKey`) is baked into the template
  **class name and cache file path**. Different configurations therefore coexist as
  different files. **An extension that changes generated code must reflect that in
  `getCacheKey`**, or a stale cache silently survives the configuration change.
- The **refresh signature** (`Cache::generateRefreshSignature`: Latte version, the
  template's *source content*, and the mtimes of extension class files) is checked
  only when `autoRefresh` is on — and it is stored **inside the `.lock` file**, which
  is a data carrier, not just a lock.

Atomicity is platform-split (documented in the source): on Linux the cache file is
included *without* a shared lock, so it must appear atomically via `rename()` of a
`.tmp` file; on Windows a file cannot be renamed over while open, so the lock is
always acquired there. After writing, `opcache_invalidate` is called. With no cache
directory set, the compiled code runs through `eval` on every request instead.
