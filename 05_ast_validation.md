# AST Validation (Layer 1)

> **Note:** the brief said tree-sitter. This codebase uses Python's **standard library `ast` module**.
> There is no tree-sitter dependency. Don't claim it.

---

## 1. Interview Answer (2–3 minutes)

> Layer 1 of my test harness is deterministic structural validation using Python's built-in `ast`
> module. The reasoning is that when you're testing an LLM, most of what you care about is fuzzy —
> but not all of it. Some properties are hard facts about the code: does it define a class that
> inherits from `Strategy`? Does `generate_signals` have a return statement? Does it call `eval` or
> `subprocess`? Those are structural, they're objectively checkable, and they're free to check.
>
> So I parse the generated source into an AST and run a registry of predicates over it. Each predicate
> has a uniform signature — takes the tree and the source, returns a boolean plus an evidence string.
> The evidence part matters: a failing assertion that just says `False` is nearly useless when you're
> debugging why an agent regressed at 2am. Mine says "no class inheriting from Strategy found" or
> "forbidden subprocess call: os.system."
>
> I used AST rather than regex deliberately. Regex operates on text; ASTs operate on structure. If I
> grep for `class.*Strategy` I match a string in a docstring, a comment, and a variable named
> `MyStrategyConfig`. If I walk the AST looking for a `ClassDef` whose `bases` contain a `Name` with
> id `Strategy`, I match the actual language construct and nothing else. And because I get a syntax
> error for free from `ast.parse`, "is this even valid Python" is the first thing I know.
>
> There are about twenty-four predicates. Five generic ones live in the harness core; nineteen
> SDK-specific ones ship in a separate plugin package, because I wanted the harness to be reusable for
> any code-generating agent, not welded to our SDK.

---

## 2. What Problem Were We Solving?

**Business.** Structurally invalid generated code means a broken backtest and a user who loses trust
immediately. These are the cheapest failures to catch and the most embarrassing to ship.

**Technical.** You need assertions that are:
- **deterministic** — the same code always yields the same verdict, so no flake
- **free** — no API call, so they can run on every rep of every case
- **precise** — matching language constructs, not text that looks like them
- **explanatory** — a failure has to tell you what's wrong

**Why not just use the LLM judge for everything?** Three reasons. It costs money per call. It's
nondeterministic, so it introduces flake into checks that don't need it. And it's *worse* at this
specific job — asking a language model "does this file contain a subprocess call" is strictly less
reliable than walking the tree, and you'd be paying for the privilege of a less reliable answer.

**Why not regex?** See section 4 — this is the question you'll actually be asked.

---

## 3. Architecture

```
generated backtest.py (string)
        │
        ▼
   ast.parse(source)
        │
        ├── SyntaxError ──► [{name: "parses_to_python",
        │                      passed: False, evidence: str(e)}]
        │                    (short-circuit — nothing else can run)
        ▼
   ast.Module tree
        │
        ▼
┌───────────────────────────────────────────────────┐
│ run_all_predicates(predicate_specs, source)       │
│                                                   │
│  for spec in specs:                               │
│    fn = _REGISTRY[spec["name"]]                   │
│    passed, evidence = fn(tree, source=source,     │
│                          **spec.get("args", {}))  │
└───────────────────┬───────────────────────────────┘
                    ▼
      [{name, passed, evidence}, ...]
                    │
                    ▼
        layer1_passed = all(passed)
        ANY False → case fails hard, judge never runs


REGISTRY POPULATION
  layer1.py         ──register_predicate──►  _REGISTRY  ◄──register()── quantjo_plugin
  5 generic                                  (dict)                     19 SDK-specific
                                                                        + B1–B6 boundaries
```

**Helper layer** (`harness/ast_utils.py`) sits under the predicates so each one stays a few lines:

```
find_class(tree, "Strategy")        → ClassDef whose bases contain Strategy
find_method(cls, "generate_signals")→ FunctionDef inside a class
class_attr_value(cls, "assets")     → RHS of a class-level assignment (Assign or AnnAssign)
walk_calls(node)                    → every ast.Call
call_name(call)                     → "os.system" from nested Attribute chains
string_literals_in(node)            → all string constants
subscript_keys(dict_node)           → string keys of a dict literal
assigned_column_names(func)         → df['entry_long'] = ... → {"entry_long"}
has_for_loop_with_range(func)       → detects `for i in range(len(df))`
keyword_value(call, "period")       → kwarg AST node
constant_value(node)                → literal value or None
```

