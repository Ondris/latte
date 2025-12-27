# Context-sensitive escaping

Latte's automatic escaping depends on **where** the output lands (HTML text vs an
attribute vs `<script>` vs `<style>` vs a URL), and it is implemented across **two
layers that must stay in sync**.

## The two-place invariant

- **Compile time** — `Compiler/Escaper.php` runs a context state machine
  (`enterHtmlText`/`Tag`/`Attribute`/`Comment`/`Raw`/`BogusTag`) and, per
  `(contentType, state)`, emits a string that **calls** a runtime escaping function
  (e.g. `LR\HtmlHelpers::escapeText(...)`, `LR\Helpers::escapeCss(...)`; plain-text
  output goes through the `escape` filter instead).
- **Runtime** — the actual escaping functions live in `Runtime\Helpers` (JS / CSS /
  iCal) and `Runtime\HtmlHelpers` / `XmlHelpers` (HTML / XML). The routines are
  single-source there, but the **context→function mapping exists twice** inside
  `Compiler\Escaper`: the `escape()`/`escapeMandatory()` match tables drive
  compile-time escaping, while the separate `Convertors` constant behind
  `Escaper::getConvertor` drives only the runtime content-type conversion
  (`Helpers::convertTo(FilterInfo, $dest, $s)`) — `escape()` does not consult
  `getConvertor`.

**Therefore: changing escaping in only one layer is a silent hole.** A new content
type or a changed escaping rule must be reflected in *both* `Compiler\Escaper` and the
`Runtime\*` helpers, or generated templates will call an escaper that behaves
differently than the compiler assumed.

## How nodes obtain the context: the `PrintContext` escaper stack

At print time the current escaping context lives in a **stack of `Escaper`s inside
`PrintContext`** (seeded by its constructor). `%escape` in `format()` resolves
against the stack's top. The trap: **`getEscaper()` returns a *clone*** — mutating
it (calling `enter*`) does not change the ambient context. A node that really needs
to switch context for nested output must push/pop via
`beginEscape()`/`restoreEscape()` and mutate the escaper `beginEscape()` returns.

## Dynamic attributes bypass the state machine

A whole HTML attribute generated from an expression (`Html\ExpressionAttributeNode`)
is not escaped via `Escaper` at all. The node picks a
`Runtime\HtmlHelpers::format*Attribute` method at compile time — the attribute *type*
(`bool`/`list`/`data`/`aria`/`style`/`json`) comes from the attribute name via
`classifyAttributeType`, or is forced by an attribute modifier (`|toggle`, `|json` as
the **last** modifier); XML always uses `XmlHelpers::formatAttribute`. These
formatters do their own escaping (`escapeAttr`, JSON smart-quoting), so they are a
**third place** the escaping contract lives — audit them alongside the two layers
above.

## `ContentType` and contextual filters

`ContentType` is **not a PHP enum** — it is a `final class` of string constants
(`Html='html'`, `Text='text'`, `JavaScript`, `Css`, `Xml`, `ICal`). A **contextual
filter** takes a `FilterInfo` first parameter and may set `$info->contentType`; that
is how a filter declares "my output is already HTML" (which disables further
auto-escaping) or asserts what input type it accepts (`FilterInfo::validate`). Filters
applied to a `{block}` must be contextual for exactly this reason — the block's
content type has to flow through them.
