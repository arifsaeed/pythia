Implementation Plan: Generics
Goal
Enable the definition and usage of generic functions, structs, and enums, allowing for code reuse and type safety.

Design Principles
Explicit Syntax: Use <T> for generic parameters, matching Rust and TypeScript (familiarity).
Monomorphization-ready: The design will enforce that generics are verified at compile time.
Simplicity: Support basic generics first (no trait bounds yet), adding bounds in a later phase.
Proposed Changes
[MODIFY] 
crates/ast/src/lib.rs
Add generics: Vec<String> to 
FnDecl
, 
StructDecl
, 
EnumDecl
.
[MODIFY] 
crates/parser/src/items.rs
Update 
parse_fn
, 
parse_struct
, 
parse_enum
 to parse <Identifier, ...> after the name.
[MODIFY] 
crates/hir/src/lib.rs
Update 
HirFunction
, 
HirStruct
, 
HirEnum
 to include generics: Vec<String>.
Enable HirType::GenericParam(String).
[MODIFY] 
crates/hir/src/lowering.rs
Update 
lower_fn
, lower_struct, lower_enum to copy generic parameters.
Update 
lower_type
 to resolve paths to HirType::GenericParam if they match active generic parameters.
[MODIFY] 
crates/type_check/src/check.rs
Inference: Implement unification for function calls to infer generic type arguments (e.g., 
id(5)
 infers T=Int).
Instantiation: Replace generic parameters with concrete types during call checks.
Verification Plan
Automated Tests
crates/parser/tests/generics.rs: Test parsing of fn id<T>(x: T).
crates/hir/src/lowering.rs
: Test lowering of generic params.
crates/type_check/src/check.rs
: Test inference of generic function calls.
Manual Verification
Create a simple generic function and call it with different types.


#########################################################################################

PZero — Development Plan
LLM-Optimized Programming Language
1. Vision Statement
PZero is a programming language designed for LLM AI agents as the primary developer, not humans. Every design decision optimises for how language models generate, reason about, and verify code. Humans remain in the loop as architects, spec writers, and reviewers — but the language itself is built for machines to write correctly on the first attempt.

2. Transpilation Target: Explicit Recommendation
Recommendation: Transpile to a restricted subset of Rust
This is the explicit recommendation and the reasoning follows.

What "restricted subset of Rust" means
The generated Rust code will deliberately avoid hard-to-generate Rust features. Specifically:

Feature	Strategy in generated code
Ownership / borrowing	Avoid. Use Clone and Arc<T> everywhere.
Lifetime annotations	Never generated. All data is either owned or Arc-wrapped.
Mutable references (&mut)	Rarely generated. Functional core means most data is immutable.
Sum types / enums	Direct mapping — PZero ADTs → Rust enums (natural fit).
Pattern matching	Direct mapping — PZero match → Rust match.
Traits	Used for PZero interfaces/typeclasses.
Error handling	PZero Result<T, E> → Rust Result<T, E> directly.
Memory allocation	Heap-allocate freely via Box<T> and Arc<T>. Optimise later.
Async	Not in v1. Add later via tokio codegen if needed.
Why Rust over the alternatives
Target	Verdict	Reason
Rust (restricted subset)	✅ CHOSEN	Best semantic match. Enums, pattern matching, Result/Option, no null, strong types — all map 1:1 from PZero. Generated code compiles to native binaries. Access to cargo ecosystem.
Go	❌ Rejected	No sum types, no pattern matching, nil everywhere. You'd spend more effort emulating PZero semantics in Go than just generating restricted Rust.
C	❌ Rejected	Manual memory management codegen is a project in itself. No type-level safety in the output.
LLVM IR	❌ Rejected	Massive complexity. You'd be writing a full compiler backend. Kills achievability.
Cranelift / WASM	⏸️ Future option	Worth revisiting in v2 for portable/sandboxed execution. Too much infrastructure for v1.
The key insight
Generating idiomatic Rust is hard. Generating valid, compiling Rust that uses Arc<T> and Clone liberally is straightforward. The generated code won't win style points, but it will:

Compile reliably
Run correctly
Produce native binaries
Benefit from Rust's optimiser (LLVM backend)
Integrate with the cargo ecosystem
Performance tuning the generated code is a v2+ concern. Correctness and achievability come first.

Risk mitigation
If restricted-Rust codegen proves harder than expected in practice (e.g., trait resolution edge cases, orphan rules), the fallback is:

Fallback: Transpile to C with a simple reference-counted GC

C is universally compilable, and a reference-counted allocator for an immutable-first language is well-understood. This fallback exists as insurance, not as the plan.

3. Core Design Principles (Prioritised)
These are ordered by implementation priority — build the foundation first, then layer on advanced features.