---

## 4. Deep Technical Explanation

### Why AST and not regex — the real answer

This is the question. Four distinct arguments:

**1. Regex matches text; AST matches meaning.**

```python
# Regex r"class\s+\w+\(Strategy\)" matches ALL of these:
"""This module defines a class Foo(Strategy)."""   # a docstring
# class Old(Strategy)  — deprecated                # a comment
example = "class Demo(Strategy)"                   # a string literal
```

The AST walk sees a `ClassDef` node or it doesn't. Docstrings, comments, and string literals are
different node types entirely — comments aren't even in the tree.

**2. Regex can't handle nesting or arbitrary structure.**

`call_name()` resolves `a.b.c.d()` by walking a chain of `ast.Attribute` nodes into a `Name`. A regex
for "does this call `subprocess.run`" has to anticipate whitespace, line continuations, aliased
imports, and nesting depth. The AST version is a loop over a linked structure — and it's correct by
construction.

**3. You get syntax validation for free.**

`ast.parse()` raising `SyntaxError` *is* the "is this valid Python" check. No separate step. Note
that `parses_to_python` in the registry unconditionally returns `True` — because if the source didn't
parse, `run_all_predicates` already short-circuited before any predicate ran. That's a small design
detail worth being able to explain; it looks like a bug until you see the control flow.

**4. Semantic queries are expressible at all.**

`assigned_column_names()` finds every `df['some_col'] = ...` by looking for `Assign` nodes whose
target is a `Subscript` with a string constant slice. Try that with regex across multi-line pandas
chains and you will fail on the third real example.

**The honest limit:** AST is *syntactic*, not semantic. It tells you a `for i in range(len(df))` loop
exists. It cannot tell you the loop is correct, or that the strategy is profitable, or that the stop
loss is on the right side of the entry. That's exactly why Layer 2 exists. Say this — knowing what
your tool *can't* do is the mature half of the answer.

### The predicate contract

```python
PredicateFn = Callable[..., tuple[bool, str]]
# fn(tree: ast.Module, *, source: str, **kwargs) -> (passed, evidence)
```

Three deliberate properties:

- **`source` is always passed** even though most predicates only need the tree — so a predicate that
  needs raw text (checking formatting, or Jupytext `# %%` markers) doesn't need a different signature.
- **`**kwargs` enables parameterization.** `uses_indicator_config_api(indicator="sma")` and
  `assets_primary_interval(interval="1h")` are the same predicate reused with args from YAML.
- **Evidence is mandatory.** `(False, "no class inheriting from Strategy found")` versus a bare
  `False`. This is what makes a 20-case failing report actionable rather than a puzzle.

### The registry and plugin boundary

```python
_REGISTRY: dict[str, PredicateFn] = {}
def register_predicate(name, fn): _REGISTRY[name] = fn
```

Five generic predicates register in `layer1.py`. Nineteen SDK-specific ones register from
`quantjo_plugin` through the entry-point mechanism.

*Why the split?* The core harness knows nothing about trading. `defines_strategy_subclass` and
`safe_python` apply to any Python-generating agent. `uses_indicator_config_api` and
`aligns_secondary_asset` are meaningless outside this SDK. Keeping them in a plugin means the harness
could be pointed at a different agent by swapping one package — and it forced a clean interface
rather than a pile of special cases.

*Failure handling:* an unknown predicate name returns
`(False, "unknown predicate 'x' — install the required plug-in?")` rather than raising, so one
misconfigured case doesn't kill a 20-minute run. And every predicate call is wrapped:

```python
except Exception as exc:
    return (False, f"predicate error: {exc}")
```

A buggy predicate fails its own assertion instead of crashing the suite.

### The predicate catalogue

**Generic (5, in `layer1.py`):**

| Predicate | Checks |
|---|---|
| `parses_to_python` | Valid Python (implicit — short-circuit guarantees it) |
| `defines_strategy_subclass` | A `ClassDef` with `Strategy` in `bases` |
| `has_assets_dict` | Class attr `assets` is a `Dict` containing key `'primary'` |
| `has_generate_signals` | Method `generate_signals` exists and contains a `Return` |
| `safe_python` | No `eval`, `exec`, `__import__`, `os.system`, `subprocess.*` |

