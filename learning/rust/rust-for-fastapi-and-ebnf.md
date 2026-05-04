# Rust for Python/FastAPI Engineers

This guide is a fast bridge from Python/FastAPI mental models to core Rust.

## 1) Rust in One Page

- Rust is **compiled, statically typed, memory-safe without GC**.
- Safety is enforced at compile-time through:
  - ownership
  - borrowing
  - lifetimes
  - exhaustive pattern matching
- Concurrency defaults are safer: data races are compile-time errors.
- Error handling is explicit (`Result`, `Option`), not exception-driven by default.

If Python/FastAPI optimizes for rapid iteration, Rust optimizes for **correctness + performance + maintainability under scale**.

---

## 2) Mental Model Mapping (Python/FastAPI -> Rust)

- `dict` / `list` dynamic shapes -> `struct` / `enum` / `Vec<T>` / `HashMap<K,V>`
- runtime type errors -> compile-time type errors
- exceptions -> `Result<T, E>` and `?`
- `None` -> `Option<T>` (`Some(T)` / `None`)
- decorators + DI -> traits, generics, explicit composition
- `async def` + event loop -> `async fn` + executor (`tokio`)
- Pydantic models -> `struct` with `serde` derives

---

## 3) Core Rust Concepts You Need First

## 3.1 Ownership and Borrowing

- Every value has one owner.
- Move semantics by default.
- Borrow with references:
  - immutable: `&T`
  - mutable: `&mut T`
- Many immutable borrows OR one mutable borrow at a time.

Why this matters: prevents aliasing + mutation bugs and many thread-safety issues.

## 3.2 Lifetimes (practical view)

- Lifetimes describe how long references are valid.
- Usually inferred by compiler.
- You write explicit lifetimes mainly when returning references tied to input references.

## 3.3 Struct, Enum, and Pattern Matching

- `struct` for product types (like Pydantic models / DTOs).
- `enum` for sum types (states/variants/errors).
- `match` is exhaustive; compiler forces handling all variants.

## 3.4 Traits and Generics

- Trait = interface/behavior contract.
- Generics + trait bounds = reusable zero-cost abstractions.
- `impl Trait` and where-clauses keep APIs clean.

## 3.5 Error Handling

- Recoverable: `Result<T, E>`
- Optional presence: `Option<T>`
- Propagate errors with `?`
- Model domain errors with enums.

## 3.6 Concurrency + Async

- Use `tokio` for async runtime.
- `Send`/`Sync` traits capture thread-safety guarantees.
- Use `Arc<T>` for shared ownership, `Mutex/RwLock` when mutation is needed.

---

## 4) FastAPI-Oriented Rust Stack Overview

Typical API stack:

- HTTP framework: `axum` (or `actix-web`)
- async runtime: `tokio`
- serialization: `serde`, `serde_json`
- DB: `sqlx` or `diesel`
- config: `config`, `dotenvy`
- observability: `tracing`, `tracing-subscriber`
- testing: `cargo test`, integration tests in `tests/`

### Analogy to FastAPI

- FastAPI router -> `axum::Router`
- dependency injection -> explicit state injection (`State<T>`) and layered middleware
- Pydantic request/response models -> `#[derive(Serialize, Deserialize)] struct`
- exception handlers -> `IntoResponse` + error enums
- startup/shutdown events -> runtime init + graceful shutdown hooks

---

## 5) Rust Project Shape (Service)

- `src/main.rs`: process bootstrap, router wiring
- `src/lib.rs`: reusable core logic
- `src/domain/*`: business entities/logic
- `src/adapters/*`: DB, HTTP, queues
- `src/api/*`: handlers and DTOs
- `tests/*`: black-box integration tests

Keep business logic separate from transport layer, similar to clean architecture in Python services.

---

## 6) What Will Feel Different vs Python