Priority	Principle	Phase
P0	Familiar-but-unique syntax identity	Phase 1
P0	Explicit, minimal grammar (PEG-style, unambiguous)	Phase 1
P0	Strong, static typing with type inference	Phase 1–2
P0	Explicit scope, bracketed blocks, no implicit context	Phase 1
P1	No null — Option/Result types mandatory	Phase 2
P1	Functional core + explicit effects	Phase 2
P1	Canonical formatting (single valid style)	Phase 2
P1	First-class testing & TDD workflow	Phase 3
P1	Dependency context generation for LLM debugging	Phase 2–3
P2	AI-optimised structured error feedback	Phase 3
P2	Intent-aware error recovery & suggestions	Phase 3
P2	Type-level contracts (refinement types, pre/post)	Phase 4
P2	Built-in documentation as structured data	Phase 4
P3	Mocks, stubs, DI via effect handlers	Phase 5
P3	Multi-agent coordination primitives	Phase 6+
4. Specific Objective: Familiar-But-Unique Syntax Identity
Problem
If PZero syntax looks identical to Rust, LLMs will generate Rust semantics. If it looks identical to Go, they'll assume Go semantics. The syntax must be close enough to trigger correct structural patterns but distinct enough that the LLM recognises it as PZero and applies PZero-specific rules.

Strategy: "Rust-shaped, uniquely keyed"
The syntax borrows structural patterns from the C-family (braces, semicolons, fn, typed parameters) because LLMs have billions of training tokens in this style. But it introduces deliberate, consistent markers that signal "this is PZero":

Familiarity anchors (keep these from Rust/Go/TS)
{} for blocks
; as statement terminator (mandatory, not optional)
fn for function declarations
let for bindings
if / else / match for control flow
Type annotation syntax: name: Type
// for line comments
Unique identity markers (things that make PZero recognisable)
These are suggestions to evaluate during syntax design — the exact choices are Phase 1 deliverables:

