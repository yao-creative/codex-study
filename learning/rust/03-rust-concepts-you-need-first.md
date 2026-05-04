# 03 - Core Rust Concepts You Need First

This chapter expands the core Rust concepts with concrete examples, especially for Python/FastAPI engineers.

## 3.1 Ownership and Borrowing

Core rule:
- one owner per value
- you can borrow immutably many times (`&T`) OR mutably once (`&mut T`)

Python style (implicit sharing/mutation):

```python
def add_tag(tags: list[str], tag: str) -> None:
    tags.append(tag)

items = ["a", "b"]
add_tag(items, "c")
```

Rust style (explicit mutability):

```rust
fn add_tag(tags: &mut Vec<String>, tag: String) {
    tags.push(tag);
}

fn demo() {
    let mut items = vec!["a".to_string(), "b".to_string()];
    add_tag(&mut items, "c".to_string());
}
```

Why it matters:
- mutation sites are explicit
- aliasing + mutation bugs are reduced by type system checks

## 3.2 Lifetimes (practical view)

Lifetimes tell the compiler how long references remain valid.

Most of the time, lifetimes are inferred:

```rust
fn first_char(s: &str) -> Option<char> {
    s.chars().next()
}
```

You need explicit lifetimes when returning references tied to input references:

```rust
fn longer<'a>(a: &'a str, b: &'a str) -> &'a str {
    if a.len() >= b.len() { a } else { b }
}
```

Python analogy:
- Python reference lifetimes are managed at runtime.
- Rust reference lifetimes are validated at compile time.

## 3.3 Struct, Enum, and Pattern Matching

Use `struct` for data records and `enum` for states/variants.

```rust
#[derive(Debug)]
struct User {
    id: u64,
    email: String,
}

#[derive(Debug)]
enum AuthState {
    Anonymous,
    Authenticated(User),
    Suspended { reason: String },
}

fn describe(state: AuthState) -> String {
    match state {
        AuthState::Anonymous => "guest".to_string(),
        AuthState::Authenticated(user) => format!("user:{}", user.email),
        AuthState::Suspended { reason } => format!("suspended:{reason}"),
    }
}
```

Why this matters:
- `match` is exhaustive, so missing cases are compile errors
- avoids stringly-typed state handling

## 3.4 Traits and Generics

Trait = behavior contract (similar to interface/protocol conceptually).

```rust
trait UserRepo {
    fn get_email(&self, id: u64) -> Option<String>;
}

struct InMemoryRepo;

impl UserRepo for InMemoryRepo {
    fn get_email(&self, id: u64) -> Option<String> {
        Some(format!("user{id}@example.com"))
    }
}

fn load_user_email<R: UserRepo>(repo: &R, id: u64) -> Option<String> {
    repo.get_email(id)
}
```

Why this matters:
- testability via trait-based boundaries
- compile-time polymorphism with low runtime cost

## 3.5 Error Handling

Rust does not rely on exceptions for ordinary control flow.

- `Option<T>` for absence
- `Result<T, E>` for recoverable failure
- `?` to propagate errors

```rust
#[derive(Debug)]
enum ParseAgeError {
    Empty,
    NotANumber,
}

fn parse_age(input: &str) -> Result<u32, ParseAgeError> {
    if input.trim().is_empty() {
        return Err(ParseAgeError::Empty);
    }
    input
        .parse::<u32>()
        .map_err(|_| ParseAgeError::NotANumber)
}

fn age_bucket(input: &str) -> Result<&'static str, ParseAgeError> {
    let age = parse_age(input)?;
    Ok(if age >= 18 { "adult" } else { "minor" })
}
```

Python analogy:
- Python: `try/except` at runtime.
- Rust: error shape becomes part of function signature.

## 3.6 Concurrency + Async

Use `tokio` for async runtime in backend services.

```rust
use tokio::time::{sleep, Duration};

async fn fetch_profile(user_id: u64) -> String {
    sleep(Duration::from_millis(20)).await;
    format!("profile-{user_id}")
}

#[tokio::main]
async fn main() {
    let a = tokio::spawn(fetch_profile(1));
    let b = tokio::spawn(fetch_profile(2));

    let a = a.await.expect("task 1 failed");
    let b = b.await.expect("task 2 failed");
    println!("{a}, {b}");
}
```

For shared mutable state across tasks:

```rust
use std::sync::Arc;
use tokio::sync::Mutex;

#[derive(Default)]
struct Counter {
    n: u64,
}

async fn inc(counter: Arc<Mutex<Counter>>) {
    let mut guard = counter.lock().await;
    guard.n += 1;
}
```

Why this matters:
- concurrency model is explicit
- thread-safety constraints are checked by compiler

## 3.7 Summary: Concept -> What Changes in Daily Coding

- Ownership/borrowing: explicit data flow and mutation boundaries
- Lifetimes: reference validity checked before runtime
- Struct/enum/match: domain modeling becomes type-first
- Traits/generics: composable and testable service boundaries
- Result/Option: explicit failure/absence handling
- Async/tokio: explicit runtime and task orchestration
