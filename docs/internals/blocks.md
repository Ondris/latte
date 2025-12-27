# Blocks & template inheritance

Blocks are organized into **layers** on the runtime `Template`, and the way they
merge up an `{extends}` chain determines a subtle, non-local availability model.

## Layers and resolution

The layer keys are `LayerTop` (**the integer `0`**, not a string), `LayerLocal`
(`'local'`), and `LayerSnippet` (`'snippet'`) — there is no "inner" layer. A block
lookup resolves **local before top**: `blocks[LayerLocal][$name] ?? blocks[LayerTop][$name]`.
Both `{block}` and `{define}` register into `LayerTop`; **only the `local` keyword**
(`{block local name}`, `{define local name}`) targets `LayerLocal`. The
block/define difference lies elsewhere: `{block}` also renders in place, `{define}`
only registers.

## Merging up the extends chain (by reference)

When a template `{extends}` a parent, `createTemplate` copies the **parent's**
`LayerTop` blocks into the child's registry (`addBlock`, appending to `functions` so
the child's override stays first, checking content-type compatibility via the
convertor) and then **binds the parent's `LayerTop` by reference to the child's**, so
the whole chain shares one top-block registry. `LayerLocal` is **not** merged up.

## The availability model when rendering a single block

This is the emergent, empirically-verified fact:
`{include X from 'target'}` runs `target->render('X')` — **only that block, not
`main()`**. So availability is exactly the runtime's, not a safe superset:

- **Imported blocks are NOT available.** `{import}` executes inside `main()`, which
  does not run when a single block is rendered — so a block that exists only via
  `{import}` fails at runtime when reached through `include ... from`.
- **A local block of an *intermediate* template is NOT available.** Only `LayerTop`
  merges up, and the final lookup runs on the topmost parent, so only *its*
  `LayerLocal` applies.

The linter's `TemplateReferenceCheck` currently verifies only that **statically
named template paths exist** (`{include}`, `{import}`, `{extends}`/`{layout}`,
`{embed file}`, `{sandbox}`, and the `from` of `{include block from}`), resolving
them through the loader's `getReferredName`; anything not statically determinable —
a dynamic name, `{extends auto}`, a loader that cannot resolve references — is
silently skipped rather than guessed. It does **not** check block names against the
availability model above. (Note the linter checks are first-class `Linting\Check`s
owned by the `Linter`, not compiler passes — the engine only compiles; the linter
inspects the parsed AST and reports value-object `Issue`s.)