Concept	Familiar form	PZero candidate	Rationale
Module declaration	mod, package, module	module	Explicit, readable
Effect annotation	(none standard)	effect keyword	Signals impure code — unique to PZero
Test block	#[test], describe()	test as first-class block	test "name" { ... } inline with function
Contract/spec	(none standard)	require / ensure	Pre/post conditions as keywords
Pipe operator	|> (Elixir/F#)	|>	Functional composition — familiar from F#/Elixir but rare in C-family
Type alias	type X = Y	type X = Y	Keep familiar
Sum types / ADT	enum (Rust)	enum	Keep familiar
Struct / record	struct (Rust/Go)	struct	Keep familiar
Error handling	? operator (Rust)	? operator	Keep familiar
Pure function marker	(none standard)	inferred — all fn are pure unless effect is declared	Purity by default is the identity
The identity rule
Every PZero function is pure by default. Side effects require the effect keyword.

This single rule is the language's philosophical fingerprint. It means:

LLMs learn: "if I see fn, it's pure — I can reason about it deterministically"
LLMs learn: "if I see effect fn, I need to handle side effects explicitly"
No other mainstream language works this way syntactically
Syntax example (illustrative, subject to Phase 1 design)
module math;

/// Adds two integers.
/// @param a - first operand
/// @param b - second operand  
/// @returns the sum
fn add(a: Int, b: Int) -> Int {
    a + b
}

test "add works" {
    assert_eq(add(2, 3), 5);
    assert_eq(add(-1, 1), 0);
}

require x > 0
ensure result > x
fn double_positive(x: Int) -> Int {
    x * 2
}

enum Shape {
    Circle { radius: Float },
    Rect { width: Float, height: Float },
}

fn area(s: Shape) -> Float {
    match s {
        Shape::Circle { radius } => 3.14159 * radius * radius,
        Shape::Rect { width, height } => width * height,
    }
}

effect fn read_file(path: String) -> Result<String, IoError> {
    fs::read(path)
}
Key signals an LLM would pick up:

module keyword → "not Rust, not Go"
test block inline → "not any mainstream language"
require / ensure as keywords → "this language has contracts"
effect fn → "this is PZero, purity is default"
Everything else (braces, semicolons, fn, match, enum) → familiar muscle memory
Validation approach
During Phase 1, we will test syntax candidates by prompting frontier LLMs with partial PZero code and measuring:

Can the LLM complete the code correctly given a few examples?
Does the LLM confuse PZero with another language?
Does the LLM respect PZero-specific rules (purity, contracts, test blocks)?
5. Specific Objective: Dependency Context for LLM Agentic Development (Authoring & Debugging)
Problem
When an LLM needs to modify existing code or fix a bug, its first task is gathering relevant context. In most languages, this means:

1. Read the failing function
2. Guess which other functions/types are involved
3. Search the codebase (grep, semantic search, file reads)
4. Hope it found everything
5. Often miss a transitive dependency and produce a wrong fix

This is slow, unreliable, and wastes the LLM's context window on irrelevant code. The language should do this automatically.

Solution: The compiler knows the full dependency graph
After type checking, PZero's compiler has a complete, typed dependency graph — it knows exactly which functions call which other functions, which types are used where, which modules depend on which modules. This is the same graph used for effect checking (2.4) and name resolution (2.6).

The context generation feature exposes this graph to the LLM via CLI commands.

How it works: Proactive Context Retrieval (Authoring)
Ideally, an agent should "ask" the language for context *before* making edits.

$ pzero context <symbol> — Get everything needed to understand a symbol

Given a function name (or file:line), the compiler walks the dependency graph and outputs a self-contained context slice: every definition the LLM needs to understand and modify that symbol.

Example: If `process_order` calls `validate_order`, which uses `Order` struct and `ValidationError` enum, and `validate_order` calls `check_stock` which uses `Inventory`:

$ pzero context process_order --format source

// === Context for: process_order (src/orders.p0:45) ===
// === Depth 0: Direct target ===

fn process_order(order: Order) -> Result<Receipt, OrderError> {
    let validated = validate_order(order)?;
    let receipt = charge_payment(validated)?;
    receipt
}

// === Depth 1: Direct dependencies ===

struct Order {
    items: List<Item>,
    customer: Customer,
    total: Float,
}

enum OrderError {
    ValidationFailed { reason: String },
    PaymentFailed { reason: String },
    OutOfStock { item: String },
}

require order.items.len() > 0
fn validate_order(order: Order) -> Result<Order, OrderError> {
    let stock_ok = check_stock(order.items)?;
    if stock_ok { Ok(order) } else { Err(OrderError::OutOfStock { item: "unknown" }) }
}

fn charge_payment(order: Order) -> Result<Receipt, OrderError> { ... }

// === Depth 2: Transitive dependencies ===

struct Item { name: String, price: Float, quantity: Int }
struct Customer { name: String, email: String }
struct Receipt { order_id: String, total: Float }

effect fn check_stock(items: List<Item>) -> Result<Bool, OrderError> { ... }

The output is:

*   **Ordered by dependency depth** — direct target first, then immediate dependencies, then transitive
*   **Self-contained** — an LLM can understand the entire context without reading any other file
*   **Trimmed** — function bodies beyond depth 1 are replaced with `{ ... }` to save context window space (configurable with `--depth N`)
*   **Includes contracts** — `require`/`ensure` blocks are included because they're essential for understanding expected behaviour

pzero context JSON mode — Machine-consumable
$ pzero context process_order --format json
{
  "target": {
    "name": "process_order",
    "file": "src/orders.p0",
    "line": 45,
    "kind": "function"
  },
  "dependencies": [
    {
      "name": "validate_order",
      "file": "src/orders.p0",
      "line": 22,
      "kind": "function",
      "depth": 1,
      "relationship": "called_by_target"
    },
    {
      "name": "Order",
      "file": "src/types.p0",
      "line": 5,
      "kind": "struct",
      "depth": 1,
      "relationship": "type_used_by_target"
    },
    {
      "name": "check_stock",
      "file": "src/inventory.p0",
      "line": 12,
      "kind": "effect_function",
      "depth": 2,
      "relationship": "called_by_validate_order"
    }
  ],
  "source_slice": "// ... complete source as above ...",
  "context_tokens": 847,
  "total_tokens_in_project": 52000
}
The `context_tokens` field tells the LLM exactly how much of its context window this will consume.

How it works: Impact Analysis (Reverse Dependencies)
When modifying a function, an agent needs to know what *other* code will break.

$ pzero context <symbol> --dependents

This walks the graph in the *reverse* direction: finding all functions/modules that call or use the target symbol.

Example: If we plan to change the signature of `functionA`, we run:

$ pzero context functionA --dependents

Output:
// === Dependents of: functionA (src/core.p0:10) ===
// These functions call functionA and may break if it changes.

fn functionB() { ... calls functionA ... }
fn functionC() { ... calls functionA ... }

This allows the agent to:
1.  **Retrieve Dependents**: See all call sites.
2.  **Plan Refactoring**: Update all callers in the same edit or creating a plan to update them.

How it works: Reactive Context Retrieval (Debugging)
pzero diagnose — Error + context in one command
This is the killer feature for LLM debugging workflows. It combines test execution with context generation:

$ pzero diagnose
{
  "test_results": {
    "passed": 14,
    "failed": 1,
    "total": 15
  },
  "failures": [
    {
      "test": "order processing handles empty cart",
      "file": "src/orders.p0",
      "line": 78,
      "error": {
        "code": "R0001",
        "message": "require failed: order.items.len() > 0",
        "trace": "process_order -> validate_order -> require"
      },
      "context": {
        "source_slice": "// ... full dependency context of the failing test ...",
        "depth": 2,
        "context_tokens": 623
      },
      "suggestion": "The test passes an empty items list but validate_order requires items.len() > 0. Either the test expectation is wrong (it should expect an Err) or process_order should handle empty carts before calling validate_order."
    }
  ]
}
The LLM receives:

What failed — structured error with trace
All the code it needs to understand why — dependency context of the failing test/function
A suggestion — based on the contract that failed
Nothing irrelevant — context is precisely scoped, not the entire codebase
Context depth control
Flag	Behaviour
--depth 0	Target function/type only
--depth 1	Target + direct dependencies (default)
--depth 2	Target + direct + transitive dependencies
--depth full	Everything reachable (may be large)
--max-tokens N	Stop expanding context when it would exceed N tokens
The --max-tokens flag is specifically designed for LLMs — the agent can say "give me context but keep it under 4000 tokens" and the compiler will prioritise the most relevant dependencies.

Why this is almost free to implement
The dependency graph is already built by Phase 2 tasks:

Phase 2 task	What it produces	Reused for context generation
2.5 Module resolution	Module dependency graph	✅ Which modules to include
2.6 Name resolution	Symbol → declaration mapping	✅ Which symbols to look up
2.4 Effect checker	Call graph	✅ Which functions call which
2.3 Type checker	Type → usage mapping	✅ Which types are involved
The context generation engine (2.10) is essentially a graph traversal + source extraction on data structures that already exist. The pzero diagnose command (3.13) wraps the test runner output with a context lookup per failure. Neither requires new analysis — just new output formatting.

6. Specific Objective: Integrated Knowledge Base via Recursive Language Models (RLM)
Problem
LLMs often hallucinate language features or get stuck on obscure errors. Traditional RAG (Retrieval Augmented Generation) suffers from "context rot" and imprecise retrieval, often missing the exact rule needed to fix a complex bug.

Solution: RLM-Powered Explanation Engine
Instead of retrieving static chunks, we implement an **RLM (Recursive Language Model)** system alongside the compiler.

Mechanism: `pzero explain` with RLM
$ pzero explain <query>

1.  **Environment**: The system initializes a secure REPL environment (sandboxed Python or PZero interpreter).
2.  **Context Loading**: The Knowledge Base "Master Index" and relevant large documentation files are loaded as variables in the REPL.
3.  **Root LM**: The model receives the user's query and the REPL interface.
    *   It does *not* try to act immediately.
    *   It "peeks" at the `INDEX.md` variable.
    *   It decides to "grep" for specific keywords or "read" specific modules (e.g., `syntax/control_flow.md`).
4.  **Recursion**: The Root LM can spawn recursive sub-calls to summarize or extract data from large doc files.
5.  **Synthesis**: The final answer is constructed from these recursive insights, ensuring extremely high accuracy even for complex, multi-hop language questions.

Use cases:
1. **Error Codes**: `pzero explain E0123` -> Returns detailed markdown explanation of the error, common causes, and fix examples (positive/negative).
2. **Concepts**: `pzero explain effects` -> Returns the "Effects" chapter of the language manual, explaining syntax, semantics, and best practices.
3. **Syntax**: `pzero explain match` -> Returns the syntax definition and examples for match expressions.

Formatting Methodology: "Context as Code"
The documentation (`docs/kb/`) is structured to be "executable" by the RLM:
*   **Monolithic Files**: Related topics (e.g., all control flow) are one file to maintain context.
*   **Standardized Headers**: H2/H3 headers act as "anchors" for the RLM to regex-search.
*   **Master Index**: A single source of truth that maps concepts to file paths.

Integration with `diagnose`
When `pzero diagnose` fails, it passes the error and the *entire* relevant knowledge base (via the RLM environment) to the agent, allowing it to "research" the fix dynamically rather than relying on a static retrieval.



7. Phased Development Plan
Phase 1 — Foundation: Lexer, Parser, AST (Months 1–2)
Goal: Parse PZero source into a well-typed AST. No code generation yet.

#	Task	Detail
1.1	Design and finalise syntax	Write the formal grammar (PEG). Resolve all ambiguity. Test with LLM prompting experiments. Produce a grammar.peg file.
1.2	Define the AST data structures	Rust enums + structs for every node type. This is the central data structure everything else operates on.
1.3	Implement lexer	Use logos crate for fast, declarative tokenisation. Define all tokens.
1.4	Implement parser	Recursive descent parser (hand-written for control and good error messages). Produce AST from token stream.
1.5	Implement pretty-printer	Canonical formatter: AST → source. Establishes the single valid formatting style.
1.6	Error recovery in parser	When parsing fails, attempt to identify what the user/LLM intended (edit-distance matching against known syntax patterns). Produce structured error JSON.
1.7	Test suite for parser	Extensive test corpus — valid programs, invalid programs with expected errors, edge cases.
Deliverables:

grammar.peg — formal grammar specification
pzero parse <file> CLI command that outputs AST as JSON
pzero fmt <file> CLI command that canonically formats source
Structured JSON error output for all parse failures
Parser test suite (aim for 200+ test cases)
Phase 2 — Type System & Semantic Analysis (Months 3–4)
Goal: Validate that parsed programs are type-correct. No code generation yet.

#	Task	Detail
2.1	Define type system	Core types: Int, Float, Bool, String, Option<T>, Result<T,E>, List<T>, Map<K,V>, user-defined structs, enums, function types. No null.
2.2	Implement type inference	Hindley-Milner style with local inference. Types flow forward through let bindings and function bodies. Function signatures are always explicitly typed (no global inference).
2.3	Implement type checker	Walk the AST, assign types to every node, report type errors with structured JSON diagnostics.
2.4	Implement effect checker	Verify that functions without effect do not call effectful functions. This is a simple taint analysis on the call graph.
2.5	Implement module resolution	Resolve module declarations and imports. Build a dependency graph. Detect cycles.
2.6	Implement name resolution	Resolve all identifiers to their declarations. Report "undefined variable" with suggestions (edit-distance).
2.7	Contract parsing	Parse require and ensure blocks. Attach them to AST function nodes. (Verification is Phase 4 — for now, just parse and store.)
2.8	Typed AST output	Produce a fully-typed, annotated AST for the codegen phase.
2.9	Dependency graph construction	Build a full dependency graph from the typed AST: function → functions it calls, types it uses, modules it imports. Store as a queryable graph data structure.
2.10	Context extraction engine	Given a symbol (function, type, test), walk the dependency graph and extract the transitive closure: all definitions the symbol depends on, recursively. Output as a self-contained PZero source slice.
2.11	pzero context CLI command	pzero context <file>:<line> or pzero context <symbol> outputs the complete context an LLM needs to understand and fix that symbol. Structured JSON and source-code output modes.
2.12	Knowledge Base Structure	Define schema for concepts and errors updates. Create initial content for core concepts (functions, types, effects).
2.13	pzero explain CLI command	Implement lookup engine for concepts and errors. Support fuzzy matching for concepts.
Deliverables:

pzero check <file> CLI command that type-checks and reports errors
pzero context <symbol> CLI command that outputs dependency context
pzero explain <query> CLI command for knowledge base lookup
Structured JSON diagnostics for all type/effect/name errors
Type checker test suite (aim for 300+ test cases)
Phase 3 — Code Generation & Test Runner (Months 5–7)
Goal: Generate compiling Rust code from type-checked PZero. Run tests.

#	Task	Detail
3.1	Design Rust codegen mapping	Document how every PZero construct maps to Rust. Define the restricted Rust subset.
3.2	Implement codegen: expressions	Literals, arithmetic, function calls, match, if/else → Rust equivalents.
3.3	Implement codegen: data types	Structs, enums → Rust structs + enums with #[derive(Clone, Debug, PartialEq)].
3.4	Implement codegen: functions	fn → fn. effect fn → fn (effects are checked at PZero level, Rust doesn't need to know).
3.5	Implement codegen: modules	PZero modules → Rust modules + mod.rs scaffolding.
3.6	Implement codegen: memory	All compound types wrapped in Arc<T>. Clone on assignment. No lifetime annotations.
3.7	Implement codegen: Option/Result	Direct 1:1 mapping. ? operator passes through.
3.8	Implement test runner	PZero test blocks → Rust #[test] functions. pzero test invokes cargo test on the generated project.
3.9	Implement contract runtime checks	require → assert! at function entry. ensure → assert! before return. (Runtime enforcement for now; static verification in Phase 4.)
3.10	End-to-end pipeline	pzero build <file> → parse → check → codegen → cargo build. pzero run <file> → build → execute.
3.11	Structured build errors	Capture cargo errors, map back to PZero source locations, output as structured JSON.
3.12	Error-triggered context generation	When a build or test fails, automatically attach the dependency context of the failing function/test to the error output. The LLM receives the error AND everything it needs to fix it in a single response.
3.13	pzero diagnose command	Combines pzero test + pzero context: runs tests, and for each failure, outputs the error plus the full dependency context of the failing test. Single command gives the LLM everything it needs.
Deliverables:

pzero build — compiles PZero to native binary via Rust
pzero run — build and execute
pzero test — build and run all test blocks
pzero diagnose — run tests + output error context for failures
Source-mapped error output (Rust errors → PZero line numbers)
Automatic dependency context attached to every error
End-to-end test suite: PZero source → expected output
Phase 3.5 — Frontend Target: JS Codegen & UI Runtime (Months 8–10)
Goal: Compile the same PZero source to JavaScript for browser execution. One language, two targets.

Design philosophy
PZero is ONE language with ONE semantic model that compiles to different runtimes. It can describe UI state and behaviour, and we compile it to an existing frontend platform.

The frontend target reuses 100% of the compiler frontend (lexer, parser, type checker, effect checker). Only the codegen module is new.

UI architecture: The Elm Architecture (TEA)
The UI model is functional and fits PZero's core philosophy:

Concept	PZero construct
Model (state)	A struct — immutable, explicit
Msg (events)	An enum — exhaustive, typed
update	A pure fn(Msg, Model) -> Model — deterministic, testable
view	A pure fn(Model) -> Html<Msg> — declarative, no side effects
effects	effect fn returning Cmd<Msg> — explicit, mockable
This architecture is ideal for LLMs because:

Every function is pure and testable in isolation
State transitions are explicit enum matches — LLMs excel at exhaustive pattern matching
No hidden DOM state, no lifecycle methods, no this
The entire UI is a function from data to description
Type-checked UI: Interface checking works out of the box
The Html<Msg> type system gives us compile-time UI correctness for free:

What the type system catches	How
Invalid element nesting	Html<Msg> constructors enforce which children are valid — e.g., tr only inside table, li only inside ul/ol
Mismatched event types	on_click(Msg::Submit) only compiles if Submit is a variant of the page's Msg enum
Missing state handling	match msg { ... } must be exhaustive — the compiler forces the LLM to handle every possible user action
Invalid attribute types	value(count) requires count: String — type mismatch caught at compile time
Broken component contracts	If a view function requires Model with field users: List<User>, passing a model without that field is a compile error
This means the LLM gets instant feedback on UI structure correctness before the browser is even opened. Most frontend bugs that slip through in JavaScript/React are caught here at compile time.

Three-tier UI testing strategy
PZero provides three levels of UI testing, each giving the LLM progressively more confidence:

Level	What it tests	How it runs	Speed
Tier 1: Type checking	UI structure, event wiring, state shape	pzero check — compile time	Instant
Tier 2: Unit tests	update logic, view output structure	pzero test — pure function tests, no browser	Fast (~ms)
Tier 3: Browser tests	Visual rendering, user interactions, full flows	pzero test --browser — Playwright against live app	Slower (~sec)
The LLM development loop becomes:

1. Write/modify PZero UI code
2. `pzero check`        → Type errors? Fix immediately (instant feedback)
3. `pzero test`         → Unit test failures? Fix logic (fast feedback)
4. `pzero test --browser` → Visual/interaction failures? Fix UI (rich feedback)
5. All green → commit
Each tier catches different classes of bugs, and the LLM gets structured JSON feedback at every stage.

Playwright integration: LLM-driven browser testing
Playwright is the ideal browser testing framework for LLM agents because:

Structured API: Playwright's selector and assertion model is explicit and type-safe — LLMs generate correct Playwright code reliably
Headless by default: No GUI needed — LLM agents run tests in CI-style headless mode
Built-in waiting: Auto-waits for elements, reducing flaky tests that confuse LLMs
Trace & screenshot capture: On failure, produces visual evidence the LLM can reason about (via structured error output)
PZero integrates Playwright as a first-class browser test primitive:

module counter;

// ... model, update, view as before ...

// Tier 2: Pure unit test — no browser, instant
test "increment updates model" {
    let m = update(Msg::Increment, Model { count: 0 });
    assert_eq(m.count, 1);
}

// Tier 2: View structure test — no browser, instant
test "view contains count display" {
    let html = view(Model { count: 42 });
    assert(contains_text(html, "42"));
}

// Tier 3: Browser test — Playwright, runs against live app
browser_test "clicking plus increments the counter" {
    navigate("/");
    let display = query("[data-testid='count']");
    assert_text(display, "0");
    click("[data-testid='increment']");
    assert_text(display, "1");
}

// Tier 3: Browser test — visual regression
browser_test "counter renders correctly" {
    navigate("/");
    screenshot("counter-initial");
    click("[data-testid='increment']");
    screenshot("counter-after-increment");
}
The browser_test block is a new PZero primitive that:

Compiles to Playwright test code (TypeScript) under the hood
Runs via pzero test --browser which starts pzero serve + Playwright in headless mode
Returns structured JSON results: pass/fail, selectors that failed, screenshots on failure, DOM snapshots
Uses PZero's familiar syntax — the LLM doesn't need to context-switch to a different language
Structured browser test failure output (JSON):

{
  "test": "clicking plus increments the counter",
  "status": "failed",
  "tier": "browser",
  "failure": {
    "assertion": "assert_text",
    "selector": "[data-testid='count']",
    "expected": "1",
    "actual": "0",
    "line": 28,
    "screenshot": "artifacts/counter-fail-L28.png",
    "dom_snapshot": "artifacts/counter-fail-L28.html",
    "suggestion": "The click event may not be wired to Msg::Increment. Check that the increment button uses on_click(Msg::Increment)."
  }
}
This gives the LLM everything it needs to diagnose and fix the issue in one iteration.

#	Task	Detail
3.5.1	JS codegen module	New backend: Typed AST → JavaScript source. Simpler than Rust codegen (no ownership, no lifetimes). Target ES2020+.
3.5.2	JS codegen: expressions & data types	Map PZero types to JS. Structs → object literals with Object.freeze. Enums → tagged unions { tag: "Variant", ...fields }. Pattern matching → switch on tag.
3.5.3	JS codegen: functions & modules	Pure functions → JS functions. Modules → ES modules with explicit exports.
3.5.4	JS codegen: Option/Result	Same tagged-union representation. None → { tag: "None" }. Some(x) → { tag: "Some", value: x }.
3.5.5	Minimal UI runtime	~500 lines of JS: virtual DOM diffing + patching, event delegation, TEA message loop. OR use existing library (morphdom, lit-html) as the DOM backend.
3.5.6	Html stdlib module	PZero module defining Html<Msg> type and element constructors (div, span, button, input, on_click, on_input, etc.).
3.5.7	Effect runtime for browser	Handlers for browser effects: HTTP (fetch), timers, local storage, URL routing. Mapped from PZero effect fn declarations.
3.5.8	Dev server	pzero serve — file watcher + recompile + hot reload in browser. Wraps a simple HTTP server.
3.5.9	Test runner in JS	PZero test blocks → JS test functions. pzero test --target js runs them in Node.js. Verifies that the same tests pass on both Rust and JS targets.
3.5.10	Cross-target test validation	Run the same PZero test suite against both Rust and JS codegen. If a test passes on one target but fails on the other, that's a codegen bug.
3.5.11	Playwright integration	browser_test blocks compile to Playwright TypeScript tests. pzero test --browser starts pzero serve in headless mode and runs Playwright against it.
3.5.12	Browser test primitives	Built-in functions: navigate(), click(), type_text(), query(), assert_text(), assert_visible(), assert_count(), screenshot(), wait_for(). All compile to Playwright API calls.
3.5.13	Structured browser test output	On failure: JSON with assertion details, expected vs actual, selector, screenshot path, DOM snapshot path, and fix suggestion.
3.5.14	Test artifact capture	Screenshots and DOM snapshots saved to artifacts/ on failure. Trace files for debugging complex interaction sequences.
3.5.15	data-testid generation	Compiler auto-generates data-testid attributes on elements that have event handlers or dynamic content, making them queryable in browser tests without manual annotation.
Deliverables:

pzero build --target js <file> — compiles PZero to a JS bundle
pzero serve — dev server with hot reload
pzero test --target js — runs unit tests in Node.js
pzero test --browser — runs browser tests via Playwright (headless)
Html<Msg> stdlib module for declarative UI
browser_test block as a first-class language primitive
Structured JSON output for all three test tiers (type check, unit, browser)
Auto-generated data-testid attributes for testability
Cross-target test validation (same unit tests, both backends)
5+ example UI applications with full test suites (counter, todo list, form, data table, API consumer)
What this enables
A developer (or LLM agent) writes one codebase:

module app;

// Shared types — used by both backend and frontend
struct User {
    name: String,
    email: String,
}

// Backend: API endpoint (compiles to native via Rust)
effect fn get_users() -> Result<List<User>, ApiError> {
    db::query("SELECT name, email FROM users")
}

// Frontend: UI (compiles to JS for browser)
fn view_user(user: User) -> Html<Msg> {
    div([class("user-card")], [
        h2([], [text(user.name)]),
        p([], [text(user.email)]),
    ])
}

// Same test language, works on both targets
test "user display" {
    let u = User { name: "Alice", email: "a@b.com" };
    let html = view_user(u);
    assert(contains_text(html, "Alice"));
}
Phase 4 — Contracts, Diagnostics & LSP (Months 11–13)
Goal: Advanced type-level contracts and development tooling.

#	Task	Detail
4.1	Refinement types	x: Int where x > 0 — checked at function boundaries at runtime, with static analysis for obvious cases.
4.2	Static contract checking	For simple cases (integer bounds, non-empty lists), verify contracts at compile time without SMT. Use abstract interpretation.
4.3	Enhanced error diagnostics	Add intent inference: when the parser/checker fails, analyse what the LLM was probably trying to do and suggest the correct syntax/type.
4.4	Property-based test generation	Given a function's type signature and contracts, auto-generate property tests. E.g., fn add(a: Int, b: Int) -> Int → test commutativity, identity.
4.5	LSP server	Language Server Protocol for editor integration (VS Code). Completion, diagnostics, go-to-definition, hover types.
4.6	Structured documentation	/// doc comments parsed as structured data (params, returns, errors, examples). pzero doc generates JSON/HTML docs.
Deliverables:

Refinement type checking (runtime + partial static)
Auto-generated property tests
LSP server (VS Code extension)
pzero doc command
Phase 5 — Standard Library & Effect Handlers (Months 14–16)
Goal: A usable standard library and the effect/DI system.

#	Task	Detail
5.1	Core stdlib	Collections (List, Map, Set), string operations, math, basic IO.
5.2	Effect handler system	Algebraic-style effects: define effect interfaces, provide handlers. This replaces traditional DI/mocking.
5.3	Mock generation via effects	effect fn read_file(...) → in tests, provide a mock handler that returns fake data. No special mock syntax needed.
5.4	Concurrency primitives	Structured concurrency via effects. effect fn spawn(task: fn() -> T) -> Future<T>.
5.5	Package manager	pzero.toml manifest file. Dependency resolution. Registry (can use a git-based registry initially).
Deliverables:

Usable standard library
Effect handler system with test mocking
Basic package management
Phase 6 — Ecosystem & Advanced Features (Month 17+)
Phase 6 — Ecosystem & Advanced Features (Month 17+)
Goal: Production readiness and advanced features.

#	Task	Detail
6.1	LLM integration toolkit	Ship a grammar.json + stdlib.json + prompt templates that LLMs can use to generate correct PZero.
6.2	Multi-agent primitives	Task queues, dependency declarations, parallel composition — as stdlib, not language primitives (keeps core simple).
6.3	WASM target	Add Cranelift/WASM as an alternative backend for portable/sandboxed execution.
6.4	Performance optimisation	Analyse generated Rust code. Replace Arc<T> with owned types where provably safe. Reduce cloning.
6.5	Self-hosting	Eventually, write the PZero compiler in PZero itself.
6.6	Mobile targets	Investigate compiling PZero UI to React Native / native mobile via additional codegen backends.
8. Architecture Overview
                    PZero Source (.ll)
                          │
                          ▼
                    ┌───────────┐
                    │   Lexer   │  (logos crate)
                    └─────┬─────┘
                          │ Token stream
                          ▼
                    ┌───────────┐
                    │  Parser   │  (recursive descent, hand-written)
                    └─────┬─────┘
                          │ Untyped AST
                          ▼
                    ┌───────────┐
                    │Type Checker│  (Hindley-Milner + effect checking)
                    └─────┬─────┘
                          │ Typed AST
                          ▼
                    ┌───────────┐
                    │ Contract  │  (require/ensure validation)
                    │ Checker   │
                    └─────┬─────┘
                          │ Verified AST
                          ▼
              ┌───────────┴───────────┐
              │                       │
              ▼                       ▼
    ┌──────────────────┐   ┌──────────────────┐
    │  Rust Codegen    │   │   JS Codegen     │
    │  (backend)       │   │   (frontend)     │
    └────────┬─────────┘   └────────┬─────────┘
             │                      │
             ▼                      ▼
    ┌──────────────────┐   ┌──────────────────┐
    │   Cargo          │   │  JS Bundle +     │
    │   (rustc + LLVM) │   │  UI Runtime      │
    └────────┬─────────┘   └────────┬─────────┘
             │                      │
             ▼                      ▼
      Native Binary          Browser App
All stages emit structured JSON diagnostics on error, with:

Error code
Source location (file, line, column, span)
What was expected vs. what was found
Intent inference ("did you mean...?")
Fix suggestion (replacement text)
9. Technology Stack
Component	Technology	Reason
Compiler implementation	Rust	Best for compiler front-ends: enums, pattern matching, safety
Lexer	logos crate	Fast, declarative, well-maintained
Parser	Hand-written recursive descent	Best error messages, full control over recovery
Codegen target (backend)	Rust (restricted subset)	Best semantic mapping, native output, cargo ecosystem
Codegen target (frontend)	JavaScript (ES2020+)	Universal browser support, ecosystem access
UI runtime	Minimal virtual DOM (~500 LOC)	TEA architecture, no framework dependency
Test runner (backend)	Wraps cargo test	Reuse Rust's test infrastructure
Test runner (frontend)	Wraps Node.js	Same tests, different runtime
Build orchestration	Wraps cargo build + JS bundler	Reuse existing build infrastructure
Dev server	Built-in HTTP + file watcher	pzero serve for frontend dev
LSP server	tower-lsp crate	Standard Rust LSP framework
CLI framework	clap crate	Standard Rust CLI framework
Error output	JSON (structured) + human-readable (coloured)	Dual-mode: machine-first, human-friendly
10. Success Criteria
v0.1 (end of Phase 3 — ~Month 7)
 PZero source can be written, parsed, type-checked, and compiled to a native binary
 Test blocks run and report pass/fail
 Errors are structured JSON with source locations
 An LLM (GPT-4/Claude) can generate valid PZero given 5 examples in-context
 10 non-trivial example programs compile and run correctly
v0.3 (end of Phase 3.5 — ~Month 10)
 Same PZero source compiles to both native binary (Rust) and JS bundle (browser)
 TEA-based UI applications run in the browser
 pzero serve provides dev server with hot reload
 Same unit test suite passes on both Rust and JS targets
 browser_test blocks run via Playwright in headless mode
 Structured JSON output from all three test tiers (type check → unit → browser)
 LLM can write a complete UI feature (model + update + view + tests + browser tests) from a spec
 5 example UI applications with full three-tier test suites
v0.5 (end of Phase 5 — ~Month 16)
 Contracts (require/ensure) work at runtime
 Effect system allows mocking in tests
 LSP provides completion and diagnostics in VS Code
 Standard library covers basic data structures and IO
 An LLM can generate PZero with >80% first-pass compilation success (measured on a benchmark suite)
v1.0 (Phase 6+)
 Package manager and registry
 LLM integration toolkit shipped
 Self-hosting feasibility demonstrated
 Community of early adopters
11. Open Questions to Resolve in Phase 1
These need answers before writing the first line of parser code:

File extension: .ll conflicts with LLVM IR. Alternatives: .p0, .lang, .ai. Recommendation: .p0
Entry point: fn main() like Rust/C? Or implicit top-level execution? Recommendation: explicit fn main().
String interpolation: "hello {name}" (Rust-style) or template literals? Need to decide syntax.
Generic syntax: List<T> (angle brackets) or List[T] (square brackets, avoids HTML/XML confusion for LLMs)? Recommendation: angle brackets — more training data.
Semicolons: Mandatory (Rust/C) or optional with newline-as-terminator (Go)? Recommendation: mandatory — removes ambiguity.
Whitespace: Completely insignificant (C-style) or meaningful indentation (Python)? Recommendation: insignificant — per principle #6.
Standard library surface area for v1: What's the minimum usable stdlib?
12. Immediate Next Steps
Set up the Rust project — cargo init with workspace structure for compiler crates
Write the PEG grammar — formal syntax definition, iterate with examples
Test the syntax with LLMs — prompt GPT-4/Claude with example PZero code and measure generation accuracy
Implement lexer — first compilable component
Implement parser — the core of Phase 1
