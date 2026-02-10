# [Pythia: Interpreting Autoregressive Transformers Across Time and Scale](https://arxiv.org/pdf/2304.01373.pdf)

This repository is for EleutherAI's project *Pythia* which combines interpretability analysis and scaling laws to understand how knowledge develops and evolves during training in autoregressive transformers. For detailed info on the models, their training, and their behavior, please see [our paper](https://arxiv.org/pdf/2304.01373.pdf).

## Models

| Params              | n_layers | d_model | n_heads | d_head | Batch Size | Learning Rate | Checkpoints                                                | Evaluations     |
| ------------------- | -------- | ------- | ------- | ------ | ---------- | ------------- | ---------------------------------------------------------- | --------------- |
| Pythia-70M          | 6        | 512     | 8       | 64     | 2M         | 1e-3          | [Here](https://huggingface.co/EleutherAI/pythia-70m)          | Ready           |
| Pythia-70M-Deduped  | 6        | 512     | 8       | 64     | 2M         | 1e-3          | [Here](https://huggingface.co/EleutherAI/pythia-70m-deduped)  | Ready           |
| Pythia-160M         | 12       | 768     | 12      | 64     | 2M         | 6e-4          | [Here](https://huggingface.co/EleutherAI/pythia-160m)         | Ready           |
| Pythia-160M-Deduped | 12       | 768     | 12      | 64     | 2M         | 6e-4          | [Here](https://huggingface.co/EleutherAI/pythia-160m-deduped) | Ready           |
| Pythia-410M         | 24       | 1024    | 16      | 64     | 2M         | 3e-4          | [Here](https://huggingface.co/EleutherAI/pythia-410m)         | Ready           |
| Pythia-410M-Deduped | 24       | 1024    | 16      | 64     | 2M         | 3e-4          | [Here](https://huggingface.co/EleutherAI/pythia-410m-deduped) | Ready           |
| Pythia-1B         | 16       | 2048    | 8       | 256   | 2M         | 3e-4          | [Here](https://huggingface.co/EleutherAI/pythia-1b)         | Ready           |
| Pythia-1B-Deduped | 16       | 2048    | 8       | 256    | 2M         | 3e-4          | [Here](https://huggingface.co/EleutherAI/pythia-1b-deduped) | Ready           |
| Pythia-1.4B         | 24       | 2048    | 16      | 128    | 2M         | 2e-4          | [Here](https://huggingface.co/EleutherAI/pythia-1.4b)         | Ready           |
| Pythia-1.4B-Deduped | 24       | 2048    | 16      | 128    | 2M         | 2e-4          | [Here](https://huggingface.co/EleutherAI/pythia-1.4b-deduped) | Ready           |
| Pythia-2.8B         | 32       | 2560    | 32      | 80     | 2M         | 1.6e-4        | [Here](https://huggingface.co/EleutherAI/pythia-2.8b)         | Ready           |
| Pythia-2.8B-Deduped | 32       | 2560    | 32      | 80     | 2M         | 1.6e-4        | [Here](https://huggingface.co/EleutherAI/pythia-2.8b-deduped) | Ready           |
| Pythia-6.9B         | 32       | 4096    | 32      | 128    | 2M         | 1.2e-4        | [Here](https://huggingface.co/EleutherAI/pythia-6.9b)         | Ready           |
| Pythia-6.9B-Deduped | 32       | 4096    | 32      | 128    | 2M         | 1.2e-4        | [Here](https://huggingface.co/EleutherAI/pythia-6.9b-deduped) | Ready           |
| Pythia-12B          | 36       | 5120    | 40      | 128    | 2M         | 1.2e-4        | [Here](https://huggingface.co/EleutherAI/pythia-12b)          | Ready |
| Pythia-12B-Deduped  | 36       | 5120    | 40      | 128    | 2M         | 1.2e-4        | [Here](https://huggingface.co/EleutherAI/pythia-12b-deduped)  | Ready |

We train and release a suite of 8 model sizes on 2 different datasets: [the Pile](https://pile.eleuther.ai/), as well as the Pile with deduplication applied.

All 8 model sizes are trained on the exact same data, in the exact same order. Each model saw 299,892,736,000 ~= 299.9B tokens during training, and *143 checkpoints* for each model are saved every 2,097,152,000 ~= 2B tokens, evenly spaced throughout training. This corresponds to just under 1 epoch on the Pile for non-"deduped" models, and ~= 1.5 epochs on the deduped Pile (which contains 207B tokens in 1 epoch).

Config files used to train these models within the [GPT-NeoX library](https://github.com/EleutherAI/gpt-neox) can be found at the `models/` directory within this repository.

We also upload the pre-tokenized data files and a script to reconstruct the dataloader as seen during training for all models. See **Reproducing Training** section for more details.

## Changelog

[April 3, 2023] We have released a new version of all Pythia models, with the following changes to our training procedure:

- All model sizes are now trained with uniform batch size of 2M tokens. Previously, the models of size 160M, 410M, and 1.4B parameters were trained with batch sizes of 4M tokens.
- We added checkpoints at initialization (step 0) and steps {1,2,4,8,16,32,64, 128,256,512} in addition to every 1000 training steps.
- Flash Attention was used in the new retrained suite. Empirically, this seems to have effected the dynamic range of model outputs in some cases, which we are investigating further.
- We remedied a minor inconsistency that existed in the original suite: all models of size 2.8B parameters or smaller had a learning rate (LR) schedule which decayed to a minimum LR of 10% the starting LR rate, but the 6.9B and 12B models all used an LR schedule which decayed to a minimum LR of 0. In the redone training runs, we rectified this inconsistency: all models now were trained with LR decaying to a minimum of 0.1× their maximum LR.
- the new `EleutherAI/pythia-1b` is trained with bf16, because in fp16 the model corrupted due to loss spikes late in training.

The old models ("V0") remain available at [https://huggingface.co/models?other=pythia_v0](https://huggingface.co/models?other=pythia_v0).

[January 20, 2023]
On January 20, 2023, we chose to rename the \textit{Pythia} model suite to better reflect including both embedding layer and unembedding layer parameters in our total parameter counts, in line with many other model suites and because we believe this convention better reflects the on-device memory usage of these models. See [https://huggingface.co/EleutherAI/pythia-410m-deduped#naming-convention-and-parameter-count](https://huggingface.co/EleutherAI/pythia-410m-deduped#naming-convention-and-parameter-count) for more details

## Quickstart

All Pythia models are hosted on [the Huggingface hub](https://huggingface.co/EleutherAI). They can be loaded and used via the following code (shown for the 3rd `pythia-70M-deduped` model checkpoint):

```python
from transformers import GPTNeoXForCausalLM, AutoTokenizer

model = GPTNeoXForCausalLM.from_pretrained(
  "EleutherAI/pythia-70m-deduped",
  revision="step3000",
  cache_dir="./pythia-70m-deduped/step3000",
)

tokenizer = AutoTokenizer.from_pretrained(
  "EleutherAI/pythia-70m-deduped",
  revision="step3000",
  cache_dir="./pythia-70m-deduped/step3000",
)

inputs = tokenizer("Hello, I am", return_tensors="pt")
tokens = model.generate(**inputs)
tokenizer.decode(tokens[0])
```

All models were trained for the equivalent of 143000 steps at a batch size of 2,097,152 tokens. Revision/branch `step143000` (e.g. [https://huggingface.co/EleutherAI/pythia-70m-deduped/tree/step143000](https://huggingface.co/EleutherAI/pythia-19m-deduped/tree/step143000)) corresponds exactly to the model checkpoint on the `main` branch of each model.

We additionally have all model checkpoints in the format accepted by the [GPT-NeoX library](https://github.com/EleutherAI/gpt-neox), but do not serve them at scale due to size of optimizer states and anticipated lower demand. If you would like to perform analysis using the models within the GPT-NeoX codebase, or would like the optimizer states, please email hailey@eleuther.ai and stella@eleuther.ai to arrange access.


*`pythia-{size}-v0` models on Huggingface of sizes `160m, 410m, 1.4b` were trained with a batch size of 4M tokens  and were originally trained for 71500 steps instead, and checkpointed every 500 steps. The checkpoints on Huggingface for these v0 models are renamed for consistency with all 2M batch models, so `step1000` is the first checkpoint for `pythia-1.4b-v0` that was saved (corresponding to step 500 in training), and `step1000` is likewise the first pythia-6.9b-v0 checkpoint that was saved (corresponding to 1000 "actual" steps.)*

## Reproducing Training

We provide the training data for replication of our training runs. The [GPT-NeoX library](https://github.com/EleutherAI/gpt-neox) requires the pre-tokenized training data in the form of 2 memory-mapped numpy arrays: a `.bin` and `.idx` file.

We provide these files, hosted on the Hugging Face hub.

To download and use the deduplicated Pile training data, run:

```bash
git lfs clone https://huggingface.co/datasets/EleutherAI/pythia_deduped_pile_idxmaps

python utils/unshard_memmap.py --input_file ./pythia_deduped_pile_idxmaps/pile_0.87_deduped_text_document-00000-of-00082.bin --num_shards 83 --output_dir ./pythia_pile_idxmaps/
```
This will take over a day to run, though it should not require more than 5 GB of RAM. We recommend downloading this rather than retokenizing the Pile from scratch, in order to preserve the data order seen by the Pythia models.

TODO: forthcoming: more information on how to replicate + relaunch the Pythia training runs, once the data is actually downloaded.


### Dataset Viewer

We provide a tool to view particular portions of the training dataloader used by all models during training, at `utils/batch_viewer.py`.

To run, first substitute the filepath to the downloaded `.bin` and `.idx` files for either the Pile or deduplicated Pile in `utils/dummy_config.yml`.

```python
PYTHONPATH=utils/gpt-neox/ python utils/batch_viewer.py \
  --start_iteration 0 \
  --end_iteration 1000 \
  --mode save \
  --conf_dir utils/dummy_config.yml 
```

Passing `--mode save` will save a separate file containing each batch as a numpy array. 

Passing `--mode custom` will save a dictionary for each batch to a JSONL file--it can be used to compute arbitrary statistics over each batch seen during training.


## Benchmark Scores

We also provide benchmark 0-shot and 5-shot results on a variety of NLP datasets:

- Lambada (`lambada_openai`)
- Wikitext (`wikitext`)
- PiQA (`piqa`)
- SciQ (`sciq`)
- WSC (`wsc`)
- Winogrande (`winogrande`)
- ARC-challenge (`arc_challenge`)
- ARC-easy (`arc_easy`)
- LogiQA (`logiqa`)
- BLiMP (`blimp_*`)
- MMLU (`hendrycksTest*`)

Evaluations were performed in GPT-NeoX using the [LM Evaluation Harness](https://github.com/EleutherAI/lm-evaluation-harness), and are viewable by model and step at `results/json/v1.1-evals/*` in this repository.



## Citation Details

If you use the Pythia models or data in your research, please consider citing our paper via:

```
@misc{biderman2023pythia,
      title={Pythia: A Suite for Analyzing Large Language Models Across Training and Scaling}, 
      author={Stella Biderman and Hailey Schoelkopf and Quentin Anthony and Herbie Bradley and Kyle O'Brien and Eric Hallahan and Mohammad Aflah Khan and Shivanshu Purohit and USVSN Sai Prashanth and Edward Raff and Aviya Skowron and Lintang Sutawika and Oskar van der Wal},
      year={2023},
      eprint={2304.01373},
      archivePrefix={arXiv},
      primaryClass={cs.CL}
}
```

## License

```
   Copyright 2023 EleutherAI

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
```



# PZero — Development Plan

## LLM-Optimized Programming Language

---

## 1. Vision Statement

PZero is a programming language designed for **LLM AI agents as the primary developer**, not humans. Every design decision optimises for how language models generate, reason about, and verify code. Humans remain in the loop as architects, spec writers, and reviewers — but the language itself is built for machines to write correctly on the first attempt.

---

## 2. Transpilation Target: Explicit Recommendation

### Recommendation: **Transpile to a restricted subset of Rust**

This is the explicit recommendation and the reasoning follows.

### What "restricted subset of Rust" means

The generated Rust code will deliberately avoid hard-to-generate Rust features. Specifically:

| Feature | Strategy in generated code |
|---|---|
| Ownership / borrowing | Avoid. Use `Clone` and `Arc<T>` everywhere. |
| Lifetime annotations | Never generated. All data is either owned or `Arc`-wrapped. |
| Mutable references (`&mut`) | Rarely generated. Functional core means most data is immutable. |
| Sum types / enums | Direct mapping — PZero ADTs → Rust enums (natural fit). |
| Pattern matching | Direct mapping — PZero match → Rust match. |
| Traits | Used for PZero interfaces/typeclasses. |
| Error handling | PZero `Result<T, E>` → Rust `Result<T, E>` directly. |
| Memory allocation | Heap-allocate freely via `Box<T>` and `Arc<T>`. Optimise later. |
| Async | Not in v1. Add later via `tokio` codegen if needed. |

### Why Rust over the alternatives

| Target | Verdict | Reason |
|---|---|---|
| **Rust (restricted subset)** | **✅ CHOSEN** | Best semantic match. Enums, pattern matching, Result/Option, no null, strong types — all map 1:1 from PZero. Generated code compiles to native binaries. Access to `cargo` ecosystem. |
| Go | ❌ Rejected | No sum types, no pattern matching, `nil` everywhere. You'd spend more effort emulating PZero semantics in Go than just generating restricted Rust. |
| C | ❌ Rejected | Manual memory management codegen is a project in itself. No type-level safety in the output. |
| LLVM IR | ❌ Rejected | Massive complexity. You'd be writing a full compiler backend. Kills achievability. |
| Cranelift / WASM | ⏸️ Future option | Worth revisiting in v2 for portable/sandboxed execution. Too much infrastructure for v1. |

### The key insight

Generating *idiomatic* Rust is hard. Generating *valid, compiling* Rust that uses `Arc<T>` and `Clone` liberally is **straightforward**. The generated code won't win style points, but it will:
- Compile reliably
- Run correctly
- Produce native binaries
- Benefit from Rust's optimiser (LLVM backend)
- Integrate with the cargo ecosystem

Performance tuning the generated code is a v2+ concern. Correctness and achievability come first.

### Risk mitigation

If restricted-Rust codegen proves harder than expected in practice (e.g., trait resolution edge cases, orphan rules), the fallback is:

**Fallback: Transpile to C with a simple reference-counted GC**

C is universally compilable, and a reference-counted allocator for an immutable-first language is well-understood. This fallback exists as insurance, not as the plan.

---

## 3. Core Design Principles (Prioritised)

These are ordered by **implementation priority** — build the foundation first, then layer on advanced features.

| Priority | Principle | Phase |
|---|---|---|
| P0 | Familiar-but-unique syntax identity | Phase 1 |
| P0 | Explicit, minimal grammar (PEG-style, unambiguous) | Phase 1 |
| P0 | Strong, static typing with type inference | Phase 1–2 |
| P0 | Explicit scope, bracketed blocks, no implicit context | Phase 1 |
| P1 | No null — Option/Result types mandatory | Phase 2 |
| P1 | Functional core + explicit effects | Phase 2 |
| P1 | Canonical formatting (single valid style) | Phase 2 |
| P1 | First-class testing & TDD workflow | Phase 3 |
| P1 | Dependency context generation for LLM debugging | Phase 2–3 |
| P2 | AI-optimised structured error feedback | Phase 3 |
| P2 | Intent-aware error recovery & suggestions | Phase 3 |
| P2 | Type-level contracts (refinement types, pre/post) | Phase 4 |
| P2 | Built-in documentation as structured data | Phase 4 |
| P3 | Mocks, stubs, DI via effect handlers | Phase 5 |
| P3 | Multi-agent coordination primitives | Phase 6+ |

---

## 4. Specific Objective: Familiar-But-Unique Syntax Identity

### Problem

If PZero syntax looks identical to Rust, LLMs will generate Rust semantics. If it looks identical to Go, they'll assume Go semantics. The syntax must be **close enough to trigger correct structural patterns** but **distinct enough that the LLM recognises it as PZero** and applies PZero-specific rules.

### Strategy: "Rust-shaped, uniquely keyed"

The syntax borrows structural patterns from the C-family (braces, semicolons, `fn`, typed parameters) because LLMs have billions of training tokens in this style. But it introduces **deliberate, consistent markers** that signal "this is PZero":

#### Familiarity anchors (keep these from Rust/Go/TS)

- `{}` for blocks
- `;` as statement terminator (mandatory, not optional)
- `fn` for function declarations
- `let` for bindings
- `if / else / match` for control flow
- `Type` annotation syntax: `name: Type`
- `//` for line comments

#### Unique identity markers (things that make PZero recognisable)

These are suggestions to evaluate during syntax design — the exact choices are Phase 1 deliverables:

| Concept | Familiar form | PZero candidate | Rationale |
|---|---|---|---|
| Module declaration | `mod`, `package`, `module` | `module` | Explicit, readable |
| Effect annotation | (none standard) | `effect` keyword | Signals impure code — unique to PZero |
| Test block | `#[test]`, `describe()` | `test` as first-class block | `test "name" { ... }` inline with function |
| Contract/spec | (none standard) | `require` / `ensure` | Pre/post conditions as keywords |
| Pipe operator | `\|>` (Elixir/F#) | `\|>` | Functional composition — familiar from F#/Elixir but rare in C-family |
| Type alias | `type X = Y` | `type X = Y` | Keep familiar |
| Sum types / ADT | `enum` (Rust) | `enum` | Keep familiar |
| Struct / record | `struct` (Rust/Go) | `struct` | Keep familiar |
| Error handling | `?` operator (Rust) | `?` operator | Keep familiar |
| Pure function marker | (none standard) | inferred — all `fn` are pure unless `effect` is declared | Purity by default is the identity |

#### The identity rule

> **Every PZero function is pure by default. Side effects require the `effect` keyword.**

This single rule is the language's philosophical fingerprint. It means:
- LLMs learn: "if I see `fn`, it's pure — I can reason about it deterministically"
- LLMs learn: "if I see `effect fn`, I need to handle side effects explicitly"
- No other mainstream language works this way syntactically

#### Syntax example (illustrative, subject to Phase 1 design)

```
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
```

**Key signals an LLM would pick up:**
- `module` keyword → "not Rust, not Go"
- `test` block inline → "not any mainstream language"
- `require` / `ensure` as keywords → "this language has contracts"
- `effect fn` → "this is PZero, purity is default"
- Everything else (braces, semicolons, `fn`, `match`, `enum`) → familiar muscle memory

### Validation approach

During Phase 1, we will test syntax candidates by prompting frontier LLMs with partial PZero code and measuring:
1. Can the LLM complete the code correctly given a few examples?
2. Does the LLM confuse PZero with another language?
3. Does the LLM respect PZero-specific rules (purity, contracts, test blocks)?

---

## 5. Specific Objective: Dependency Context Generation for LLM Debugging

### Problem

When an LLM encounters a bug, its first task is gathering relevant context. In most languages, this means:
1. Read the failing function
2. Guess which other functions/types are involved
3. Search the codebase (grep, semantic search, file reads)
4. Hope it found everything
5. Often miss a transitive dependency and produce a wrong fix

This is slow, unreliable, and wastes the LLM's context window on irrelevant code. **The language should do this automatically.**

### Solution: The compiler knows the full dependency graph

After type checking, PZero's compiler has a **complete, typed dependency graph** — it knows exactly which functions call which other functions, which types are used where, which modules depend on which modules. This is the same graph used for effect checking (2.4) and name resolution (2.6).

The context generation feature exposes this graph to the LLM via CLI commands.

### How it works

#### `pzero context <symbol>` — Get everything needed to understand a symbol

Given a function name (or file:line), the compiler walks the dependency graph and outputs a **self-contained context slice**: every definition the LLM needs to understand and modify that symbol.

Example: If `process_order` calls `validate_order`, which uses `Order` struct and `ValidationError` enum, and `validate_order` calls `check_stock` which uses `Inventory`:

```
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
```

The output is:
- **Ordered by dependency depth** — direct target first, then immediate dependencies, then transitive
- **Self-contained** — an LLM can understand the entire context without reading any other file
- **Trimmed** — function bodies beyond depth 1 are replaced with `{ ... }` to save context window space (configurable with `--depth N`)
- **Includes contracts** — `require`/`ensure` blocks are included because they're essential for understanding expected behaviour

#### `pzero context` JSON mode — Machine-consumable

```
$ pzero context process_order --format json
```

```json
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
```

The `context_tokens` field tells the LLM exactly how much of its context window this will consume.

#### `pzero diagnose` — Error + context in one command

This is the killer feature for LLM debugging workflows. It combines test execution with context generation:

```
$ pzero diagnose
```

```json
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
```

The LLM receives:
1. **What failed** — structured error with trace
2. **All the code it needs to understand why** — dependency context of the failing test/function
3. **A suggestion** — based on the contract that failed
4. **Nothing irrelevant** — context is precisely scoped, not the entire codebase

#### Context depth control

| Flag | Behaviour |
|---|---|
| `--depth 0` | Target function/type only |
| `--depth 1` | Target + direct dependencies (default) |
| `--depth 2` | Target + direct + transitive dependencies |
| `--depth full` | Everything reachable (may be large) |
| `--max-tokens N` | Stop expanding context when it would exceed N tokens |

The `--max-tokens` flag is specifically designed for LLMs — the agent can say "give me context but keep it under 4000 tokens" and the compiler will prioritise the most relevant dependencies.

### Why this is almost free to implement

The dependency graph is **already built** by Phase 2 tasks:

| Phase 2 task | What it produces | Reused for context generation |
|---|---|---|
| 2.5 Module resolution | Module dependency graph | ✅ Which modules to include |
| 2.6 Name resolution | Symbol → declaration mapping | ✅ Which symbols to look up |
| 2.4 Effect checker | Call graph | ✅ Which functions call which |
| 2.3 Type checker | Type → usage mapping | ✅ Which types are involved |

The context generation engine (2.10) is essentially a **graph traversal + source extraction** on data structures that already exist. The `pzero diagnose` command (3.13) wraps the test runner output with a context lookup per failure. Neither requires new analysis — just new output formatting.

---

## 6. Phased Development Plan

### Phase 1 — Foundation: Lexer, Parser, AST (Months 1–2)

**Goal:** Parse PZero source into a well-typed AST. No code generation yet.

| # | Task | Detail |
|---|---|---|
| 1.1 | **Design and finalise syntax** | Write the formal grammar (PEG). Resolve all ambiguity. Test with LLM prompting experiments. Produce a `grammar.peg` file. |
| 1.2 | **Define the AST data structures** | Rust enums + structs for every node type. This is the central data structure everything else operates on. |
| 1.3 | **Implement lexer** | Use `logos` crate for fast, declarative tokenisation. Define all tokens. |
| 1.4 | **Implement parser** | Recursive descent parser (hand-written for control and good error messages). Produce AST from token stream. |
| 1.5 | **Implement pretty-printer** | Canonical formatter: AST → source. Establishes the single valid formatting style. |
| 1.6 | **Error recovery in parser** | When parsing fails, attempt to identify what the user/LLM intended (edit-distance matching against known syntax patterns). Produce structured error JSON. |
| 1.7 | **Test suite for parser** | Extensive test corpus — valid programs, invalid programs with expected errors, edge cases. |

**Deliverables:**
- `grammar.peg` — formal grammar specification
- `pzero parse <file>` CLI command that outputs AST as JSON
- `pzero fmt <file>` CLI command that canonically formats source
- Structured JSON error output for all parse failures
- Parser test suite (aim for 200+ test cases)

---

### Phase 2 — Type System & Semantic Analysis (Months 3–4)

**Goal:** Validate that parsed programs are type-correct. No code generation yet.

| # | Task | Detail |
|---|---|---|
| 2.1 | **Define type system** | Core types: `Int`, `Float`, `Bool`, `String`, `Option<T>`, `Result<T,E>`, `List<T>`, `Map<K,V>`, user-defined structs, enums, function types. No `null`. |
| 2.2 | **Implement type inference** | Hindley-Milner style with local inference. Types flow forward through `let` bindings and function bodies. Function signatures are always explicitly typed (no global inference). |
| 2.3 | **Implement type checker** | Walk the AST, assign types to every node, report type errors with structured JSON diagnostics. |
| 2.4 | **Implement effect checker** | Verify that functions without `effect` do not call effectful functions. This is a simple taint analysis on the call graph. |
| 2.5 | **Implement module resolution** | Resolve `module` declarations and imports. Build a dependency graph. Detect cycles. |
| 2.6 | **Implement name resolution** | Resolve all identifiers to their declarations. Report "undefined variable" with suggestions (edit-distance). |
| 2.7 | **Contract parsing** | Parse `require` and `ensure` blocks. Attach them to AST function nodes. (Verification is Phase 4 — for now, just parse and store.) |
| 2.8 | **Typed AST output** | Produce a fully-typed, annotated AST for the codegen phase. |
| 2.9 | **Dependency graph construction** | Build a full dependency graph from the typed AST: function → functions it calls, types it uses, modules it imports. Store as a queryable graph data structure. |
| 2.10 | **Context extraction engine** | Given a symbol (function, type, test), walk the dependency graph and extract the transitive closure: all definitions the symbol depends on, recursively. Output as a self-contained PZero source slice. |
| 2.11 | **`pzero context` CLI command** | `pzero context <file>:<line>` or `pzero context <symbol>` outputs the complete context an LLM needs to understand and fix that symbol. Structured JSON and source-code output modes. |

**Deliverables:**
- `pzero check <file>` CLI command that type-checks and reports errors
- `pzero context <symbol>` CLI command that outputs dependency context
- Structured JSON diagnostics for all type/effect/name errors
- Type checker test suite (aim for 300+ test cases)

---

### Phase 3 — Code Generation & Test Runner (Months 5–7)

**Goal:** Generate compiling Rust code from type-checked PZero. Run tests.

| # | Task | Detail |
|---|---|---|
| 3.1 | **Design Rust codegen mapping** | Document how every PZero construct maps to Rust. Define the restricted Rust subset. |
| 3.2 | **Implement codegen: expressions** | Literals, arithmetic, function calls, match, if/else → Rust equivalents. |
| 3.3 | **Implement codegen: data types** | Structs, enums → Rust structs + enums with `#[derive(Clone, Debug, PartialEq)]`. |
| 3.4 | **Implement codegen: functions** | `fn` → `fn`. `effect fn` → `fn` (effects are checked at PZero level, Rust doesn't need to know). |
| 3.5 | **Implement codegen: modules** | PZero modules → Rust modules + `mod.rs` scaffolding. |
| 3.6 | **Implement codegen: memory** | All compound types wrapped in `Arc<T>`. Clone on assignment. No lifetime annotations. |
| 3.7 | **Implement codegen: Option/Result** | Direct 1:1 mapping. `?` operator passes through. |
| 3.8 | **Implement test runner** | PZero `test` blocks → Rust `#[test]` functions. `pzero test` invokes `cargo test` on the generated project. |
| 3.9 | **Implement contract runtime checks** | `require` → `assert!` at function entry. `ensure` → `assert!` before return. (Runtime enforcement for now; static verification in Phase 4.) |
| 3.10 | **End-to-end pipeline** | `pzero build <file>` → parse → check → codegen → `cargo build`. `pzero run <file>` → build → execute. |
| 3.11 | **Structured build errors** | Capture `cargo` errors, map back to PZero source locations, output as structured JSON. |
| 3.12 | **Error-triggered context generation** | When a build or test fails, automatically attach the dependency context of the failing function/test to the error output. The LLM receives the error AND everything it needs to fix it in a single response. |
| 3.13 | **`pzero diagnose` command** | Combines `pzero test` + `pzero context`: runs tests, and for each failure, outputs the error plus the full dependency context of the failing test. Single command gives the LLM everything it needs. |

**Deliverables:**
- `pzero build` — compiles PZero to native binary via Rust
- `pzero run` — build and execute
- `pzero test` — build and run all test blocks
- `pzero diagnose` — run tests + output error context for failures
- Source-mapped error output (Rust errors → PZero line numbers)
- Automatic dependency context attached to every error
- End-to-end test suite: PZero source → expected output

---

### Phase 3.5 — Frontend Target: JS Codegen & UI Runtime (Months 8–10)

**Goal:** Compile the same PZero source to JavaScript for browser execution. One language, two targets.

#### Design philosophy

> PZero is ONE language with ONE semantic model that compiles to different runtimes.
> It can describe UI state and behaviour, and we compile it to an existing frontend platform.

The frontend target reuses 100% of the compiler frontend (lexer, parser, type checker, effect checker). Only the codegen module is new.

#### UI architecture: The Elm Architecture (TEA)

The UI model is functional and fits PZero's core philosophy:

| Concept | PZero construct |
|---|---|
| **Model** (state) | A `struct` — immutable, explicit |
| **Msg** (events) | An `enum` — exhaustive, typed |
| **update** | A pure `fn(Msg, Model) -> Model` — deterministic, testable |
| **view** | A pure `fn(Model) -> Html<Msg>` — declarative, no side effects |
| **effects** | `effect fn` returning `Cmd<Msg>` — explicit, mockable |

This architecture is **ideal for LLMs** because:
- Every function is pure and testable in isolation
- State transitions are explicit enum matches — LLMs excel at exhaustive pattern matching
- No hidden DOM state, no lifecycle methods, no `this`
- The entire UI is a function from data to description

#### Type-checked UI: Interface checking works out of the box

The `Html<Msg>` type system gives us **compile-time UI correctness for free**:

| What the type system catches | How |
|---|---|
| Invalid element nesting | `Html<Msg>` constructors enforce which children are valid — e.g., `tr` only inside `table`, `li` only inside `ul`/`ol` |
| Mismatched event types | `on_click(Msg::Submit)` only compiles if `Submit` is a variant of the page's `Msg` enum |
| Missing state handling | `match msg { ... }` must be exhaustive — the compiler forces the LLM to handle every possible user action |
| Invalid attribute types | `value(count)` requires `count: String` — type mismatch caught at compile time |
| Broken component contracts | If a view function requires `Model` with field `users: List<User>`, passing a model without that field is a compile error |

This means the LLM gets **instant feedback on UI structure correctness before the browser is even opened**. Most frontend bugs that slip through in JavaScript/React are caught here at compile time.

#### Three-tier UI testing strategy

PZero provides three levels of UI testing, each giving the LLM progressively more confidence:

| Level | What it tests | How it runs | Speed |
|---|---|---|---|
| **Tier 1: Type checking** | UI structure, event wiring, state shape | `pzero check` — compile time | Instant |
| **Tier 2: Unit tests** | `update` logic, `view` output structure | `pzero test` — pure function tests, no browser | Fast (~ms) |
| **Tier 3: Browser tests** | Visual rendering, user interactions, full flows | `pzero test --browser` — Playwright against live app | Slower (~sec) |

The LLM development loop becomes:

```
1. Write/modify PZero UI code
2. `pzero check`        → Type errors? Fix immediately (instant feedback)
3. `pzero test`         → Unit test failures? Fix logic (fast feedback)
4. `pzero test --browser` → Visual/interaction failures? Fix UI (rich feedback)
5. All green → commit
```

Each tier catches different classes of bugs, and the LLM gets structured JSON feedback at every stage.

#### Playwright integration: LLM-driven browser testing

Playwright is the ideal browser testing framework for LLM agents because:
- **Structured API**: Playwright's selector and assertion model is explicit and type-safe — LLMs generate correct Playwright code reliably
- **Headless by default**: No GUI needed — LLM agents run tests in CI-style headless mode
- **Built-in waiting**: Auto-waits for elements, reducing flaky tests that confuse LLMs
- **Trace & screenshot capture**: On failure, produces visual evidence the LLM can reason about (via structured error output)

PZero integrates Playwright as a **first-class browser test primitive**:

```
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
```

The `browser_test` block is a new PZero primitive that:
- Compiles to Playwright test code (TypeScript) under the hood
- Runs via `pzero test --browser` which starts `pzero serve` + Playwright in headless mode
- Returns structured JSON results: pass/fail, selectors that failed, screenshots on failure, DOM snapshots
- Uses PZero's familiar syntax — the LLM doesn't need to context-switch to a different language

**Structured browser test failure output (JSON):**

```json
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
```

This gives the LLM everything it needs to diagnose and fix the issue in one iteration.

| # | Task | Detail |
|---|---|---|
| 3.5.1 | **JS codegen module** | New backend: Typed AST → JavaScript source. Simpler than Rust codegen (no ownership, no lifetimes). Target ES2020+. |
| 3.5.2 | **JS codegen: expressions & data types** | Map PZero types to JS. Structs → object literals with `Object.freeze`. Enums → tagged unions `{ tag: "Variant", ...fields }`. Pattern matching → `switch` on tag. |
| 3.5.3 | **JS codegen: functions & modules** | Pure functions → JS functions. Modules → ES modules with explicit exports. |
| 3.5.4 | **JS codegen: Option/Result** | Same tagged-union representation. `None` → `{ tag: "None" }`. `Some(x)` → `{ tag: "Some", value: x }`. |
| 3.5.5 | **Minimal UI runtime** | ~500 lines of JS: virtual DOM diffing + patching, event delegation, TEA message loop. OR use existing library (`morphdom`, `lit-html`) as the DOM backend. |
| 3.5.6 | **Html<Msg> stdlib module** | PZero module defining `Html<Msg>` type and element constructors (`div`, `span`, `button`, `input`, `on_click`, `on_input`, etc.). |
| 3.5.7 | **Effect runtime for browser** | Handlers for browser effects: HTTP (`fetch`), timers, local storage, URL routing. Mapped from PZero `effect fn` declarations. |
| 3.5.8 | **Dev server** | `pzero serve` — file watcher + recompile + hot reload in browser. Wraps a simple HTTP server. |
| 3.5.9 | **Test runner in JS** | PZero `test` blocks → JS test functions. `pzero test --target js` runs them in Node.js. Verifies that the same tests pass on both Rust and JS targets. |
| 3.5.10 | **Cross-target test validation** | Run the same PZero test suite against both Rust and JS codegen. If a test passes on one target but fails on the other, that's a codegen bug. |
| 3.5.11 | **Playwright integration** | `browser_test` blocks compile to Playwright TypeScript tests. `pzero test --browser` starts `pzero serve` in headless mode and runs Playwright against it. |
| 3.5.12 | **Browser test primitives** | Built-in functions: `navigate()`, `click()`, `type_text()`, `query()`, `assert_text()`, `assert_visible()`, `assert_count()`, `screenshot()`, `wait_for()`. All compile to Playwright API calls. |
| 3.5.13 | **Structured browser test output** | On failure: JSON with assertion details, expected vs actual, selector, screenshot path, DOM snapshot path, and fix suggestion. |
| 3.5.14 | **Test artifact capture** | Screenshots and DOM snapshots saved to `artifacts/` on failure. Trace files for debugging complex interaction sequences. |
| 3.5.15 | **data-testid generation** | Compiler auto-generates `data-testid` attributes on elements that have event handlers or dynamic content, making them queryable in browser tests without manual annotation. |

**Deliverables:**
- `pzero build --target js <file>` — compiles PZero to a JS bundle
- `pzero serve` — dev server with hot reload
- `pzero test --target js` — runs unit tests in Node.js
- `pzero test --browser` — runs browser tests via Playwright (headless)
- `Html<Msg>` stdlib module for declarative UI
- `browser_test` block as a first-class language primitive
- Structured JSON output for all three test tiers (type check, unit, browser)
- Auto-generated `data-testid` attributes for testability
- Cross-target test validation (same unit tests, both backends)
- 5+ example UI applications with full test suites (counter, todo list, form, data table, API consumer)

#### What this enables

A developer (or LLM agent) writes **one codebase**:

```
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
```

---

### Phase 4 — Contracts, Diagnostics & LSP (Months 11–13)

**Goal:** Advanced type-level contracts and development tooling.

| # | Task | Detail |
|---|---|---|
| 4.1 | **Refinement types** | `x: Int where x > 0` — checked at function boundaries at runtime, with static analysis for obvious cases. |
| 4.2 | **Static contract checking** | For simple cases (integer bounds, non-empty lists), verify contracts at compile time without SMT. Use abstract interpretation. |
| 4.3 | **Enhanced error diagnostics** | Add intent inference: when the parser/checker fails, analyse what the LLM was probably trying to do and suggest the correct syntax/type. |
| 4.4 | **Property-based test generation** | Given a function's type signature and contracts, auto-generate property tests. E.g., `fn add(a: Int, b: Int) -> Int` → test commutativity, identity. |
| 4.5 | **LSP server** | Language Server Protocol for editor integration (VS Code). Completion, diagnostics, go-to-definition, hover types. |
| 4.6 | **Structured documentation** | `///` doc comments parsed as structured data (params, returns, errors, examples). `pzero doc` generates JSON/HTML docs. |

**Deliverables:**
- Refinement type checking (runtime + partial static)
- Auto-generated property tests
- LSP server (VS Code extension)
- `pzero doc` command

---

### Phase 5 — Standard Library & Effect Handlers (Months 14–16)

**Goal:** A usable standard library and the effect/DI system.

| # | Task | Detail |
|---|---|---|
| 5.1 | **Core stdlib** | Collections (`List`, `Map`, `Set`), string operations, math, basic IO. |
| 5.2 | **Effect handler system** | Algebraic-style effects: define effect interfaces, provide handlers. This replaces traditional DI/mocking. |
| 5.3 | **Mock generation via effects** | `effect fn read_file(...)` → in tests, provide a mock handler that returns fake data. No special mock syntax needed. |
| 5.4 | **Concurrency primitives** | Structured concurrency via effects. `effect fn spawn(task: fn() -> T) -> Future<T>`. |
| 5.5 | **Package manager** | `pzero.toml` manifest file. Dependency resolution. Registry (can use a git-based registry initially). |

**Deliverables:**
- Usable standard library
- Effect handler system with test mocking
- Basic package management

---

### Phase 6 — Ecosystem & Advanced Features (Month 17+)

### Phase 6 — Ecosystem & Advanced Features (Month 17+)

**Goal:** Production readiness and advanced features.

| # | Task | Detail |
|---|---|---|
| 6.1 | **LLM integration toolkit** | Ship a `grammar.json` + `stdlib.json` + prompt templates that LLMs can use to generate correct PZero. |
| 6.2 | **Multi-agent primitives** | Task queues, dependency declarations, parallel composition — as stdlib, not language primitives (keeps core simple). |
| 6.3 | **WASM target** | Add Cranelift/WASM as an alternative backend for portable/sandboxed execution. |
| 6.4 | **Performance optimisation** | Analyse generated Rust code. Replace `Arc<T>` with owned types where provably safe. Reduce cloning. |
| 6.5 | **Self-hosting** | Eventually, write the PZero compiler in PZero itself. |
| 6.6 | **Mobile targets** | Investigate compiling PZero UI to React Native / native mobile via additional codegen backends. |

---

## 7. Architecture Overview

```
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
```

All stages emit **structured JSON diagnostics** on error, with:
- Error code
- Source location (file, line, column, span)
- What was expected vs. what was found
- Intent inference ("did you mean...?")
- Fix suggestion (replacement text)

---

## 8. Technology Stack

| Component | Technology | Reason |
|---|---|---|
| Compiler implementation | Rust | Best for compiler front-ends: enums, pattern matching, safety |
| Lexer | `logos` crate | Fast, declarative, well-maintained |
| Parser | Hand-written recursive descent | Best error messages, full control over recovery |
| Codegen target (backend) | Rust (restricted subset) | Best semantic mapping, native output, cargo ecosystem |
| Codegen target (frontend) | JavaScript (ES2020+) | Universal browser support, ecosystem access |
| UI runtime | Minimal virtual DOM (~500 LOC) | TEA architecture, no framework dependency |
| Test runner (backend) | Wraps `cargo test` | Reuse Rust's test infrastructure |
| Test runner (frontend) | Wraps Node.js | Same tests, different runtime |
| Build orchestration | Wraps `cargo build` + JS bundler | Reuse existing build infrastructure |
| Dev server | Built-in HTTP + file watcher | `pzero serve` for frontend dev |
| LSP server | `tower-lsp` crate | Standard Rust LSP framework |
| CLI framework | `clap` crate | Standard Rust CLI framework |
| Error output | JSON (structured) + human-readable (coloured) | Dual-mode: machine-first, human-friendly |

---

## 9. Success Criteria

### v0.1 (end of Phase 3 — ~Month 7)
- [ ] PZero source can be written, parsed, type-checked, and compiled to a native binary
- [ ] Test blocks run and report pass/fail
- [ ] Errors are structured JSON with source locations
- [ ] An LLM (GPT-4/Claude) can generate valid PZero given 5 examples in-context
- [ ] 10 non-trivial example programs compile and run correctly

### v0.3 (end of Phase 3.5 — ~Month 10)
- [ ] Same PZero source compiles to both native binary (Rust) and JS bundle (browser)
- [ ] TEA-based UI applications run in the browser
- [ ] `pzero serve` provides dev server with hot reload
- [ ] Same unit test suite passes on both Rust and JS targets
- [ ] `browser_test` blocks run via Playwright in headless mode
- [ ] Structured JSON output from all three test tiers (type check → unit → browser)
- [ ] LLM can write a complete UI feature (model + update + view + tests + browser tests) from a spec
- [ ] 5 example UI applications with full three-tier test suites

### v0.5 (end of Phase 5 — ~Month 16)
- [ ] Contracts (require/ensure) work at runtime
- [ ] Effect system allows mocking in tests
- [ ] LSP provides completion and diagnostics in VS Code
- [ ] Standard library covers basic data structures and IO
- [ ] An LLM can generate PZero with >80% first-pass compilation success (measured on a benchmark suite)

### v1.0 (Phase 6+)
- [ ] Package manager and registry
- [ ] LLM integration toolkit shipped
- [ ] Self-hosting feasibility demonstrated
- [ ] Community of early adopters

---

## 10. Open Questions to Resolve in Phase 1

These need answers before writing the first line of parser code:

1. **File extension**: `.ll` conflicts with LLVM IR. Alternatives: `.p0`, `.lang`, `.ai`. Recommendation: `.p0`
2. **Entry point**: `fn main()` like Rust/C? Or implicit top-level execution? Recommendation: explicit `fn main()`.
3. **String interpolation**: `"hello {name}"` (Rust-style) or template literals? Need to decide syntax.
4. **Generic syntax**: `List<T>` (angle brackets) or `List[T]` (square brackets, avoids HTML/XML confusion for LLMs)? Recommendation: angle brackets — more training data.
5. **Semicolons**: Mandatory (Rust/C) or optional with newline-as-terminator (Go)? Recommendation: mandatory — removes ambiguity.
6. **Whitespace**: Completely insignificant (C-style) or meaningful indentation (Python)? Recommendation: insignificant — per principle #6.
7. **Standard library surface area for v1**: What's the minimum usable stdlib?

---

## 11. Immediate Next Steps

1. **Set up the Rust project** — `cargo init` with workspace structure for compiler crates
2. **Write the PEG grammar** — formal syntax definition, iterate with examples
3. **Test the syntax with LLMs** — prompt GPT-4/Claude with example PZero code and measure generation accuracy
4. **Implement lexer** — first compilable component
5. **Implement parser** — the core of Phase 1




################################################################################################################
################################################################################################################

# Phase 1 — Detailed Subplan

## Foundation: Syntax Design, Lexer, Parser, AST

**Duration:** Months 1–2 (8 weeks)  
**Goal:** Parse PZero source into a well-typed AST with structured error output.

---

## 0. Prerequisites: Decisions We Must Lock Before Writing Code

These are the open questions from the development plan. Every one of them affects the lexer and parser. We cannot write a single token definition until these are resolved.

### Decision 1: File Extension

| Option | Conflict? | LLM recognition |
|---|---|---|
| `.ll` | ❌ Conflicts with LLVM IR | Poor — LLMs will assume LLVM |
| `.p0` | ✅ No conflict | Good — unique, searchable, unambiguous |
| `.lang` | Possible conflict (generic) | Moderate — too generic |
| `.ai` | Possible conflict (AI file formats) | Poor — overloaded term |

**Decision: `.p0`**

Rationale: Unique, no conflicts, LLMs won't confuse it with anything else. Searchable in codebases. Short, memorable, and mirrors the language name PZero.

### Decision 2: Entry Point

| Option | Familiar to LLMs? | Explicit? |
|---|---|---|
| `fn main()` | ✅ Very — C, Rust, Go, Java | ✅ Yes |
| Top-level execution (like Python/JS) | ✅ Familiar | ❌ Implicit — which file runs first? |
| `fn main()` required in the root module only | ✅ | ✅ |

**Decision: `fn main()` required in the root module**

Rationale: Every LLM knows `fn main()`. It's explicit. Avoids the "what runs first?" ambiguity that plagues Python scripts. Library modules don't need it — only the entry-point module.

### Decision 3: String Interpolation

| Option | Example | Familiar from | Risk |
|---|---|---|---|
| Braces in strings | `"hello {name}"` | Rust (but Rust needs `format!`), Python f-strings | Could conflict with braces in expressions |
| Dollar-braces | `"hello ${name}"` | JavaScript, Kotlin, Shell | Well-understood, less ambiguous |
| No interpolation (explicit concat) | `"hello " + name` | Go, C | Verbose, LLMs generate it fine |

**Decision: `"hello ${name}"` (dollar-brace)**

Rationale: The `$` prefix eliminates ambiguity with expression braces. JavaScript and Kotlin use this — billions of training tokens. Simple to lex: `$` followed by `{` triggers interpolation mode.

### Decision 4: Generic Syntax

| Option | Example | Familiar from | Risk |
|---|---|---|---|
| Angle brackets | `List<Int>` | Rust, Java, C++, TS | None — overwhelmingly common in training data |
| Square brackets | `List[Int]` | Python type hints, Scala | Less training data |

**Decision: `List<Int>` (angle brackets)**

Rationale: LLMs default to angle brackets for generics. More training data by far. The "ambiguity with comparison operators" issue (C++) does not apply to PZero because we have a simpler grammar where the context is always clear (types vs expressions are syntactically distinct positions).

### Decision 5: Semicolons

| Option | Example | Familiar from | Risk |
|---|---|---|---|
| Mandatory | `let x = 5;` | Rust, C, Java, TS | None — explicit |
| Optional (newline-terminated) | `let x = 5` | Go, Kotlin, Swift | Ambiguity with multi-line expressions |
| Expression-based (last expr is return) | `{ a + b }` returns value | Rust | Slightly unfamiliar but powerful |

**Decision: Mandatory semicolons for statements; last expression in a block is the return value (no semicolon)**

Example:
```
fn add(a: Int, b: Int) -> Int {
    let sum = a + b;   // statement — semicolon required
    sum                 // return expression — no semicolon
}
```

Rationale: This is exactly Rust's model. LLMs handle it well. The distinction between "statement (;)" and "expression (no ;)" is a powerful signal.

### Decision 6: Whitespace

**Decision: Completely insignificant — confirmed per design principle #6**

Braces and semicolons determine structure, never indentation. This is non-negotiable — already decided in the plan.

### Decision 7: Comment Syntax

| Feature | Syntax | Familiar from |
|---|---|---|
| Line comment | `//` | C, Rust, JS, Go, Java |
| Block comment | `/* ... */` | C, Rust, JS, Java |
| Doc comment | `/// ...` | Rust |
| Structured doc fields | `/// @param`, `/// @returns`, `/// @error` | JSDoc, JavaDoc (but parsed structurally) |

**Decision: All four. `//`, `/* */`, `///` with structured `@` tags.**

### Decision 8: Mutability

| Option | Syntax | Familiar from |
|---|---|---|
| Immutable by default, explicit `mut` | `let x = 5;` vs `let mut x = 5;` | Rust |
| All bindings immutable (no `mut`) | `let x = 5;` (rebinding via `let x = ...`) | Haskell, Elm |

**Decision: Immutable by default with `let mut` for opt-in mutability**

Rationale: Functional core principle says immutable by default. But `let mut` is pragmatically useful for loops and accumulators inside a function. LLMs know the `let mut` pattern from Rust. This keeps purity at the function interface level while allowing controlled mutability internally.

### Decision 9: Loop Constructs

| Option | Syntax | Notes |
|---|---|---|
| `for` + `while` + `loop` | Familiar from Rust | Full set, LLMs handle well |
| `for...in` only + recursion | More functional | May frustrate LLMs on performance-sensitive code |
| All three (Rust-style) | `for item in list { }`, `while cond { }`, `loop { }` | Maximum familiarity |

**Decision: `for item in iterable { }`, `while condition { }`, `loop { }` — Rust-style**

### Decision 10: Closures / Lambdas

| Option | Syntax | Familiar from |
|---|---|---|
| `\|x, y\| x + y` | Rust closures | Rust |
| `(x, y) => x + y` | Arrow functions | JS, TS |
| `fn(x, y) { x + y }` | Anonymous fn | Go |

**Decision: `|x, y| x + y` (Rust-style closures)**

Rationale: Consistent with `fn` keyword for named functions. `|` pipes as closure delimiters are recognisable from Rust. Arrow functions would create confusion with the `->` return type syntax.

### Decision 11: Unique Function-Level Identity — The `with` Effect Clause

**Problem:** PZero needs a syntactic marker at the function level that is unique to the language, instantly recognisable by LLMs, and semantically meaningful — not just decorative.

**Decision: Replace `effect fn` with `fn ... with <effects>` clause**

Instead of prefixing effectful functions with the `effect` keyword, PZero places a `with` clause **after** the return type, declaring exactly which effects the function uses:

```
// Pure function — no with clause needed
fn add(a: Int, b: Int) -> Int {
    a + b
}

// Single effect
fn read_file(path: String) -> Result<String, IoError> with io {
    fs::read(path)
}

// Multiple effects
fn fetch_and_save(url: String) -> Result<Data, Error> with io, network {
    let data = http::get(url)?;
    fs::write("cache.json", data)?;
    data
}

// Full function header: contracts + effects
fn process_order(order: Order) -> Result<Receipt, Error>
    with io, db
    requires order.items.len() > 0
    ensures result.is_ok()
{
    // ...
}
```

**Why this is uniquely "PZero":**

1. **No other language** puts `with <effect-set>` after the return type in function signatures
2. **Semantically meaningful** — declares WHICH effects, not just "has effects" (more granular than `effect fn`)
3. **The combination** of `with` + `requires`/`ensures` in a function header is unprecedented across all languages
4. **Strong identity signal** — LLMs will learn: "if I see `fn ... with io`, this is PZero"
5. **One keyword for all functions** — `fn` is universal; the optional `with` clause separates pure from effectful

**Grammar:**
```
FunctionDecl ::= DocComment? Contract? 'fn' IDENT '(' Params ')' ('->' Type)? WithClause? RequiresClause? EnsuresClause? Block
WithClause   ::= 'with' IDENT (',' IDENT)*
```

**Keyword changes:**
- `with` is added as a keyword (context-sensitive: only valid in function headers)
- `effect` is removed from function declarations
- `effect` is retained for declaring effect **types** in Phase 5 (e.g., `effect io { ... }`)

**Key signals an LLM picks up from a PZero function:**
- `fn` → "function declaration, familiar"
- `with io, db` → "this is PZero, not Rust — I can see which effects are used"
- `requires ... ensures ...` → "this language has contracts"
- No `with` clause → "guaranteed pure — I can reason deterministically"

### Decision 12: Logical Statements & LLM Line Comparison

**Problem:** LLMs need to identify and compare specific code locations when making changes. Physical line numbers are fragile — they shift with formatting. The user needs an explicit, stable concept of a "line" in PZero code.

**Decision: Semicolons define logical statements; the canonical formatter makes physical lines deterministic**

PZero's mandatory semicolons and canonical formatting solve the line comparison problem through two mechanisms:

#### Mechanism 1: Semicolons as Logical Statement Boundaries

Every `;` terminates exactly one **logical statement**. The compiler assigns each a sequential **statement index** (SI) within its containing scope:

```
fn example() -> Int {        // SI 0 (function declaration, top-level)
    let x = 5;               // SI 0.1 (first statement in function body)
    let y = x + 10;          // SI 0.2
    let z = if x > 3 {       // SI 0.3
        x * 2                 //   (expression — part of SI 0.3)
    } else {
        x + 1                 //   (expression — part of SI 0.3)
    };
    z                         // SI 0.4 (return expression)
}
```

A "logical statement" is:
- Any semicolon-terminated statement (`;`)
- Any top-level item declaration (`fn`, `struct`, `enum`, `test`, etc.)
- The final expression in a block (no `;`)

#### Mechanism 2: Canonical Formatter as Line Stabiliser

`pzero fmt` always produces **deterministic, identical output** for the same AST. This means:
- Each logical statement starts on its own physical line (when formatted)
- Physical line numbers in formatted code ARE stable across edits that don't change logic
- Two developers (or LLMs) always see the same line numbers for the same code

#### How LLMs use this

| Workflow | How it works |
|---|---|
| **Referencing code** | LLM says "change statement at SI 3.2" — unambiguous regardless of formatting |
| **Comparing changes** | `pzero diff` compares at the statement level, not raw text lines |
| **Error reporting** | Errors reference both physical line AND statement index |
| **Context generation** | `pzero context` includes statement indices for precise navigation |

#### Tooling support

- `pzero parse --format json` outputs each AST node with its `span` (physical) AND `statement_index` (logical)
- `pzero fmt` normalises code so physical lines match logical statements
- The IDE can show the statement index alongside the physical line number
- `pzero diff` operates on statement indices, not raw text — immune to formatting changes

**The rule:** In PZero, a "line" is a semicolon-terminated statement or a brace-delimited declaration. The compiler tracks line numbers using both the physical position (for human IDE display) and the logical statement index (for machine comparison). LLMs should reference code by statement index when precision matters.

---

## 1. Week-by-Week Execution Plan

### Week 1: Project Setup & Syntax Finalisation

| Day | Task | Output |
|---|---|---|
| 1 | **Resolve all open decisions above** | Documented decisions in this file (fill in final answers) |
| 1 | **Set up Rust workspace** | `cargo init --name pzero` with workspace layout: `crates/lexer`, `crates/parser`, `crates/ast`, `crates/cli` |
| 2 | **Write the complete token list** | Every keyword, operator, delimiter, literal type documented |
| 2–3 | **Write the formal PEG grammar** | `grammar.peg` covering all constructs: modules, functions, types, expressions, statements, test blocks, contracts, effects |
| 3–4 | **Write 20 example PZero programs** | Cover all syntax features. These become the parser test corpus AND the LLM validation examples |
| 5 | **LLM syntax validation experiment** | Prompt GPT-4/Claude/etc. with 5 example programs → ask them to write 5 more → measure correctness. Adjust syntax if LLMs consistently misunderstand something |

**Deliverables:**
- [ ] All 12 decisions above locked and documented
- [ ] Rust workspace compiles (`cargo check` passes)
- [ ] `grammar.peg` — complete formal grammar
- [ ] `examples/` — 20 example `.p0` files
- [ ] LLM validation results documented (which models, what accuracy)

### Week 2: AST Definition & Lexer

| Day | Task | Output |
|---|---|---|
| 1–2 | **Define AST data structures** | Rust `enum`/`struct` types in `crates/ast/src/lib.rs` covering every grammar node: `Module`, `Function`, `Struct`, `Enum`, `Expr`, `Stmt`, `Type`, `Pattern`, `TestBlock`, `Contract`, `WithClause`, `StatementIndex` |
| 2 | **Define source span types** | `Span { file, start_line, start_col, end_line, end_col }` and `StatementIndex { scope, index }` attached to every AST node for error reporting and LLM line comparison |
| 3–4 | **Implement lexer with `logos`** | Token enum covering: keywords, identifiers, literals (int, float, string with interpolation, bool), operators, delimiters, comments, doc comments |
| 4–5 | **Lexer test suite** | Test every token type. Test edge cases: nested string interpolation, multiline strings, adjacent operators, Unicode identifiers |
| 5 | **Lexer error reporting** | Invalid tokens produce structured errors with position and "did you mean?" suggestions |

**Deliverables:**
- [ ] `crates/ast/` — complete AST type definitions
- [ ] `crates/lexer/` — working lexer, all tokens
- [ ] 50+ lexer tests passing
- [ ] Lexer errors as structured JSON

### Week 3–4: Parser (Core)

| Day | Task | Output |
|---|---|---|
| W3 D1 | **Parser skeleton** | Recursive descent framework: `Parser` struct with `peek()`, `advance()`, `expect()`, `eat()` methods. Error recovery scaffolding. |
| W3 D1–2 | **Parse: literals & expressions** | Int, Float, Bool, String literals. Binary ops with precedence (Pratt parser). Unary ops. Function calls. Field access. Index access. |
| W3 D2–3 | **Parse: statements** | `let` bindings, `let mut`, assignment, expression statements, `return`, blocks. |
| W3 D3–4 | **Parse: control flow** | `if/else`, `match` with patterns (literal, identifier, struct destructure, enum variant, wildcard), `for..in`, `while`, `loop`, `break`, `continue`. |
| W3 D4–5 | **Parse: functions** | `fn` declarations with params, return type, `with` effect clause, body. Closures `|params| expr`. |
| W4 D1 | **Parse: types** | Type annotations: named types, generics (`List<T>`), function types (`fn(Int) -> Int`), `Option<T>`, `Result<T, E>`. |
| W4 D1–2 | **Parse: data types** | `struct` definitions (named fields). `enum` definitions (variants with optional named fields). `type` aliases. |
| W4 D2–3 | **Parse: modules** | `module` declaration. `import` statements (specify exact path and imported names). Module-level items. |
| W4 D3 | **Parse: test blocks** | `test "name" { body }` — first-class test parsing. |
| W4 D3–4 | **Parse: contracts** | `require expr` and `ensure expr` attached to following `fn`. |
| W4 D4 | **Parse: browser_test blocks** | `browser_test "name" { body }` — parsed as a distinct AST node. |
| W4 D5 | **Parse: doc comments** | `///` comments with `@param`, `@returns`, `@error` tags parsed into structured fields on the following item. |

**Deliverables:**
- [ ] `crates/parser/` — complete parser for all grammar constructs
- [ ] All 20 example programs parse successfully
- [ ] 150+ parser tests passing

### Week 5: Pretty-Printer & Error Recovery

| Day | Task | Output |
|---|---|---|
| 1–2 | **Canonical pretty-printer** | AST → source code. One definitive formatting: consistent indentation (4 spaces), brace style, spacing. `pzero fmt` always produces identical output for semantically identical programs. |
| 2 | **Round-trip test** | For every example program: parse → pretty-print → parse again → ASTs are equal. This validates both parser and printer. |
| 3–4 | **Error recovery** | When parsing fails: synchronise to next statement/item boundary. Report the error. Continue parsing the rest of the file. This is critical for LLM workflows — one error shouldn't block all diagnostics. |
| 4–5 | **Intent-aware error messages** | On syntax error, compare the failing token sequence to known patterns using edit distance. Example: `fn foo(x Int)` → "Missing `:` between parameter name and type. Did you mean `fn foo(x: Int)`?" |

**Deliverables:**
- [ ] `pzero fmt` command — canonical formatting
- [ ] Round-trip tests pass for all examples
- [ ] Parser recovers from errors and reports multiple diagnostics per file
- [ ] Intent-aware error messages for 15+ common mistake patterns

### Week 6: Structured Error Output & CLI

| Day | Task | Output |
|---|---|---|
| 1 | **JSON error schema** | Define the exact JSON shape for all diagnostics. Every error has: `code`, `severity`, `file`, `span`, `message`, `expected`, `actual`, `suggestion`, `context`. |
| 1–2 | **Implement JSON error output** | All parser errors emit structured JSON. Add `--format json` and `--format human` flags. Default is human-readable with colour. |
| 2–3 | **CLI implementation** | Using `clap`: `pzero parse <file>` (output AST JSON), `pzero fmt <file>` (format), `pzero check <file>` (stub for Phase 2). |
| 3–4 | **AST JSON serialisation** | Implement `serde::Serialize` for all AST types. `pzero parse --format json` outputs the complete AST. This is used by tooling and for debugging. |
| 4–5 | **Integration tests** | End-to-end: `.p0` file → CLI → expected output (AST / formatted source / error JSON). Test both valid and invalid inputs. |

**Deliverables:**
- [ ] `pzero` CLI binary with `parse`, `fmt` subcommands
- [ ] `--format json` and `--format human` output modes
- [ ] Complete JSON error schema documented
- [ ] 30+ integration tests (CLI end-to-end)

### Week 7: Test Corpus Expansion & Hardening

| Day | Task | Output |
|---|---|---|
| 1–2 | **Expand test corpus to 200+ cases** | Systematic coverage: every grammar rule has positive and negative test cases. Edge cases: deeply nested expressions, long match arms, large modules, empty modules, many parameters. |
| 2–3 | **Fuzz the parser** | Use `cargo-fuzz` or `arbtest` to generate random token streams and ensure the parser never panics. It should always return either a valid AST or structured errors. |
| 3–4 | **Error message review** | Go through every possible parse error path. Ensure every error has a meaningful message and suggestion. No "unexpected token" without context. |
| 4–5 | **Performance baseline** | Benchmark: parse a 10,000-line PZero file. Establish timing baseline. The parser should comfortably handle large files (target: <100ms for 10K lines). |

**Deliverables:**
- [ ] 200+ test cases passing
- [ ] Fuzzer runs for 10 minutes with zero panics
- [ ] Every error path has a meaningful message
- [ ] Performance benchmark documented

### Week 8: LLM Validation & Phase 1 Closeout

| Day | Task | Output |
|---|---|---|
| 1–2 | **LLM generation benchmark** | Prompt 3+ frontier models (GPT-4, Claude, Gemini) with PZero examples. Ask each to generate 20 programs. Measure: parse success rate, common errors, syntax confusion. |
| 2–3 | **Analyse LLM failure modes** | Categorise what LLMs get wrong. If a consistent pattern emerges (e.g., they always forget semicolons), consider whether the syntax should adapt. |
| 3 | **Syntax adjustments (if needed)** | If LLM testing reveals consistent issues, make targeted grammar changes. Update lexer, parser, tests. |
| 4 | **Documentation** | Write `SYNTAX.md` — the complete syntax reference. Machine-readable grammar. Every construct with an example. |
| 5 | **Phase 1 sign-off** | All deliverables verified. Green test suite. Ready for Phase 2. |

**Deliverables:**
- [ ] LLM benchmark results (3+ models, 20 programs each)
- [ ] `SYNTAX.md` — complete syntax reference
- [ ] Phase 1 complete — all criteria met

---

## 2. Rust Project Structure

```
pzero/
├── Cargo.toml                  # Workspace root
├── DEVELOPMENT_PLAN.md
├── PHASE1_PLAN.md
├── SYNTAX.md                   # Written in Week 8
├── grammar.peg                 # Written in Week 1
├── crates/
│   ├── ast/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       └── lib.rs          # AST node types, Span, serde
│   ├── lexer/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       └── lib.rs          # Token enum, Lexer (logos)
│   ├── parser/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs          # Parser entry point
│   │       ├── expr.rs         # Expression parsing
│   │       ├── stmt.rs         # Statement parsing
│   │       ├── types.rs        # Type parsing
│   │       ├── items.rs        # Top-level items (fn, struct, enum, test, etc.)
│   │       └── error.rs        # Error recovery, intent inference
│   ├── diagnostics/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       └── lib.rs          # Structured errors, JSON output, human output
│   └── cli/
│       ├── Cargo.toml
│       └── src/
│           └── main.rs         # clap CLI: parse, fmt, check (stub)
├── examples/                   # .p0 example programs
│   ├── hello.p0
│   ├── fibonacci.p0
│   ├── shapes.p0
│   ├── counter.p0
│   ├── effects.p0
│   ├── contracts.p0
│   └── ... (20 total)
└── tests/
    ├── lexer/                  # Lexer test fixtures
    ├── parser/                 # Parser test fixtures (valid + invalid)
    └── integration/            # CLI end-to-end tests
```

---

## 3. Token List (Complete — For Lexer Implementation)

### Keywords
```
module    import    fn        with      let       mut
if        else      match     for       in        while
loop      break     continue  return    struct    enum
type      test      browser_test
require   ensure    true      false     effect
```

> **Note:** `effect` is reserved for effect type declarations (Phase 5). Function-level effects use `fn ... with <effects>`. The `with` keyword is only valid in function signatures.

### Types (built-in, treated as keywords or predefined identifiers)
```
Int       Float     Bool      String    Option    Result
List      Map       Set       Html      Cmd
```

### Operators
```
+    -    *    /    %              // arithmetic
==   !=   <    >    <=   >=       // comparison
&&   ||   !                       // logical
|>                                // pipe
?                                 // error propagation
=                                 // assignment
->                                // return type
=>                                // match arm
::                                // path separator
.                                 // field access
..                                // range
```

### Delimiters
```
(    )    {    }    [    ]    <    >
,    ;    :    |
```

### Literals
```
42                   // Int
3.14                 // Float
true / false         // Bool
"hello"              // String
"hello ${name}"      // String with interpolation
```

### Comments
```
// line comment
/* block comment */
/// doc comment
/// @param name - description
/// @returns description
/// @error ErrorType - description
```

---

## 4. Diagnostic JSON Schema

Every error from the parser follows this shape:

```json
{
  "diagnostics": [
    {
      "code": "E0001",
      "severity": "error",
      "message": "Expected `:` between parameter name and type",
      "file": "src/main.p0",
      "span": {
        "start": { "line": 5, "column": 12 },
        "end": { "line": 5, "column": 15 }
      },
      "expected": "`:` followed by a type annotation",
      "actual": "`Int` (identifier)",
      "suggestion": {
        "message": "Add `:` before the type",
        "replacement": "x: Int",
        "span": {
          "start": { "line": 5, "column": 10 },
          "end": { "line": 5, "column": 15 }
        }
      },
      "context": {
        "intent": "function_parameter",
        "nearby_valid_syntax": "fn foo(x: Int) -> Int { ... }"
      }
    }
  ]
}
```

Fields:
| Field | Required | Purpose |
|---|---|---|
| `code` | Yes | Stable error code (machine-consumable) |
| `severity` | Yes | `error`, `warning`, `info` |
| `message` | Yes | Human/LLM readable description |
| `file` | Yes | Source file path |
| `span` | Yes | Exact location in source |
| `expected` | Yes | What the parser expected at this point |
| `actual` | Yes | What it found instead |
| `suggestion` | Optional | Concrete fix: replacement text + target span |
| `context.intent` | Optional | What the parser thinks the LLM was trying to do |
| `context.nearby_valid_syntax` | Optional | Example of the correct form |

---

## 5. Success Criteria for Phase 1

All of these must be true before proceeding to Phase 2:

- [ ] All 12 syntax decisions documented and locked
- [ ] `grammar.peg` exists and covers all constructs
- [ ] Lexer tokenises all valid PZero programs
- [ ] Parser produces correct AST for all 20+ example programs
- [ ] Parser recovers from errors (reports multiple errors per file)
- [ ] Every parse error includes a structured suggestion
- [ ] `pzero parse <file>` outputs AST as JSON
- [ ] `pzero fmt <file>` canonically formats source
- [ ] Round-trip (parse → format → parse) produces identical ASTs
- [ ] 200+ tests passing across lexer, parser, integration
- [ ] Fuzzer runs 10 minutes with zero panics
- [ ] LLM benchmark: 3+ models tested, results documented
- [ ] `SYNTAX.md` complete syntax reference exists
- [ ] Parser handles a 10K-line file in under 100ms

---

## 6. Risk Mitigation

| Risk | Impact | Mitigation |
|---|---|---|
| Syntax design takes too long (bikeshedding) | Delays everything | Time-box to Week 1. Lock decisions. Iterate later if LLM testing reveals issues. |
| String interpolation parsing is complex | Lexer complexity | Implement as a separate lexer mode. Test thoroughly. Worst case: defer interpolation to Week 7. |
| Pratt parser for expressions is tricky | Parser correctness | Well-documented algorithm. Test with precedence-heavy expressions. Reference Matklad's blog posts. |
| Error recovery is hard to get right | Poor LLM experience | Start with simple statement-level recovery. Refine based on real error patterns in Week 7–8. |
| LLMs consistently misinterpret a syntax choice | Wasted work | This is why Week 8 exists. Budget time for syntax adjustments based on empirical LLM testing. |

---

## 7. Immediate First Action

**Before anything else, resolve the 10 decisions in Section 0 above.**

Once those are locked, the very first technical task is:

```
cargo init --name pzero
```

Then set up the workspace structure and write the first token enum in `crates/lexer/src/lib.rs`.
