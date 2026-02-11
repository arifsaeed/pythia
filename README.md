The Core Insight
Traditional IDEs are human-first text editors with language tooling bolted on. VS Code, IntelliJ — they're all "file tree + editor + terminal" because humans navigate files, type code, and run commands.

PZero's IDE should invert this. The LLM is the primary code author. The human is the architect, reviewer, and intent provider. The IDE is a collaboration surface where the human describes what they want and reviews what the LLM produces — with the compiler providing instant, structured feedback to both.

This means:

No terminal. The LLM doesn't need one. The human doesn't either. Everything is API-driven with structured results.
No file tree as the primary navigation. Navigate by module graph, function, test, or intent — not by filesystem paths.
The editor is one panel, not THE panel. Test results, preview, diagnostics, dependency context, and the LLM conversation are all equal citizens.
Architecture

┌─────────────────────────────────────────────────────────────┐│                    Browser (User's machine)                  ││                                                             ││  ┌─────────────────────────────────────────────────────┐   ││  │              PZero Studio (Web App)                   │   ││  │  ┌──────────┬──────────────┬───────────────────┐    │   ││  │  │  Intent   │    Code      │   Feedback         │    │   ││  │  │  Panel    │    Editor    │   Panel            │    │   ││  │  │          │              │  ┌──────────────┐  │    │   ││  │  │  Spec    │  CodeMirror  │  │ Diagnostics  │  │    │   ││  │  │  Chat    │  6 + PZero   │  │ Test Results │  │    │   ││  │  │  History │  grammar     │  │ Type Info    │  │    │   ││  │  │          │              │  └──────────────┘  │    │   ││  │  │          │              │  ┌──────────────┐  │    │   ││  │  │          │              │  │ App Preview  │  │    │   ││  │  │          │              │  │ (iframe)     │  │    │   ││  │  │          │              │  └──────────────┘  │    │   ││  │  └──────────┴──────────────┴───────────────────┘    │   ││  └─────────────────────────────────────────────────────┘   ││                          │ WebSocket                        │└──────────────────────────┼──────────────────────────────────┘                           │┌──────────────────────────┼──────────────────────────────────┐│                    PZero Server                              ││                          │                                   ││  ┌───────────────────────┴────────────────────────────┐     ││  │              Session Manager                        │     ││  │  (per-user workspace, WebSocket routing)            │     ││  └──────┬──────────┬──────────┬──────────┬────────────┘     ││         │          │          │          │                    ││  ┌──────┴───┐ ┌────┴────┐ ┌──┴───┐ ┌───┴──────┐           ││  │ Compiler │ │   LLM   │ │ Test │ │ Sandbox  │           ││  │ Service  │ │ Orchestr│ │Runner│ │ Runtime  │           ││  │          │ │         │ │      │ │          │           ││  │ parse    │ │ routes  │ │ unit │ │ Rust bin │           ││  │ check    │ │ context │ │ tier │ │ JS serve │           ││  │ codegen  │ │ to LLM  │ │ brow.│ │ (contain)│           ││  │ format   │ │ applies │ │      │ │          │           ││  │ context  │ │ edits   │ │      │ │          │           ││  │ diagnose │ │         │ │      │ │          │           ││  └──────────┘ └─────────┘ └──────┘ └──────────┘           │└──────────────────────────────────────────────────────────────┘
Technology Choices
Component	Choice	Rationale
Editor component	CodeMirror 6	Lightweight, extensible, no VS Code baggage. Built for embedding. We write a PZero grammar extension for syntax highlighting, bracket matching, and inline decorations.
Web framework	SolidJS or Svelte	Reactive, fast, small bundle. React is fine too but heavier. The IDE needs to feel instant — sub-frame UI updates when the compiler returns results.
Server	Rust (Axum)	Same language as the compiler. No FFI boundary. The compiler is a library, not a subprocess. Axum has great WebSocket support.
Real-time comms	WebSocket	Bidirectional: server pushes compilation results, test outcomes, LLM responses. Client pushes code changes, intent messages.
App preview	iframe with sandboxed JS	PZero's JS codegen output runs in an isolated iframe. Same security model as CodePen/StackBlitz.
Sandboxing	Container per session (Firecracker/gVisor)	User's Rust codegen compiles and runs inside a sandboxed container. The JS codegen runs in the iframe (browser sandbox).
LLM integration	Server-side orchestration	The LLM never sees a terminal. The server translates between LLM tool calls and the Compiler Service API.
The Compiler Service API (The Heart of the IDE)
This is what makes the IDE unique. Instead of the LLM running shell commands, it calls structured APIs. Instead of the human reading terminal output, they see rendered results.


// Compilation & CheckingPOST /api/check                    → Full type check, returns structured diagnosticsPOST /api/check-function/{name}    → Check a single function + its dependenciesPOST /api/format                   → Canonical format, returns formatted sourcePOST /api/build                    → Full codegen (Rust or JS target)POST /api/build-incremental        → Only recompile changed functions// TestingPOST /api/test                     → Run all test blocksPOST /api/test-function/{name}     → Run tests associated with a specific functionPOST /api/test-browser             → Run browser_test blocks via PlaywrightPOST /api/diagnose                 → Run tests + return failure context (the killer feature)// Context (for LLM and human)POST /api/context/{symbol}         → Dependency context for a symbolPOST /api/context/{symbol}/dependents → Reverse dependenciesPOST /api/explain/{query}          → Knowledge base lookup// ExecutionPOST /api/run                      → Build + execute (backend target)POST /api/serve                    → Build JS + start dev server (frontend target)// File Operations (for LLM)POST /api/files/read               → Read file contentsPOST /api/files/write              → Write/create filePOST /api/files/edit               → Apply structured edits (not raw text replacement)POST /api/files/list               → List project files
Every response is structured JSON. The IDE renders it. The LLM consumes it. No parsing terminal output. No guessing.

IDE Layout & Panels
Panel 1: Intent Panel (Left)
This is where the human communicates what they want, not how to code it.


┌─ Intent Panel ──────────────────────┐│                                      ││  [Spec Mode] [Chat Mode] [History]  ││                                      ││  ┌────────────────────────────────┐ ││  │ "Add a user registration flow  │ ││  │  with email validation. The    │ ││  │  form should show inline       │ ││  │  errors. Store users in db."   │ ││  └────────────────────────────────┘ ││                                      ││  LLM Plan:                          ││  ✅ 1. Create User struct           ││  ✅ 2. Create ValidationError enum  ││  🔄 3. Write validate_email fn     ││  ⬚ 4. Write registration view      ││  ⬚ 5. Write browser tests          ││                                      ││  [Approve All] [Step Through]       ││                                      │└──────────────────────────────────────┘
Key UX decisions:

The human writes intent ("add user registration"), not code
The LLM produces a plan with discrete steps the human can approve/reject/modify
Each step shows a diff preview before applying
The human can say "step through" to watch the LLM work function by function, or "approve all" to let it run
History: Every intent → plan → code change is logged. The human can revert to any point.
Panel 2: Code Editor (Center)
CodeMirror 6 with PZero-specific extensions:


┌─ Code Editor ───────────────────────────────────────┐│ [Module: app/registration.p0]  [Graph] [Split]      ││                                                       ││  1  module registration;                              ││  2                                                    ││  3  struct User {                     📋 3 fields    ││  4      name: String,                                 ││  5      email: String,                                ││  6      age: Int,                                     ││  7  }                                                 ││  8                                                    ││  9  ✅ fn validate_email ──── with io ── requires ── ││ 10  fn validate_email(email: String) -> Result<...>  ││ 11      with io                                       ││ 12      require email.len() > 0;                      ││ 13      ensure result.is_ok() || result.is_err();     ││ 14  {                                                 ││ 15      // ...                                        ││ 16  }                                                 ││ 17                                                    ││ 18  ── Tests (2/2 passing) ──────────────────────── ││ 19  test "valid email passes" {          ✅ 2ms      ││ 20      assert(validate_email("a@b.com").is_ok());   ││ 21  }                                                 ││ 22  test "empty email fails" {           ✅ 1ms      ││ 23      assert(validate_email("").is_err());          ││ 24  }                                                 ││ 25  ─────────────────────────────────────────────── │└───────────────────────────────────────────────────────┘
Key features:

Function headers are rich — inline badges for effects (with io), contracts (require, ensure), and test status (✅/❌)
Tests are visually grouped with their function — not buried in a separate file. The test blocks immediately following a function are rendered in a collapsible "test band"
No file tree — navigation is by module graph (visual) or symbol search (⌘P equivalent). The file system is an implementation detail.
Inline diff review — when the LLM proposes a change, the diff appears inline in the editor (accept/reject per hunk)
Contract visualization — require and ensure render as visual "guards" with green/red status based on last test run
Panel 3: Feedback Panel (Right)
This panel shows everything the compiler knows, structured and rendered:


┌─ Feedback Panel ────────────────────┐│ [Diagnostics] [Tests] [Types] [Deps]││                                      ││ ── Diagnostics (0 errors) ────────  ││ ✅ All clear                        ││                                      ││ ── Tests ─────────────────────────  ││ Tier 1 (Type Check): ✅ Pass       ││ Tier 2 (Unit):  12/12 ✅           ││ Tier 3 (Browser): 3/3 ✅           ││                                      ││ ── Current Function ──────────────  ││ validate_email                       ││ Type: (String) -> Result<Bool, Err> ││ Effects: io                          ││ Contracts: 1 require, 1 ensure      ││ Tests: 2 passing                     ││ Dependents: register_user,          ││             update_profile           ││                                      ││ [View Dependency Graph]              ││ [View Context (LLM)]                │└──────────────────────────────────────┘
Tabs:

Diagnostics — structured errors rendered with source spans, suggestions, "did you mean?"
Tests — three-tier results. Click a failure to see the full pzero diagnose output (error + context + suggestion)
Types — hover-style type info for the current cursor position, but always visible
Deps — dependency graph of the current function/module, interactive
Panel 4: App Preview (Bottom-Right or Separate Tab)
For frontend code, an embedded browser:


┌─ App Preview ────────────────────────┐│ [localhost:3000/registration]  [⟳]   ││ ┌──────────────────────────────────┐ ││ │                                  │ ││ │   Register                       │ ││ │   ┌──────────────────────────┐  │ ││ │   │ name@example.com         │  │ ││ │   └──────────────────────────┘  │ ││ │   ⚠️ Invalid email format       │ ││ │                                  │ ││ │   [Submit]                       │ ││ │                                  │ ││ └──────────────────────────────────┘ ││                                       ││ ── Browser Test Overlay ───────────  ││ ✅ "form shows validation error"     ││ ❌ "submit sends to backend"         ││    Expected: POST /api/register      ││    Actual: No request sent           ││    [Show test code] [Fix with LLM]   │└───────────────────────────────────────┘
Key features:

Live reload — code changes → instant recompile → iframe refreshes
Browser test overlays — browser_test results render as overlays on the preview, highlighting the elements that failed
"Fix with LLM" button — on any test failure, one click sends the pzero diagnose context to the LLM and asks it to fix the issue
Element inspector — click an element in the preview to jump to the view function that generated it
data-testid visibility — toggle to show the auto-generated test IDs on elements
The LLM Integration Model (The Key Innovation)
The LLM doesn't interact with a terminal. It uses a structured tool protocol:


// What the LLM sees (tool definitions):{  "tools": [    {      "name": "check_project",      "description": "Type-check the entire project. Returns structured diagnostics.",      "returns": "{ diagnostics: Diagnostic[], passed: bool }"    },    {      "name": "check_function",      "params": { "function_name": "string" },      "description": "Check a single function and its dependencies.",      "returns": "{ diagnostics: Diagnostic[], type_info: TypeInfo }"    },    {      "name": "run_tests",      "params": { "function_name?": "string", "tier?": "unit|browser|all" },      "returns": "{ passed: int, failed: int, results: TestResult[] }"    },    {      "name": "get_context",      "params": { "symbol": "string", "depth?": "int" },      "description": "Get dependency context for a symbol.",      "returns": "{ source_slice: string, dependencies: Dep[], tokens: int }"    },    {      "name": "edit_file",      "params": { "file": "string", "edits": "Edit[]" },      "description": "Apply structured edits to a file."    },    {      "name": "diagnose",      "description": "Run all tests and get full failure context.",      "returns": "{ test_results: ..., failures: FailureWithContext[] }"    }  ]}
The LLM development loop becomes:

Human provides intent → "Add email validation"
LLM calls get_context("User") → understands the current code
LLM calls edit_file → writes validate_email function + tests
IDE auto-runs check_function("validate_email") → instant type feedback
If errors: LLM sees structured diagnostics, fixes immediately
IDE auto-runs run_tests(function_name: "validate_email") → test results
If failures: LLM calls diagnose → gets failure context + suggestion
LLM fixes → tests pass → human reviews diff and approves
The human sees this as: "I asked for email validation. The LLM wrote it, fixed two type errors, and all tests pass. Here's the diff for me to review." Total time: seconds.

Development Plan
Phase IDE-1: Core Shell (Weeks 1-4)
#	Task	Detail
1.1	Web app skeleton	SolidJS/Svelte app with panel layout (Intent, Editor, Feedback, Preview). CSS grid-based responsive layout.
1.2	CodeMirror 6 integration	Embed CM6 editor. Write PZero syntax grammar (TreeSitter or Lezer grammar for CM6). Syntax highlighting, bracket matching, auto-indent.
1.3	WebSocket connection	Bidirectional WS between browser and server. Message protocol: { type: "check", payload: {...} } → { type: "check_result", payload: {...} }. Reconnection handling.
1.4	Rust server skeleton	Axum server with WS endpoint. Session management (one workspace per connection). Embed the PZero compiler as a library (not subprocess).
1.5	File system abstraction	Virtual file system per session. In-memory for now, persisted to disk/DB later. The frontend sends file mutations, the server applies them and triggers compilation.
1.6	Panel system	Resizable, collapsible panels. Tab system within panels. Layout persistence (user preferences).
Deliverables: A web app that opens, shows a code editor, and connects to a server via WebSocket.

Phase IDE-2: Compiler Integration (Weeks 5-8)
#	Task	Detail
2.1	Compiler-as-library API	Wrap the PZero compiler in a service struct: CompilerService::check(), ::check_function(), ::format(), ::build(). Returns structured results, not strings.
2.2	Live type checking	On every keystroke (debounced ~300ms), send source to server → compile → return diagnostics. Render inline in editor (red underlines, hover messages).
2.3	Per-function checking	Click a function → check just that function + dependencies. Show diagnostics scoped to that function. Much faster than full project check.
2.4	Diagnostic rendering	Render structured diagnostics in Feedback panel: error code, message, span highlight in editor, suggestion with "Apply fix" button.
2.5	Auto-format on save	pzero fmt runs server-side, result pushed to client. No config — canonical formatting only.
2.6	Statement index display	Show statement indices (SI) alongside line numbers in the editor gutter. Per the language design, these are stable references for the LLM.
Deliverables: Live type checking, per-function checking, inline diagnostics, auto-format.

Phase IDE-3: Test Integration (Weeks 9-12)
#	Task	Detail
3.1	Test discovery	Parse the project, find all test and browser_test blocks. Associate each test with the function it tests (heuristic: tests immediately after a function, or tests that call the function).
3.2	Inline test status	Show ✅/❌/⬚ badges next to functions in the editor. Collapsible "test band" below each function showing its associated tests + results.
3.3	Three-tier test panel	Feedback panel tab showing: Tier 1 (type check pass/fail), Tier 2 (unit test results), Tier 3 (browser test results). Each tier collapsible with individual test results.
3.4	Test runner service	Server-side test execution. For Rust target: cargo test in sandbox. For JS target: Node.js in sandbox. Returns structured TestResult[].
3.5	Diagnose integration	"Diagnose" button on any test failure → calls pzero diagnose → renders the full failure context (error + dependency context + suggestion) in a dedicated panel.
3.6	"Fix with LLM" button	On any test failure, one click sends the diagnose output to the LLM agent. The LLM proposes a fix. The human reviews and approves.
3.7	Watch mode	Optional: re-run affected tests on every code change. Show live test status in the gutter.
Deliverables: Test discovery, inline test badges, three-tier test panel, diagnose integration.

Phase IDE-4: LLM Integration (Weeks 13-18)
#	Task	Detail
4.1	Intent Panel	Left panel for human-LLM communication. Text input for natural language specs. History of past intents.
4.2	LLM orchestration server	Server-side component that receives human intent, constructs the LLM prompt with project context, and manages the tool-use loop (edit → check → test → fix).
4.3	Structured tool protocol	Define the tool interface the LLM uses (check, edit, test, diagnose, context, explain). No terminal. All structured JSON.
4.4	Plan display	When the LLM produces a multi-step plan, render it as a checklist in the Intent Panel. Human can approve/reject/reorder steps.
4.5	Inline diff review	When the LLM edits code, show an inline diff in the editor (green/red highlights). Human accepts or rejects per hunk.
4.6	Auto-context injection	Before sending code to the LLM, automatically call pzero context for the relevant symbols. The LLM receives exactly the code it needs, nothing more.
4.7	Streaming responses	LLM responses stream to the IDE. The human sees the plan forming in real-time, can interrupt or redirect mid-stream.
4.8	Explain integration	The human (or LLM) can trigger pzero explain for any concept, error code, or syntax. Results render in a knowledge panel.
Deliverables: Full LLM loop — intent → plan → code → check → test → review → approve.

Phase IDE-5: App Preview & Frontend (Weeks 19-24)
#	Task	Detail
5.1	JS build pipeline	Server compiles PZero to JS bundle, serves via embedded HTTP server.
5.2	Embedded preview iframe	Sandboxed iframe in the Preview panel. Loads the compiled JS app. Isolated from the IDE (no cookie/storage leakage).
5.3	Hot reload	On code change → incremental JS build → push update to iframe via WebSocket → UI refreshes without full page reload. Target: <500ms from edit to visible change.
5.4	Browser test overlay	When browser tests run, overlay results on the preview. Highlight elements that failed assertions. Click an overlay to see the test code and failure details.
5.5	Element → code linking	Click any element in the preview → jump to the view function that generated it. Powered by source maps from the JS codegen.
5.6	Playwright server-side	Browser tests run headless on the server via Playwright. Results + screenshots + DOM snapshots streamed to the IDE.
5.7	data-testid toggle	Toggle in the preview to show/hide auto-generated data-testid attributes on elements. Helps the human verify test coverage.
5.8	Responsive preview	Resize the preview panel to test different viewport sizes. Mobile/tablet/desktop presets.
Deliverables: Live app preview, hot reload, browser test overlays, element→code linking.

Phase IDE-6: Advanced Features (Weeks 25-32)
#	Task	Detail
6.1	Module graph visualization	Interactive graph showing module dependencies (uses D3.js or similar). Click a module to navigate. Highlight cycles.
6.2	Function dependency graph	For any function, show its call graph visually. Powered by pzero context --dependents.
6.3	Effect flow visualization	Color-code the call graph by effect. "Show me all paths that touch IO." Helps architects review side-effect boundaries.
6.4	Contract dashboard	List all require/ensure contracts in the project. Show pass/fail status from last test run. Identify functions with no contracts.
6.5	Multi-cursor collaboration	Multiple humans (or human + LLM) editing simultaneously, Google Docs-style. CRDT-based conflict resolution.
6.6	Version history	Every LLM edit is a version. Every human approval is a checkpoint. Git-like but simpler — branching is just "try a different approach."
6.7	Project templates	"New project" wizard: "Full-stack app", "CLI tool", "Library", "API server". Pre-populates module structure with idiomatic PZero.
6.8	Keyboard shortcuts & command palette	⌘K for command palette. ⌘⇧T for run tests. ⌘⇧B for build. Standard but customizable.
Phase IDE-7: Deployment & Hosting (Weeks 33-40)
#	Task	Detail
7.1	User authentication	OAuth (GitHub, Google, email). User accounts with project storage.
7.2	Multi-tenant isolation	Each user gets a sandboxed workspace (container or VM). Compilation and execution are isolated. Resource limits (CPU, memory, time).
7.3	Project persistence	Save projects to server-side storage. Export to git repository. Import from git.
7.4	CDN & edge deployment	Static IDE assets served from CDN. WebSocket connections to nearest regional server.
7.5	Usage metering	Track compilation time, LLM token usage, test execution time. Free tier + paid tiers.
7.6	One-click app deployment	"Deploy" button publishes the user's PZero app to a public URL. Backend as a serverless function, frontend as static site.
7.7	Share & embed	Share a project via URL (view-only or editable). Embed the IDE in docs/tutorials via iframe.
What Makes This Different From Every Other Web IDE
Feature	VS Code Web / Codespaces / StackBlitz	PZero Studio
Primary user	Human developer	LLM agent (human reviews)
Navigation	File tree	Module graph + symbol search
Terminal	Central to workflow	Does not exist
Compiler feedback	Squiggly lines + Problems tab	Structured JSON rendered into rich UI, auto-fed to LLM
Tests	Separate test explorer	Inline with functions, three tiers, one-click LLM fix
App preview	Separate browser tab	Embedded panel with test overlays and element→code linking
LLM integration	Copilot sidebar chat	First-class Intent panel with plan approval, inline diff review, auto-context
Dependency context	Human manually reads files	pzero context auto-scopes what the LLM needs
Error → fix loop	Human reads error, types fix	pzero diagnose → LLM auto-fix → human reviews
The fundamental bet: by removing the terminal, removing the file tree, and making the compiler's structured output the primary interface, you create an environment where the LLM can operate at near-100% autonomy on routine code changes, and the human's job becomes architecture and review.