**SDK-specific (19, in `quantjo_plugin/predicates.py`):**

`imports_from_quantjo_sdk`, `uses_indicator_config_api`, `computes_indicator_manually`,
`assets_primary_interval`, `has_exit_rules`, `uses_inline_exit_columns`, `not_uses_exit_rules`,
`assets_dict_keys`, `aligns_secondary_asset`, `declares_pane`, `has_session_filter`,
`has_inline_state_machine`, `is_purely_vectorised`, `declares_pane_for_manual`, plus three
correctness-trap predicates: `qc001_vwap_has_session_reset`, `qc002_stop_direction_correct`,
`qc003_entry_exit_zones_disjoint`.

Those last three are the most interesting — they encode specific bugs. `qc002_stop_direction_correct`
checks that a stop loss is on the correct side of entry for the trade direction, because inverting it
is a plausible LLM error that produces code which runs fine and loses money. That's a *domain* check
expressed structurally, and it's a good thing to mention: AST validation isn't only for generic
hygiene.

### `safe_python` — validation as a security control

```python
forbidden = {"eval", "exec", "__import__"}
# plus os.system, subprocess.run/Popen/call
```

Generated code gets executed by the backtest service. LLM-generated code executed server-side is a
genuine RCE surface — whether through a prompt injection or an innocent hallucination. This predicate
is a defense-in-depth layer, not the primary control.

**Be honest about its limits when asked.** It's a denylist over direct call names, and denylists are
bypassable: `getattr(__builtins__, 'ev' + 'al')`, `importlib.import_module`, obfuscated attribute
access. It catches the accidental and the naive, not a determined adversary. The real control is
sandboxed execution — see [12](12_security.md).

---

## 5. My Actual Contribution

**What existed** — nothing. No AST validation anywhere in either repo.

**What I built** — `harness/ast_utils.py` (the 11-function helper layer), `assertions/layer1.py` (the
registry, dispatcher, and 5 generic predicates), and `plugins/quantjo_plugin/src/quantjo_plugin/
predicates.py` (19 SDK-specific predicates including the three QC traps).

**What changed** — structural correctness became a free, deterministic, hard gate that runs before
any paid judge call.

**Result** — zero-flake structural assertions with actionable evidence strings, a plugin boundary
that keeps the harness agent-agnostic, and a short-circuit that avoids paying for judge calls on code
that was never going to pass.

---

## 6. Files and Code

| Path | Contents |
|---|---|
| `harness/ast_utils.py` | `find_class`, `find_method`, `class_attr_value`, `walk_calls`, `call_name`, `string_literals_in`, `subscript_keys`, `assigned_column_names`, `has_for_loop_with_range`, `keyword_value`, `constant_value` |
| `assertions/layer1.py` | `PredicateFn`, `_REGISTRY`, `register_predicate`, `get_predicate`, `run_predicate`, `run_all_predicates`, 5 generic predicates |
| `plugins/quantjo_plugin/src/quantjo_plugin/predicates.py` | 19 SDK predicates |
| `plugins/quantjo_plugin/src/quantjo_plugin/boundary_pairs.py` | B1–B6 → predicate pairs |
| `plugins.py` | Entry-point discovery |
| Case YAMLs | `predicates: [{name, args}]` per case |

`ast` nodes used: `Module`, `ClassDef`, `FunctionDef`, `Assign`, `AnnAssign`, `Name`, `Attribute`,
`Call`, `Constant`, `Dict`, `Subscript`, `Return`, `For`.

---

## 7. Design Decisions

| Decision | Why | Tradeoff |
|---|---|---|
| Stdlib `ast` over tree-sitter | Zero deps; generated code is always Python | Python-only; no error recovery on invalid syntax |
| `(bool, str)` return | Failures must be self-explaining | Slightly more verbose predicates |
| Registry + plugin entry point | Harness stays agent-agnostic | Indirection; a layer of discovery to debug |
| Unknown predicate → fail, not raise | One bad case shouldn't kill a long run | A typo silently fails instead of loudly |
| Predicate exceptions caught | Robustness of the run | Can mask a genuinely broken predicate |
| Layer 1 hard-gates | Structure is objective | No partial credit for nearly-right code |
| Denylist in `safe_python` | Cheap defense-in-depth | Bypassable; not a real sandbox |
| `source` always passed | Uniform signature for text-needing predicates | Unused parameter in most |

---

## 8. Production Improvements

