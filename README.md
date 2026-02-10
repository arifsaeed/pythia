Phase 2 Review: What's Strong, What's Missing, and What Could Be Different
What You've Got Right
Your Phase 2 plan is well-structured around the four pillars (types, effects, context, semantics), and the decision to introduce a HIR lowering pass (Week 9) is excellent — it's more mature than the original development plan which talks about walking the AST directly. The weekly breakdown is realistic for a focused effort.

What's Missing
1. Ordering problem: Name resolution must come before type checking, not after.
The master plan lists 2.6 (name resolution) after 2.3 (type checker). Your Phase 2 plan fixes this for Week 9 but the dependency isn't stated explicitly. Name resolution → HIR lowering → type inference → effect checking is the canonical pipeline. Make this ordering a hard requirement in the plan.

2. No mention of generic type instantiation / monomorphization planning.
You mention "Monomorphized at codegen time" in passing but don't allocate any work to actually tracking generic instantiations during type checking. The type checker needs to handle List<Int> vs List<String> as distinct concrete types during unification. This is significant work — at minimum, you need type application and substitution logic in Week 10.

3. No closures or lambda types.
The development plan shows |> pipe and functional composition as core features. Your type system definition lists Type::Fun(params, ret) but doesn't address closure captures, higher-order function typing, or how lambdas interact with the effect system. For example: if you pass a closure to map, does the closure inherit the caller's effect context? This needs an answer in Phase 2, even if limited.

4. No Tuple type handling in the inference engine.
You list Tuple under composites but it doesn't appear in the roadmap tasks. Tuple types need structural typing in unification (positional, not named), which is different from structs.

5. Pattern matching exhaustiveness checking is absent.
This is a Phase 2 concern, not Phase 3. The type checker should verify that match expressions on enums are exhaustive. This is critical for LLM correctness — the whole pitch is "compiler catches mistakes before runtime." Without exhaustiveness checks, you're shipping a half-baked type system.

6. No span/source-location tracking on the HIR.
You'll need source spans carried through from AST → HIR → typed HIR for error messages and the context engine. This is easy to forget and painful to retrofit. Mention it explicitly in Week 9.

7. The context engine (Week 12) doesn't address test block dependencies.
pzero context for a test block needs to resolve the functions called inside the test, not just the test's declaration. Your dependency graph stores Function -> [Calls] but test blocks aren't functions in the grammar — they're a separate construct. Ensure the graph handles test nodes as first-class vertices.

8. No import / use statement semantics defined.
Module resolution (Week 11) says "load multiple files, build the project graph" but doesn't specify:

Syntax: import app::utils::foo vs import app::utils then utils::foo?
Selective imports: import app::utils { foo, bar }?
Re-exports?
This needs to be locked down before implementation.

9. No incremental/caching story.
Not critical for v1, but worth noting: re-running pzero check on a large project will re-parse and re-check everything. If you structure the HIR with module-level granularity now, incremental checking becomes possible later.

What You Could Do Differently
1. Week ordering: swap effects and modules.
Your plan does effects (Week 11) before module resolution (also Week 11). In practice, you need multi-file resolution before you can check cross-module effect violations. Consider:

Week 11a: Module resolver + multi-file loading
Week 11b: Effect checker (now that you can resolve cross-module calls)
2. Consider a simpler inference model than full Hindley-Milner.
You mandate explicit function signatures — that's good. But full HM with unification for just local inference is arguably over-engineered. A simpler bidirectional type checking approach (check mode + infer mode) would:

Be easier to implement in 1 week
Produce better error messages (HM unification errors are notoriously confusing)
Still handle all the local inference cases you need (let x = a + b)
Be what most modern languages actually use (Rust, Swift, Kotlin all use bidirectional, not HM)
HM is elegant but the error quality problem is real and directly conflicts with your "AI-optimised structured error feedback" principle.

3. Define the Option/Result types in PZero, not as compiler magic.
If these are defined as regular enum types in a prelude module, the type checker doesn't need special cases — they're just enums with generics. The ? operator is the only special-case syntax. This keeps the core simpler and also tests your generic enum machinery end-to-end.

4. The context engine --max-tokens feature needs a token estimation model.
How will you count tokens? A rough chars / 4 heuristic? A proper BPE tokenizer? This is a design decision that should be stated. Recommendation: ship with chars / 4 and let users override with --max-chars as a simpler alternative.

Pros and Cons of Your Proposed Approach
Detail
Pro	HIR lowering is the right architecture — separates parsing concerns from semantic concerns cleanly
Pro	Explicit function signatures eliminate the hardest part of HM inference (global inference), making the type system tractable
Pro	Boolean effect system is pragmatic — typed effects would add months of complexity for marginal LLM benefit
Pro	Putting the context engine in Phase 2 is smart — it forces you to build the dependency graph properly, which Phase 3 codegen depends on
Pro	"No null, no implicit casting" is clean and removes entire error classes
Con	4 weeks is aggressive for type checker + effect checker + module resolver + context engine. Most compilers allocate 3-6 months for the type system alone. Risk of shipping a shallow type checker that breaks on real programs
Con	No exhaustiveness checking means enum/match — the language's central feature — is only half-checked
Con	String interpolation ("Hello {name}") with automatic variable resolution is ambiguous: what if name is a field access? A method call? Define the grammar precisely or it'll haunt you
Con	pub visibility adds scope complexity to name resolution, but it's the right call — just budget the time
Con	No mention of how errors compose across phases. If the type checker hits 5 errors, does it keep going? Does it feed partial results to the effect checker? Error recovery strategy in the semantic pipeline is undefined
Answers to Your Open Questions
Q1 (Collections): Agree with stdlib + syntax sugar. But define the desugaring rule in Phase 2 so the type checker knows [1, 2, 3] has type List<Int>.

Q2 (Effect Granularity): Boolean is correct. Typed effects are a v3+ feature if ever.

Q3 (Visibility): Yes to pub, but consider pub by default at module level, private by default inside modules (the Go model). Simpler for LLMs — fewer annotations to get right.

Q4 (String Interpolation): Prefer f"Hello {name}" with the f prefix. The automatic version creates parsing ambiguity with literal braces and expression complexity inside {}. The f prefix is a clear signal to both parser and LLM.
