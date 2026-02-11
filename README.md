Context Engine Implementation Plan (Task 2.10 - 2.12)
Goal
Implement the Context Engine to provide LLM agents with self-contained, dependency-aware context windows. This enables "Correct-by-Construction" agentic workflows.

Current State Review
Parser & HIR: Robust, supports Generics, Effects, and Tests.
Type Checker: Resolves names, types, and verifies effects.
Dependency Graph: Can traverse forward (Call graph) and reverse (Impact analysis).
Implementation Plan
Phase 1: Core Engine (crates/context)
[NEW] crates/context
A new crate that orchestrates the compiler frontend to extract context.

ContextEngine Struct
pub struct ContextEngine {
    // Shared compiler state
    hir: HirProgram,
    deps: DependencyGraph,
    source_map: SourceMap, 
}
ContextView Struct (The Output)
pub struct ContextView {
    pub target: ContextItem,
    pub dependencies: Vec<ContextItem>, // Depth 1..N
    pub dependents: Vec<ContextItem>,   // Reverse dependencies
}
pub struct ContextItem {
    pub signature: String,
    pub contracts: Vec<String>,
    pub body: Option<String>, // Omitted if depth > 1
    pub docs: String,
}
Phase 2: CLI Integration (pzero context)
[MODIFY] crates/driver (CLI)
Add context subcommand.
Usage: pzero context <symbol_name> [--depth N] [--format json|source]
Logic
Resolve: Locate the 
HId
 for the given symbol name.
Traverse:
Forward: Use DependencyGraph::forward to find callees/types.
Reverse: Use DependencyGraph::reverse if --dependents flag is set.
Extract:
Reconstruct source code from Spans (using SourceMap or reading files).
Filter body implementation for items at Depth > 0 (to save tokens).
Format:
Source: Join items into a single valid PZero file (commented).
JSON: Structured data for tooling.
Phase 3: RLM Integration (Knowledge Base)
Integrate pzero_knowledge to augment the context with relevant documentation (e.g., if code uses Result<T>, include 
kb/types/primitives.md
).
Verification Plan
Manual Verification
Create a sample project with a call chain: A -> B -> C.
Run pzero context A -> Should show A (full) + B (sig) + C (sig).
Run pzero context C --dependents -> Should show B and A.
Automated Tests
test_context_extraction: Build a mock HIR, request context, assert specific items are present/absent in ContextView.