- You design types first; coding becomes easier after that.
- Refactors become safer because compiler checks all callsites.
- More up-front friction, much lower runtime ambiguity.
- Performance profiling often focuses on architecture, not interpreter overhead.

---

## 7) Common Rust Patterns (EBNF)

These are simplified, practical EBNF forms (not the full Rust grammar).

## 7.1 Variable and Binding

```ebnf
let_stmt        = "let" ["mut"] ident [":" type_expr] ["=" expr] ";" ;
```

## 7.2 Function (sync/async)

```ebnf
fn_decl         = ["pub"] ["async"] "fn" ident generic_params?
                  "(" param_list? ")" return_type? where_clause? block ;

param_list      = param { "," param } [","] ;
param           = pattern ":" type_expr ;
return_type     = "->" type_expr ;
```

## 7.3 Struct and Enum

```ebnf
struct_decl     = ["pub"] "struct" ident generic_params? struct_body ;
struct_body     = "{" field_list? "}" | "(" tuple_field_list? ")" ";" | ";" ;
field_list      = field { "," field } [","] ;
field           = ["pub"] ident ":" type_expr ;

enum_decl       = ["pub"] "enum" ident generic_params? "{" variant_list "}" ;
variant_list    = variant { "," variant } [","] ;
variant         = ident [ tuple_variant | struct_variant ] ;
tuple_variant   = "(" type_expr { "," type_expr } [","] ")" ;
struct_variant  = "{" field_list? "}" ;
```

## 7.4 Impl Blocks and Methods

```ebnf
impl_block      = "impl" generic_params? type_expr where_clause?
                  "{" impl_item* "}" ;

trait_impl      = "impl" generic_params? trait_path "for" type_expr where_clause?
                  "{" impl_item* "}" ;

impl_item       = fn_decl | assoc_type | assoc_const ;
```

## 7.5 Trait Definition

```ebnf
trait_decl      = ["pub"] "trait" ident generic_params?
                  [":" trait_bounds] where_clause?
                  "{" trait_item* "}" ;

trait_item      = fn_sig ";" | assoc_type_decl ";" | assoc_const_decl ";" ;
```

## 7.6 Match and Patterns

```ebnf
match_expr      = "match" expr "{" match_arm { "," match_arm } [","] "}" ;
match_arm       = pattern ["if" expr] "=>" expr_or_block ;

pattern         = "_"
                | ident
                | literal
                | tuple_pattern
                | struct_pattern
                | enum_pattern
                | ref_pattern ;
```

## 7.7 Result/Option Flow

```ebnf
result_type     = "Result" "<" type_expr "," type_expr ">" ;
option_type     = "Option" "<" type_expr ">" ;

try_expr        = expr "?" ;
```

## 7.8 Control Flow

```ebnf
if_expr         = "if" expr block { "else" "if" expr block } [ "else" block ] ;
while_expr      = "while" expr block ;
for_expr        = "for" pattern "in" expr block ;
loop_expr       = "loop" block ;
```

## 7.9 Macro Invocation

```ebnf
macro_call      = path "!" ("(" token_tree? ")" | "{" token_tree? "}" | "[" token_tree? "]") ;
```

## 7.10 Async + Await

```ebnf
await_expr      = expr ".await" ;
async_block     = "async" ["move"] block ;
```

---

## 8) Recommended Learning Path (FastAPI Background)

1. Learn ownership/borrowing with small CLI examples.
2. Learn `Result`/`Option` + `?` thoroughly.
3. Build one small `axum` CRUD service.
4. Add `serde` DTOs and `sqlx` DB layer.
5. Introduce trait-based service interfaces.
6. Add integration tests and tracing.

---

## 9) First Practical Milestones

- milestone 1: parse JSON request -> validate -> response
- milestone 2: DB access with typed query results
- milestone 3: structured domain errors mapped to HTTP responses
- milestone 4: background task + graceful shutdown
- milestone 5: performance and memory profile baseline

Once you reach milestone 3, Rust starts feeling natural for backend work.