- **Replace the denylist with real isolation.** gVisor/Firecracker or a subprocess with `seccomp`,
  no network, read-only FS, a memory cap. Then `safe_python` becomes a fast pre-filter rather than a
  control you depend on.
- **Import allowlisting** — walk `Import`/`ImportFrom` and permit only `quantjo_sdk`, `pandas`,
  `numpy`. Structurally stronger than denying call names, and it directly enforces the "SDK only"
  platform constraint that today lives only in the prompt.
- **Reuse the parsed tree.** `run_all_predicates` parses once — good — but Layer 3 re-parses in
  `run_pair_check`. Pass the tree through.
- **Predicate unit tests.** The predicates are themselves untested code that gates everything else.
  Fixture files with known-good and known-bad snippets would take an hour.
- **Coverage reporting** — which predicates never fire across a full run? Those are either dead or
  untested.
- **`ast.unparse` in evidence** — quoting the offending source line rather than describing it makes
  failures instantly readable.
- **Complexity/lint gates** — cyclomatic complexity, unreachable code, unused imports as advisory
  signals.

---

## 9. Common Interview Questions

1. Why AST instead of regex?
2. Walk me through how you check for a `Strategy` subclass.
3. What can AST validation *not* tell you?
4. Why does `parses_to_python` always return `True`?
5. How do you handle code that doesn't parse?
6. What's the predicate registry for?
7. Why are SDK predicates in a plugin?
8. How do you check for dangerous code?
9. Is your denylist sufficient?
10. What's `call_name` doing and why is it non-trivial?
11. Why does every predicate return an evidence string?
12. How do you parameterize a predicate from YAML?
13. What happens if a predicate raises?
14. What's a QC predicate?
15. How would you detect a stop loss on the wrong side of entry?
16. Why not use `mypy` or a linter instead?
17. What's the performance cost of AST parsing?
18. How do you test the predicates themselves?
19. What would you use if the agent generated JavaScript?
20. When would tree-sitter actually be the right tool?

---

## 10. Strong Answers

**Q1 — AST vs regex.**
> Regex operates on text, AST operates on structure — and for this job that difference is decisive.
> If I grep for `class.*Strategy` I match it inside a docstring, inside a comment, and inside a string
> literal, and I'd have to write increasingly baroque patterns to exclude each. When I walk the AST
> for a `ClassDef` whose `bases` list contains a `Name` node with id `Strategy`, I'm matching the
> language construct itself — comments aren't even in the tree. I also get syntax validation for free,
> since `ast.parse` raising `SyntaxError` *is* the "is this valid Python" check. And some things
> aren't expressible in regex at all: resolving `a.b.c.d()` into a dotted name means walking a chain
> of `Attribute` nodes, which is a loop over a structure, not a pattern match.

**Q3 — What AST can't tell you.**
> Anything semantic. It tells me a bar-by-bar loop exists; it can't tell me the loop is correct. It
> tells me `ExitRule` was used; it can't tell me the trader asked for a trailing stop and a fixed stop
> is the wrong answer. It tells me a stop-loss variable exists; it took a hand-written predicate to
> check the stop is on the correct *side* of the entry, and even that only works because I could
> express it structurally. That gap is precisely why Layer 2 exists — the judge scores intent
> fidelity, which is fundamentally not a syntactic property. Layer 1's job is to be cheap, certain,
> and narrow.

**Q4 — Why does `parses_to_python` always return True?**
> Because by the time any predicate runs, parsing already succeeded. `run_all_predicates` calls
> `ast.parse` up front inside a try; on `SyntaxError` it returns immediately with a single failing
> result named `parses_to_python` carrying the syntax error as evidence, and no predicate executes.
> So the registered predicate is really a name that lets a case *declare* the parse requirement in
> YAML, and it's unconditionally true in the only path where it can be reached. I'd concede it reads
> like a bug in isolation — a comment there would have been worth writing.

**Q9 — Is the denylist sufficient?**
> No, and I wouldn't want it presented as a security boundary. It's a denylist of direct call names —
> `eval`, `exec`, `__import__`, `os.system`, `subprocess`. It catches accidental dangerous code and a
> naive injection. It does not stop `getattr(__builtins__, 'ev'+'al')`, or `importlib`, or anything
> obfuscated. It's defense in depth, and it's cheap, so it's worth having. The actual control has to
> be execution isolation — a sandboxed process with no network, a read-only filesystem, and a memory
> cap. The structurally stronger static check would be an import allowlist rather than a call
> denylist, since that's an enumeration of what's *permitted*, and it would also enforce the
> "SDK only, no pandas-ta or ta-lib" rule that currently lives only in the prompt.

