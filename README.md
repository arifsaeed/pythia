CRITICAL: Declaration-side generic parameters don't exist yet
The generics plan correctly identifies this. The AST structs FnDecl, StructDecl, EnumDecl have no generics field. But now I can see exactly where it needs to go:

lib.rs:56 — FnDecl needs a generics field
lib.rs:84 — StructDecl needs a generics field
lib.rs:92 — EnumDecl needs a generics field
And in the parser, items.rs:51 has the telling comment:

So the plan is directionally correct here. My upgrade recommendation stands: use a GenericParam struct instead of Vec<String>:

2. CRITICAL: HIR has no generics support at all
Looking at the actual HIR in lib.rs:

HirFunction, HirStruct, HirEnum — no generics field
HirType enum — has a commented-out // Generic(String), // TODO: For generics support at lib.rs:157
HirType::Path(Vec<String>) — has no type arguments. When lowering List<Int>, the <Int> argument is lost because there's no place to store it
This is a bigger gap than the plan acknowledges. The plan says to add HirType::GenericParam(String) but also needs:

Currently HirType::Path(Vec<String>) can't represent List<Int> vs List<String> — they'd both just be Path(["List"]).

3. CRITICAL: lower_type drops generic arguments
In lowering.rs:157-170, the current lower_type:

p.args is completely ignored. The AST already carries PathType.args: Vec<Type> with the generic arguments, but lower_type discards them. The generics plan says "update lower_type" but the scope of that update is larger than implied — it needs to:

Lower p.args recursively
Check if a single-segment path matches an active generic parameter → HirType::GenericParam
Otherwise, produce an applied type with lowered args
4. HIGH: Type checker has no unification infrastructure
The type checker in check.rs uses direct equality (==) for type comparison. There's no:

Type variables / inference variables
Substitution map (HashMap<TypeVar, Type>)
Unification algorithm
Occurs check
The Call case at check.rs:86-97 directly compares param types to arg types:

For generics, this needs a fundamentally different approach: when calling id(5) where id: fn<T>(T) -> T, the checker must:

Create fresh type variables for each generic param (?T)
Unify ?T with Int (from the argument 5)
Substitute ?T → Int in the return type
Return Int
This is the hardest part of the entire plan and it's still described as one line.

5. HIGH: Missing modules will block the build
The subagent found these declared-but-missing modules:

creates/parser/src/parser.rs and creates/parser/src/stmt.rs (declared in lib.rs)
creates/hir/src/resolver.rs, scope.rs, name_resolution.rs (declared in lib.rs)
lib.rs and types.rs (the check.rs references crate::types::Type)
The entire lexer, diagnostics, cli, formatter, knowledge crates (in Cargo.toml but not in workspace)
Directory typo: actual code is in creates but Cargo.toml references crates/
These are not generics issues, but generics can't be implemented until the project compiles.

6. MEDIUM: </> ambiguity in expressions
The expression parser in expr.rs:264 treats < and > as comparison operators:

The type parser in types.rs:72 uses Token::Lt/Token::Gt for generic args. This works because types and expressions are parsed in different contexts — the parser knows whether it's parsing a type or expression. This is fine for now.

But there's a future issue: turbofish syntax (id::<Int>(5)). If PZero ever needs explicit type arguments at call sites (in expression position), the < becomes ambiguous. The generics plan should note this as a known limitation of the "infer everything" approach and document when turbofish would be needed.

Revised Verdict
Aspect	Previous Score	Updated Score	Why
Type application (usage sites)	Missing	Done	AST + parser already handle List<Int>
Declaration-site generics	Missing	Correctly planned	Plan targets the right places
Data representation	Wrong (Vec<String>)	Needs upgrade	Use GenericParam struct
HIR type args	Not reviewed	Missing from plan	HirType::Path can't carry args
Lowering	Vague	Undersized	lower_type ignores args entirely
Type checker unification	Too vague	Needs full design	No infrastructure exists
Error diagnostics	Missing	Still missing	
</> ambiguity	Flagged	Not an issue (for now)	Context-sensitive parsing works
What Should the Plan Focus On
Fix the build first — directory typo, missing modules, missing crates. Nothing works until the project compiles.
Expand HirType — add Applied(name, Vec<HirType>) and GenericParam(String). This is the structural foundation everything else depends on.
Design the unification algorithm in detail — substitution map, fresh type variables, occurs check, error reporting. This deserves its own section in the plan, not one sentence.
Update lower_type — currently drops all generic arguments. The fix is straightforward but needs to be explicit about the generic parameter scope/environment.
Error diagnostics — wrong arity, conflicting inference, unused type params. Core to PZero's LLM-friendly identity.
The plan is pointed in the right direction, but it underestimates the work in layers 2–4 (HIR, lowering, type checker) because the current code there is more skeletal than the plan assumes.