**Q20 — When would tree-sitter be right?**
> Three situations. If I needed to parse multiple languages, tree-sitter gives one uniform interface
> across grammars where `ast` is Python-only. If I needed to parse *broken* code — tree-sitter has
> error recovery and produces a partial tree, whereas `ast.parse` throws and gives you nothing, which
> matters a lot for editor tooling and incremental parsing. And if I needed byte-level source ranges
> for highlighting. None of those applied: our agent emits complete Python files, and if it emits
> invalid Python that's a hard failure I want to report, not recover from. So the stdlib was the right
> call — zero dependencies, and it's the same parser that will actually run the code.

---

## 11. Follow-up Questions

- "Your predicates gate everything but are themselves untested. Does that worry you?"
- "`has_assets_dict` checks for a `'primary'` key. What if the SDK renames that role?"
- "Layer 3 re-parses code Layer 1 already parsed. Why?"
- "How do you handle Jupytext `# %%` markers — do they survive `ast.parse`?"
- "Could an agent pass all your predicates and still generate useless code?"
- "What's your parse time on a 500-line strategy, and does it matter?"
- "Walk me through `qc002_stop_direction_correct`. How do you determine 'side' structurally?"

---

## 12. Senior Engineer Perspective

**The layering instinct is the signal.** Putting deterministic, free checks before nondeterministic,
paid ones is the same discipline as types before unit tests before integration tests. A senior
engineer will register that immediately.

**They'll push on the plugin boundary.** Is it real abstraction or premature generalization? The
defensible answer: it forced a clean predicate interface and made the harness reusable, and it cost
almost nothing — one registry dict and an entry point. Generalization is only premature when it costs
more than it returns.

**They'll ask about the security posture.** The right answer distinguishes *defense in depth* from
*a security boundary*. Candidates who present a denylist as sufficient reveal they haven't thought
adversarially.

**They'll notice the QC predicates.** `qc002_stop_direction_correct` is domain knowledge encoded as a
structural check — that's the kind of thing that only exists when an engineer understood the business
problem, not just the technical one. Mention it unprompted.

---

## 13. Common Mistakes

- **Claiming tree-sitter.** Not in the repo. And the stdlib was the *better* choice here — you have a
  stronger answer available than the one the brief suggested.
- **"AST is more robust than regex"** with no example. Give the docstring/comment/string-literal
  example; it lands instantly.
- **Overselling `safe_python`.** Calling it a security control rather than defense in depth is a
  red flag to anyone who thinks about threat models.
- **Not knowing the limits.** If you can't say what AST *can't* do, you don't understand why Layers 2
  and 3 exist.
- **Describing predicates generically.** Name three. `defines_strategy_subclass`,
  `aligns_secondary_asset`, `qc002_stop_direction_correct` — specificity is credibility.

---

## 14. Cheat Sheet

- **Tool:** Python stdlib `ast`. Not tree-sitter. Zero dependencies.
- **Contract:** `fn(tree, *, source, **kwargs) -> (bool, str)` — verdict **and evidence**.
- **Flow:** `ast.parse` → SyntaxError short-circuits → else run each registered predicate →
  `all(passed)` → hard gate before the judge runs.
- **Registry:** `_REGISTRY` dict; 5 generic in core + 19 SDK-specific via `quantjo_plugin` entry point.
- **Generic 5:** `parses_to_python`, `defines_strategy_subclass`, `has_assets_dict`,
  `has_generate_signals`, `safe_python`.
- **Why AST:** matches constructs not text (docstrings/comments/strings don't false-positive) ·
  free syntax validation · handles nested `Attribute` chains · expresses semantic queries like
  `df['col'] = ...`.
- **Limit:** syntactic only. Cannot judge intent, correctness, or profitability → Layer 2 exists.
- **`safe_python`:** denylist (`eval`, `exec`, `__import__`, `os.system`, `subprocess.*`).
  Defense in depth, **not** a sandbox. Real fix = isolated execution + import allowlist.
- **QC predicates:** encode real bugs — VWAP session reset, stop direction inversion, entry/exit
  zone overlap.
