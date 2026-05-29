# Brix Language Specification - Draft v0.1

> **File extension:** `.br`
> **Paradigm:** Systems programming, statically typed, compiled
> **Memory model:** Ownership + borrow checker
> **Backend:** LLVM
> **Compiler:** `brixc` written in Rust

---

## Design Philosophy

Brix takes the safety and performance of Rust and improves on these fronts:

1. **Simpler syntax** - methods inside structs, f-strings, no impl blocks, no macros
2. **Faster compile times** - simpler type system where possible, incremental by default
3. **Better developer experience** - clear error messages, helpful suggestions, minimal boilerplate
4. **Four-layer ownership model** - ownership contracts, explicit moves, owned APIs, inferred lifetimes - no `'a` ever, no `&` ever
5. **No macros** - everything macros do in Rust is a regular function or built-in feature
6. **Unified CLI** - one tool (`brix`) for everything: create, build, run, test, format, manage packages
7. **Built-in formatter** - one official style, no configuration, no debates
8. **Native concurrency** - `task { }` blocks with compile-time data race prevention
9. **Two-tier errors** - `!T` for app code, `Result<T,E>` for typed library errors
10. **No repeated type annotations** - type declared once on the left, never on the right
11. **User-defined effect system** - `[pure]`, `[io]`, `[network]`, `[on_chain]` - extensible, no keyword pollution
12. **Decorators** - `@log`, `@retry`, `@cache` with full lifecycle hooks - no macros needed
13. **Default parameters + named arguments** - API evolution without breaking callers
14. **Operator overloading** - `op +`, `op ==` inside the struct - no trait names to remember
15. **Unlimited pattern matching** - depth + guards - fully expressive
16. **Python-style loops** - `range()`, `enumerate()` - familiar and clean
17. **Solana-ready from day one** - corrected `[on_chain]` effect, linear account types, role types, aliasing prevention via linear + distinct typed accounts

---

## 1. Primitive Types

| Type | Description | Example |
|------|-------------|---------|
| `i8`, `i16`, `i32`, `i64`, `i128` | Signed integers | `let x: i32 = 42;` |
| `u8`, `u16`, `u32`, `u64`, `u128` | Unsigned integers | `let y: u64 = 100;` |
| `f32`, `f64` | Floating point | `let pi: f64 = 3.14;` |
| `bool` | Boolean | `let ok = true;` |
| `char` | Unicode character | `let c = 'A';` |
| `str` | String slice (borrowed) | `let s: str = "hello";` |
| `String` | Owned string | `let s = "hello";` |

Default numeric types when not annotated:
- Integer literals default to `i32`
- Float literals default to `f64`
- No literal suffixes (`42i32` not valid) - annotate the binding instead

---

## 2. Variables

Immutable by default. Use `mut` for mutable bindings. Type inference handles most cases. Type declared once on the left - never repeated on the right.

```brix
// Immutable (default)
let x = 10;
let name = "Brix";

// Mutable
let mut counter = 0;
counter = counter + 1;

// Explicit type annotation - declared once on the left
let score: f64 = 99.5;
let mut items: [i32] = [];
let result: Result<i32, String> = Ok(42);

// Constants - compile-time, global scope allowed
const MAX_SIZE: i32 = 1024;
```

### Uninitialized Variables

Variables can be declared without initialization. The compiler tracks definite assignment - using before assigning is a compile error.

```brix
let result: i32;

if condition {
    result = compute_a();
} else {
    result = compute_b();
}

print(f"{result}");   // fine - compiler proves always assigned

let x: i32;
print(f"{x}");        // ERROR - x may not be initialized
```

### Type Aliases

```brix
type Meters = f64;
type UserId = i64;
type Result<T> = Result<T, AppError>;

let distance: Meters = 100.0;
let id: UserId = 42;
```

---

## 3. Functions

Declared with `fn`. Return type after `->`. Last expression is the return value (no semicolon), or use `return` explicitly. Both are valid.

```brix
// Trailing return
fn add(a: i32, b: i32) -> i32 {
    a + b
}

// Explicit return
fn subtract(a: i32, b: i32) -> i32 {
    return a - b;
}

// No return value
fn greet(name: String) {
    print(f"Hello, {name}!");
}

// Fallible with !T
fn read_config(path: String) -> !Config {
    let content = read_file(path)!;
    parse_config(content)!
}
```

### Default Parameters

Parameters can have default values. Callers can skip them or pass them by name in any order.

```brix
fn greet(name: String, greeting: String = "Hello") {
    print(f"{greeting}, {name}!");
}

greet("Alice");                        // "Hello, Alice!"
greet("Alice", "Hi");                  // "Hi, Alice!"

fn calculate(
    amount: u64,
    rate: f64,
    precision: i32 = 2,
    currency: String = "USD",
) -> f64 {
    (amount as f64 * rate).round(precision)
}

calculate(1000, 0.003);                             // uses both defaults
calculate(1000, 0.003, currency: "EUR");            // skip precision
calculate(1000, 0.003, precision: 4);               // skip currency
calculate(1000, 0.003, precision: 4, currency: "EUR");
```

API evolution - add optional parameters without breaking callers:

```brix
// Original
fn process(data: [i32]) -> [i32] { ... }

// Add optional parameter - existing callers unaffected
fn process(data: [i32], sorted: bool = false) -> [i32] { ... }

process(nums);              // old callers still work
process(nums, sorted: true); // new callers opt in
```

---

## 4. Control Flow

### If / Else

`if` is an expression - it returns a value.

```brix
let status = if score > 90 { "pass" } else { "fail" };

if x > 0 {
    print("positive");
} else if x == 0 {
    print("zero");
} else {
    print("negative");
}
```

### Loops

```brix
// Range - 0 to 9
for i in range(10) {
    print(f"{i}");
}

// Range with start and end - 1 to 10
for i in range(1, 11) {
    print(f"{i}");
}

// Range with step - 0, 2, 4 ... 98
for i in range(0, 100, 2) {
    print(f"{i}");
}

// Over collection
for item in items {
    print(f"{item}");
}

// With index - like Python's enumerate
for i, item in enumerate(items) {
    print(f"{i}: {item}");
}

// While loop
while count < 10 {
    count = count + 1;
}

// Infinite loop
loop {
    if done { break; }
}

// Loop with break value
let result = loop {
    if found { break value; }
};
```

### Match

Pattern matching - must be exhaustive. Supports unlimited nesting depth and match guards.

```brix
// Basic matching
match direction {
    "north" => print("going up"),
    "south" => print("going down"),
    "east" | "west" => print("going sideways"),
    _ => print("unknown"),
}

// Match as expression
let label = match status_code {
    200 => "OK",
    404 => "Not Found",
    500 => "Server Error",
    _   => "Unknown",
};

// Struct destructuring
match point {
    Point { x: 0, y: 0 } => print("origin"),
    Point { x, y: 0 }    => print(f"on x-axis at {x}"),
    Point { x: 0, y }    => print(f"on y-axis at {y}"),
    Point { x, y }       => print(f"at {x}, {y}"),
}

// Unlimited depth
match data {
    Ok(Response {
        body: Body {
            user: User {
                name,
                address: Address { city, .. }
            }
        }
    }) => print(f"{name} lives in {city}"),
    Err(e) => print(f"error: {e}"),
}

// Match guards
match user {
    User { age, name } if age >= 18 => print(f"{name} is an adult"),
    User { age, name } if age < 18  => print(f"{name} is a minor"),
    _ => print("unknown"),
}

// Guards in DSA
match nums[i] {
    x if x > 0 => print(f"positive: {x}"),
    x if x < 0 => print(f"negative: {x}"),
    _           => print("zero"),
}

// Range patterns
match score {
    90..=100 => print("A"),
    80..=89  => print("B"),
    70..=79  => print("C"),
    _        => print("F"),
}

// Tuple patterns
match (x, y) {
    (0, 0) => print("origin"),
    (x, 0) => print(f"x-axis at {x}"),
    (0, y) => print(f"y-axis at {y}"),
    (x, y) => print(f"point at {x}, {y}"),
}

// .. to ignore remaining fields
match user {
    User { name, .. } => print(f"name: {name}"),
}
```

---

## 5. Strings and F-Strings

Brix uses f-strings for interpolation. The `f` prefix signals the string contains expressions. Plain strings have no interpolation.

```brix
let name = "Brix";
let version = 1;

// F-string interpolation
print(f"Welcome to {name} v{version}!");
print(f"2 + 2 = {2 + 2}");
print(f"Length: {name.len()}");
print(f"Upper: {name.to_upper()}");

// Expressions in f-strings
let msg = f"Status: {if ready { "ready" } else { "not ready" }}";

// Multi-line f-string
let report = f"
    Name: {name}
    Version: {version}
";

// Plain string - no interpolation
let plain = "Hello, world!";

// String operations
let length  = name.len();
let upper   = name.to_upper();
let lower   = name.to_lower();
let slice   = name.slice(0, 3);       // "Bri"
let last3   = name.slice(-3);         // "rix"
let ch      = name[0];                // 'B'
let has_ri  = name.contains("ri");   // true
let trimmed = name.trim();
let parts   = name.split(",");
let joined  = ["a", "b", "c"].join(", ");
```

---

## 6. Arrays

Fixed-size and dynamic arrays with bounds checking.

```brix
// Fixed-size array
let nums: [i32; 5] = [1, 2, 3, 4, 5];

// Dynamic array - type declared once on left
let mut items: [i32] = [];
items.push(1);
items.push(2);
items.pop();

// Access
let first  = nums[0];
let length = nums.len();

// Slicing - Python style
let slice  = nums.slice(1, 3);    // [2, 3]
let last2  = nums.slice(-2);      // [4, 5]
let except = nums.slice(0, -1);   // [1, 2, 3, 4]

// Iteration
for num in nums {
    print(f"{num}");
}

for i, num in enumerate(nums) {
    print(f"{i}: {num}");
}

// Array methods
let sorted   = nums.sorted();
let reversed = nums.reversed();
let found    = nums.contains(3);
let index    = nums.index_of(3);         // Option<i32>
let doubled  = nums.map(|x| x * 2);
let evens    = nums.filter(|x| x > 2);
let sum      = nums.reduce(0, |acc, x| acc + x);
```

---

## 7. Structs

Methods live inside the struct - no separate `impl` block.

```brix
struct Point {
    x: f64,
    y: f64,

    fn new(x: f64, y: f64) -> Point {
        Point { x, y }
    }

    fn distance(self, other: borrow Point) -> f64 {
        let dx = self.x - other.x;
        let dy = self.y - other.y;
        (dx * dx + dy * dy).sqrt()
    }

    fn translate(mut self, dx: f64, dy: f64) {
        self.x = self.x + dx;
        self.y = self.y + dy;
    }

    fn to_string(self) -> String {
        f"({self.x}, {self.y})"
    }

    // Operator overloading
    op +(self, other: Point) -> Point {
        Point { x: self.x + other.x, y: self.y + other.y }
    }

    op ==(self, other: Point) -> bool {
        self.x == other.x && self.y == other.y
    }
}

// Type declared once on the left
let mut p: Point = { x: 3.0, y: 4.0 };
let origin = Point::new(0.0, 0.0);
let dist = p.distance(origin);
p.translate(1.0, 1.0);
print(f"Point: {p.to_string()}");

let p2: Point = { x: 1.0, y: 2.0 };
let p3 = p + p2;       // op +
let eq = p == p2;      // op ==
```

---

## 8. Enums

Tagged unions with optional data. Methods and operators live inside enums too.

```brix
enum Direction {
    North,
    South,
    East,
    West,
}

enum Shape {
    Circle(f64),
    Rectangle(f64, f64),
    Triangle(f64, f64, f64),

    fn area(self) -> f64 {
        match self {
            Shape::Circle(r)         => 3.14159 * r * r,
            Shape::Rectangle(w, h)   => w * h,
            Shape::Triangle(a, b, c) => {
                let s = (a + b + c) / 2.0;
                (s * (s - a) * (s - b) * (s - c)).sqrt()
            }
        }
    }

    fn describe(self) -> String {
        match self {
            Shape::Circle(r)         => f"Circle with radius {r}",
            Shape::Rectangle(w, h)   => f"Rectangle {w}x{h}",
            Shape::Triangle(a, b, c) => f"Triangle with sides {a}, {b}, {c}",
        }
    }
}

let s = Shape::Circle(5.0);
print(f"Area: {s.area()}");
print(f"Shape: {s.describe()}");
```

---

## 9. Operator Overloading

Operators are overloaded using `op` syntax inside the struct or enum. No trait names to remember.

```brix
struct Vector {
    x: f64,
    y: f64,

    op +(self, other: Vector) -> Vector {
        Vector { x: self.x + other.x, y: self.y + other.y }
    }

    op -(self, other: Vector) -> Vector {
        Vector { x: self.x - other.x, y: self.y - other.y }
    }

    op *(self, scalar: f64) -> Vector {
        Vector { x: self.x * scalar, y: self.y * scalar }
    }

    op /(self, scalar: f64) -> Vector {
        Vector { x: self.x / scalar, y: self.y / scalar }
    }

    op ==(self, other: Vector) -> bool {
        self.x == other.x && self.y == other.y
    }

    op <(self, other: Vector) -> bool {
        self.magnitude() < other.magnitude()
    }

    op [](self, index: i32) -> f64 {
        if index == 0 { self.x } else { self.y }
    }

    op -() -> Vector {
        Vector { x: -self.x, y: -self.y }
    }

    fn magnitude(self) -> f64 {
        (self.x * self.x + self.y * self.y).sqrt()
    }
}

let v1 = Vector { x: 1.0, y: 2.0 };
let v2 = Vector { x: 3.0, y: 4.0 };

let v3  = v1 + v2;    // op +
let v4  = v1 - v2;    // op -
let v5  = v1 * 2.0;   // op *
let eq  = v1 == v2;   // op ==
let lt  = v1 < v2;    // op <
let x   = v1[0];      // op []
let neg = -v1;        // op -()
```

### Full list of overloadable operators

| Operator | `op` syntax | Meaning |
|----------|-------------|---------|
| `+` | `op +` | Addition |
| `-` | `op -` | Subtraction |
| `*` | `op *` | Multiplication |
| `/` | `op /` | Division |
| `%` | `op %` | Modulo |
| `==` | `op ==` | Equality |
| `!=` | `op !=` | Inequality |
| `<` | `op <` | Less than |
| `>` | `op >` | Greater than |
| `<=` | `op <=` | Less or equal |
| `>=` | `op >=` | Greater or equal |
| `[]` | `op []` | Index read |
| `[]=` | `op []=` | Index assign |
| `-` (unary) | `op -()` | Negation |
| `!` (unary) | `op !()` | Logical not |

---

## 10. Error Handling

Two tiers - `!T` for app code, `Result<T,E>` for typed library errors.

### `!T` - any-error fallible

Use when a function can fail but callers don't need the specific error type.

```brix
fn read_file(path: String) -> !String {
    let f = open(path)!;
    f.read_all()!
}

fn parse_config(path: String) -> !Config {
    let content = read_file(path)!;
    json.parse(content)!
}

fn main() -> !() {
    let config = parse_config("config.json")!;
    print(f"Loaded: {config.name}");
}
```

### `Result<T, E>` - typed errors

Use when callers need to handle specific error cases.

```brix
enum ParseError {
    InvalidFormat(String),
    Overflow,
    Empty,
}

fn parse_int(s: String) -> Result<i32, ParseError> {
    if s.is_empty() {
        return Err(ParseError::Empty);
    }
    // ...
}

fn read_number(input: String) -> Result<i32, ParseError> {
    let parsed = parse_int(input)?;
    Ok(parsed * 2)
}

// try block
fn process() -> Result<String, Error> {
    let result = try {
        let file = open("data.txt")?;
        let content = file.read()?;
        content.parse()?
    };
    result
}

match parse_int(input) {
    Ok(n)                             => print(f"Parsed: {n}"),
    Err(ParseError::Empty)            => print("input was empty"),
    Err(ParseError::Overflow)         => print("number too large"),
    Err(ParseError::InvalidFormat(s)) => print(f"bad format: {s}"),
}
```

### Option - null safety

```brix
fn find(haystack: [i32], needle: i32) -> Option<i32> {
    for i in range(haystack.len()) {
        if haystack[i] == needle {
            return Some(i);
        }
    }
    None
}

match find(nums, 42) {
    Some(index) => print(f"Found at {index}"),
    None        => print("Not found"),
}

let val = maybe.unwrap_or(0);
let val = maybe.unwrap_or_else(|| compute());
```

### When to use which

| | `!T` | `Result<T, E>` |
|--|------|----------------|
| Use for | Application code, scripts | Library code, public APIs |
| Caller handles specific errors | No | Yes |
| Error propagation | `!` | `?` |
| Verbosity | Minimal | Explicit |

---

## 11. Blueprints and Contracts

**Blueprint** - explicit, behavior-based, supports default implementations (like Rust traits).
**Contract** - implicit, structure-based, satisfied automatically (like Go interfaces).

### Blueprint

```brix
blueprint Printable {
    fn to_string(self) -> String;

    fn print(self) {
        print(f"{self.to_string()}");
    }
}

struct Point {
    x: f64,
    y: f64,

    use Printable {
        fn to_string(self) -> String {
            f"({self.x}, {self.y})"
        }
    }
}

fn display(item: Printable) {
    item.print();
}
```

### Contract

```brix
contract Measurable {
    fn len(self) -> i32;
    fn is_empty(self) -> bool;
}

struct Stack {
    items: [i32],

    fn len(self) -> i32 {
        self.items.len()
    }

    fn is_empty(self) -> bool {
        self.items.len() == 0
    }
}

// Stack automatically satisfies Measurable - no declaration needed
fn report(thing: Measurable) {
    print(f"Length: {thing.len()}, Empty: {thing.is_empty()}");
}
```

### When to use which

| | Blueprint | Contract |
|--|-----------|----------|
| Declaration | Explicit (`use Blueprint`) | Implicit |
| Default methods | Yes | No |
| Best for | Core abstractions, library APIs | Duck typing, flexible composition |
| Similar to | Rust traits | Go interfaces |

---

## 12. Generics

```brix
fn first<T>(items: [T]) -> Option<T> {
    if items.len() > 0 { Some(items[0]) } else { None }
}

struct Pair<A, B> {
    first: A,
    second: B,

    fn new(first: A, second: B) -> Pair<A, B> {
        Pair { first, second }
    }

    fn to_string(self) -> String {
        f"({self.first}, {self.second})"
    }
}

// Blueprint constraint
fn largest<T: Comparable>(items: [T]) -> T {
    let mut max = items[0];
    for item in items.slice(1) {
        if item > max { max = item; }
    }
    max
}

// Multiple constraints
fn process<T: Printable + Comparable>(item: T) {
    item.print();
}
```

---

## 13. Ownership Model

Brix has a four-layer ownership system that provides the same safety guarantees as Rust while eliminating lifetime annotations and `&` symbols entirely. There are no `&` or `&mut` symbols anywhere in Brix - not in function signatures, not in return types, not at call sites.

### Layer 1 - Ownership Contracts

Ownership contracts fully replace Rust's `&` and `&mut` symbols.

```brix
owned   // full ownership transferred - caller loses the value
borrow  // read-only borrow - caller keeps ownership
mut     // mutable borrow - exclusive, caller gets it back
share   // shared immutable access - multiple readers allowed
copy    // value is copied - original untouched
```

At the call site - no symbols needed:

```brix
// Rust - caller must add & and &mut explicitly
read(&name);
modify(&mut name);

// Brix - caller just passes the value
read(name);    // compiler sees borrow in signature
modify(name);  // compiler sees mut in signature
```

Function signatures:

```brix
// Read-only borrow
fn read(name: borrow String) {
    print(f"Hello, {name}!");
}

// Mutable borrow
fn fill(data: mut [i32]) {
    data.push(42);
}

// Takes ownership
fn consume(name: owned String) {
    print(f"Hello, {name}!");
}

// Shared read
fn display(data: share [i32]) {
    for item in data {
        print(f"{item}");
    }
}

// Return types - borrow keyword instead of &
fn get_host(config: borrow Config) -> borrow String {
    config.host
}
```

On struct fields:

```brix
struct FileHandle {
    path: owned String,    // struct owns the path
    buffer: borrow [u8],   // struct borrows the buffer
}
```

Compared to Rust:

| Rust | Brix | Meaning |
|------|------|---------|
| `fn f(x: String)` | `fn f(x: owned String)` | Ownership moved |
| `fn f(x: &String)` | `fn f(x: borrow String)` | Read-only borrow |
| `fn f(x: &mut String)` | `fn f(x: mut String)` | Mutable borrow |
| `fn f(x: &'a String)` | `fn f(x: borrow String)` | Lifetime inferred |
| `f(&name)` | `f(name)` | Call site - no & needed |
| `f(&mut name)` | `f(name)` | Call site - no &mut needed |

### Layer 2 - Explicit `return move`

Ownership transfer is explicit at the return site.

```brix
fn extract_host(config: owned Config) -> String {
    return move config.host;
}

// Partial move
fn take_host(conn: owned Connection) -> String {
    return move conn.host;  // port and buffer are dropped
}

// Move at assignment
let name = move user.name;

// Runtime-conditional move - no lifetime question
fn pick(a: owned String, b: owned String, condition: bool) -> String {
    if condition {
        return move a;
    } else {
        return move b;
    }
}
```

### Layer 3 - Owned APIs by Default

80% of lifetime pain comes from API design. When APIs prefer owned values, lifetimes disappear.

```brix
// Pattern 1 - take owned, return owned
fn process(input: String) -> String {
    input.to_upper()
}

// Pattern 2 - own your fields
struct Config {
    host: String,
    port: u16,
}

// Pattern 3 - clone at the boundary, own forever
fn new(host: borrow str) -> Config {
    Config {
        host: host.to_owned(),
    }
}
```

When references are still worth it:

```brix
// Large buffer - don't clone
fn checksum(data: borrow [u8]) -> u32 { ... }

// Zero-copy parsing - performance critical
fn parse_header(input: borrow str) -> Header { ... }
```

### Layer 4 - Inferred Lifetimes

For cases where references are needed, Brix infers lifetimes automatically. You never write `'a`.

```brix
// Single borrow in, single borrow out - inferred
fn first_word(s: borrow str) -> borrow str {
    s.slice(0, s.find(" ").unwrap_or(s.len()))
}

// Struct with borrowed field - inferred
struct Parser {
    input: borrow str,
}
```

### The Four Layers Working Together

| Layer | What it does | Eliminates |
|-------|-------------|-----------|
| Ownership contracts | Readable intent on inputs | `&`/`&mut` symbols |
| `return move` | Explicit transfer on outputs | Implicit move confusion |
| Owned APIs by default | Structural lifetime avoidance | 80% of lifetime questions |
| Inferred lifetimes | Compiler handles the rest | `'a` annotations |

### Core Ownership Rules

```
1. Each value has exactly one owner at any point in time
2. When the owner goes out of scope, the value is dropped
3. Either one mut borrow OR any number of borrow/share - never both simultaneously
4. References must always be valid - no dangling pointers
5. Data races are caught at compile time
6. No & or &mut symbols anywhere - ownership expressed through contracts only
```

---

## 14. Multi-Return Values

```brix
// Plain tuple
fn min_max(nums: [i32]) -> (i32, i32) {
    let mut min = nums[0];
    let mut max = nums[0];
    for n in nums {
        if n < min { min = n; }
        if n > max { max = n; }
    }
    (min, max)
}

let (lo, hi) = min_max([3, 1, 4, 1, 5]);
print(f"min: {lo}, max: {hi}");

// Named returns
fn divide(a: i32, b: i32) -> (result: i32, remainder: i32) {
    (result: a / b, remainder: a % b)
}

let d = divide(10, 3);
print(f"{d.result} remainder {d.remainder}");

// Or destructure directly
let (result, remainder) = divide(10, 3);
```

---

## 15. Closures

```brix
let double     = |x: i32| x * 2;
let add_one    = |x| { x + 1 };
let offset     = 10;
let add_offset = |x| x + offset;   // captures environment

let nums    = [1, 2, 3, 4, 5];
let doubled = nums.map(|x| x * 2);
let evens   = nums.filter(|x| x % 2 == 0);
let sum     = nums.reduce(0, |acc, x| acc + x);

// Multi-line closure
let process = |x| {
    let doubled = x * 2;
    doubled + 1
};
```

---

## 16. Decorators

Decorators wrap functions with cross-cutting behavior without touching the function itself. Defined with `decorator`, applied with `@`.

```brix
decorator log {
    before {
        print(f"calling {fn.name}");
    }
    after {
        print(f"done {fn.name}");
    }
    on_error {
        print(f"{fn.name} failed: {error}");
        propagate;
    }
}

decorator retry(times: i32 = 3, delay: i32 = 1000) {
    on_error {
        if attempts < times {
            sleep(delay);
            retry();
        } else {
            propagate;
        }
    }
}

decorator cache(ttl: i32 = 60) {
    before {
        let cached = cache.get(fn.args);
        if cached { return cached; }
    }
    after {
        cache.set(fn.args, result, ttl);
    }
}

decorator timeout(ms: i32 = 5000) {
    before {
        let timer = Timer::new(ms);
    }
    on_timeout {
        Err(TimeoutError)
    }
}

decorator validate(schema: Schema) {
    before {
        schema.validate(fn.args)!;
    }
}

decorator measure {
    before { let start = time.now(); }
    after  { print(f"{fn.name} took {time.now() - start}ms"); }
}
```

### Applying decorators

```brix
@log
fn process(data: [i32]) -> [i32] { ... }

@retry(times: 5, delay: 2000)
[network] fn fetch_price(token: String) -> !f64 { ... }

// Stacked - bottom to top, outermost first
@measure
@retry(times: 3)
@timeout(ms: 3000)
@validate(UserSchema)
[network] fn create_user(data: UserData) -> !User { ... }
```

### Decorator context

```brix
decorator example {
    before {
        fn.name        // "create_user"
        fn.args        // all arguments as a tuple
        fn.signature   // full type signature
    }
    after {
        result         // the return value
    }
    on_error {
        error          // the error that occurred
        attempts       // how many times tried
        propagate      // re-throw the error
        retry()        // call the function again
    }
}
```

### Stacking order

Decorators apply bottom to top:

```brix
@measure         // outermost - sees total time including retries
@retry(3)        // middle
@timeout(3000)   // innermost - closest to the function

// Execution:
// measure.before -> retry.before -> timeout.before
//   -> fn()
// timeout.after -> retry.after -> measure.after
```

---

## 17. Effect System

Effects declare what side effects a function is allowed to have. Effects are user-defined and extensible - the compiler ships with built-ins and anyone can define new ones in a package. No new compiler keywords needed for new platforms.

### Defining effects

```brix
// Built-in effects - defined in brix-core
effect pure          { }    // no side effects whatsoever - mathematical functions only
effect deterministic { }    // no randomness, no system time, no floats, no syscalls
                            // but CAN mutate state passed to it - unlike pure
effect io      { fn read() -> [u8]; fn write(data: [u8]) -> (); }
effect network { fn connect(url: String) -> Connection; }
effect alloc   { fn allocate(size: i32) -> u64; }
effect panic   { fn panic(msg: String) -> !; }
effect global  { fn read_global() -> u64; fn write_global(val: u64) -> (); }

// Solana account IO - can ONLY interact with Solana accounts
// NOT general IO - cannot touch files, network, or system
effect account_io {
    fn read_account(key: Pubkey) -> AccountData;
    fn write_account(key: Pubkey, data: AccountData) -> ();
}

// on_chain = deterministic + account_io
// Solana programs are NOT pure - they mutate accounts
// They ARE deterministic - same inputs must produce same outputs
// They can ONLY do account IO - no files, no network, no syscalls
effect on_chain: deterministic + account_io { }

// User-defined effects - anyone can define new ones in a package
effect database {
    fn query(sql: String) -> [Row];
    fn execute(sql: String) -> i32;
}

effect cache {
    fn get(key: String) -> Option<String>;
    fn set(key: String, value: String) -> ();
}

// Combine effects
effect web = io + network;
```

### Why `on_chain` is NOT `pure`

This is a critical distinction:

```brix
// pure - cannot mutate ANY external state
// A Solana program CANNOT be pure - it exists to mutate accounts
[pure] fn add(a: i32, b: i32) -> i32 { a + b }    // fine - truly pure

// on_chain = deterministic + account_io
// CAN mutate accounts (that is its job)
// CANNOT be non-deterministic
// CANNOT do file IO, network, or syscalls
[on_chain]
fn transfer(
    from: linear writable From,
    to:   linear writable To,
    amount: u64,
) -> !() {
    from.data.balance = from.data.balance - amount;  // fine - account_io
    to.data.balance   = to.data.balance   + amount;  // fine - account_io
    read_file("x")!;                                  // ERROR - not account_io
    let r = random();                                  // ERROR - not deterministic
    fetch("url")!;                                     // ERROR - not account_io
}
```

### Applying effects

Effects are applied with `[ ]` - distinct from decorators `@` and keywords.

```brix
[pure]             fn calculate(x: i32) -> i32 { x * 2 }
[deterministic]    fn hash(data: [u8]) -> [u8; 32] { ... }
[io]               fn save(path: String) -> !() { ... }
[network]          fn fetch(url: String) -> !String { ... }
[database]         fn get_user(id: i32) -> !User { ... }
[database, cache]  fn get_user_cached(id: i32) -> !User { ... }
[on_chain]         fn process_instruction(accounts: [RawAccount], data: [u8]) -> !() { ... }
[web]              fn fetch_and_save(url: String) -> !() { ... }
```

### Compiler enforcement

```brix
// pure function - cannot call anything with side effects
[pure] fn compute(x: i32) -> i32 {
    let file = read_file("x")!;   // ERROR - io effect not declared
    fetch("url")!;                 // ERROR - network effect not declared
    x * 2                          // fine
}

// on_chain - only deterministic + account_io allowed
[on_chain]
fn process_instruction(accounts: [RawAccount], data: [u8]) -> !() {
    let fee = calculate(1000);    // fine - pure implies deterministic
    read_file("x")!;              // ERROR - io not allowed in on_chain
    fetch("url")!;                // ERROR - network not allowed in on_chain
    let r = random();             // ERROR - not deterministic
}
```

### Effect inference

```brix
// Private - compiler infers [pure]
fn add(a: i32, b: i32) -> i32 { a + b }

// Public - declare explicitly
pub [io] fn save(path: String) -> !() {
    write_file(path, "data")!;
}

// Compiler warns if declared effect doesn't match
pub [pure] fn bad(path: String) -> !String {
    read_file(path)!   // ERROR - io effect used but only pure declared
}
```

### The complete Brix function signature

```brix
// decorators   effects              name            inputs                    output
   @retry(3)   [network]            fn fetch        (url: owned String)    -> !String
   @measure    [database]           fn get_user     (id: i32)              -> !User
   @log        [io, network]        fn sync         (path: borrow String)  -> !()
               [pure]               fn add          (a: i32, b: i32)       -> i32
               [deterministic]      fn hash         (data: [u8])           -> [u8; 32]
               [on_chain]           fn transfer     (from: linear writable From,
                                                     to:   linear writable To,
                                                     amount: u64)          -> !()
```

### Built-in effects reference

| Effect | Meaning | Solana programs |
|--------|---------|----------------|
| `pure` | No side effects - truly mathematical | No - they mutate accounts |
| `deterministic` | Same inputs, same outputs - no randomness | Yes - required |
| `io` | File system, stdin/stdout | No |
| `network` | HTTP, TCP, network calls | No |
| `alloc` | Heap memory allocation | Limited |
| `panic` | Can panic at runtime | No |
| `global` | Reads/writes global state | No |
| `account_io` | Solana account read/write only | Yes - this is their job |
| `on_chain` | `deterministic + account_io` | Yes - the correct model |

---

## 18. Task Blocks - Concurrency

Brix uses `task { }` blocks as its native concurrency primitive.

```brix
// Fire and forget
task {
    process_file("data.csv");
}

// Task with return value
let handle = task {
    fetch_data("https://api.example.com")
};
let result = await handle;

// Named task
task "file-processor" {
    process_file("data.csv");
}

// Parallel tasks
let (a, b) = await join(
    task { fetch_data("url1") },
    task { fetch_data("url2") },
);

// Task with timeout
let result = await task {
    fetch_data("https://api.example.com")
}.timeout(5000);

// Data race prevention - caught at compile time
let mut counter = 0;
task { counter = counter + 1; }  // ERROR - data race
task { counter = counter + 1; }

// Safe alternatives
let counter = Atomic::new(0);
task { counter.increment(); }
task { counter.increment(); }

let (tx, rx) = channel();
task { tx.send(compute()); };
let result = rx.recv();

let data: Mutex<[i32]> = Mutex::new([]);
task {
    let mut lock = data.lock();
    lock.push(42);
};
```

---

## 19. Async / Await

Built-in async runtime - no external crates needed. `await` is postfix for clean chaining.

```brix
async fn fetch_data(url: String) -> !String {
    let response = http.get(url).await!;
    response.text().await!
}

fn main() -> !() {
    // Postfix await - chains read left to right
    let body = http.get("https://api.example.com").await!;
    print(f"Got: {body}");

    let handle = task { fetch_data("https://api.example.com").await };
    let result = await handle;
    print(f"Result: {result}");
}
```

---

## 20. Modules and Imports

```brix
mod math {
    pub fn square(x: i32) -> i32 { x * x }
    pub fn cube(x: i32) -> i32 { x * x * x }
}

use math::square;
let result = square(5);

use std::io;
use std::collections::HashMap;
use std::io::{read, write};
```

---

## 21. Comments

```brix
// Single-line comment

/*
   Multi-line comment
*/

/// Documentation comment - appears in generated docs
/// This function adds two numbers.
fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

---

## 22. Keywords (Reserved)

```
let         mut         const       fn          return
if          else        match       for         while
loop        break       continue    in
struct      enum        mod         use         pub
self        async       await       task        spawn
blueprint   contract    effect      decorator
try         Option      Some        None
Result      Ok          Err
true        false       as          type
owned       borrow      share       copy        move
op          pure
```

---

## 23. Operator Precedence (highest to lowest)

| Precedence | Operators | Description |
|------------|-----------|-------------|
| 1 | `()` `[]` `.` `::` | Grouping, indexing, field access, path |
| 2 | `!` `-` `*` | Unary: not, negate, deref |
| 3 | `*` `/` `%` | Multiplicative |
| 4 | `+` `-` | Additive |
| 5 | `<<` `>>` | Bit shifts |
| 6 | `<` `>` `<=` `>=` | Comparison |
| 7 | `==` `!=` | Equality |
| 8 | `&` | Bitwise AND |
| 9 | `^` | Bitwise XOR |
| 10 | `\|` | Bitwise OR |
| 11 | `&&` | Logical AND |
| 12 | `\|\|` | Logical OR |
| 13 | `?` `!` | Error propagation |
| 14 | `=` `+=` `-=` `*=` `/=` | Assignment |

---

## 24. Example Programs

### Hello World

```brix
fn main() {
    print("Hello, World!");
}
```

### FizzBuzz

```brix
fn main() {
    for i in range(1, 101) {
        let result = match (i % 3, i % 5) {
            (0, 0) => "FizzBuzz",
            (0, _) => "Fizz",
            (_, 0) => "Buzz",
            _      => f"{i}",
        };
        print(result);
    }
}
```

### Two Sum

```brix
fn two_sum(nums: [i32], target: i32) -> Option<(i32, i32)> {
    for i in range(nums.len()) {
        for j in range(i + 1, nums.len()) {
            if nums[i] + nums[j] == target {
                return Some((i, j));
            }
        }
    }
    None
}

fn main() {
    let nums = [2, 7, 11, 15];
    match two_sum(nums, 9) {
        Some((i, j)) => print(f"Found: [{i}, {j}]"),
        None         => print("No solution"),
    }
}
```

### Linked List

```brix
enum Node {
    Cons(i32, Box<Node>),
    Nil,

    fn push(self, value: i32) -> Node {
        Node::Cons(value, Box::new(self))
    }

    fn to_string(self) -> String {
        match self {
            Node::Cons(val, next) => f"{val} -> {next.to_string()}",
            Node::Nil             => "Nil",
        }
    }
}

fn main() {
    let list = Node::Nil;
    let list = list.push(3);
    let list = list.push(2);
    let list = list.push(1);
    print(list.to_string());  // 1 -> 2 -> 3 -> Nil
}
```

### Stack

```brix
struct Stack {
    items: [i32],

    fn new() -> Stack {
        Stack { items: [] }
    }

    fn push(mut self, value: i32) {
        self.items.push(value);
    }

    fn pop(mut self) -> Option<i32> {
        self.items.pop()
    }

    fn peek(self) -> Option<i32> {
        if self.items.len() > 0 {
            Some(self.items[self.items.len() - 1])
        } else {
            None
        }
    }

    fn is_empty(self) -> bool {
        self.items.len() == 0
    }
}

fn main() {
    let mut s = Stack::new();
    s.push(10);
    s.push(20);
    s.push(30);

    match s.pop() {
        Some(val) => print(f"Popped: {val}"),
        None      => print("Empty"),
    }

    print(f"Top: {s.peek().unwrap_or(0)}");
}
```

### Valid Parentheses

```brix
fn is_valid(input: String) -> bool {
    let mut stack: [char] = [];

    for ch in input {
        match ch {
            '(' | '[' | '{' => stack.push(ch),
            ')' => { if stack.pop() != Some('(') { return false; } }
            ']' => { if stack.pop() != Some('[') { return false; } }
            '}' => { if stack.pop() != Some('{') { return false; } }
            _   => {}
        }
    }

    stack.is_empty()
}

fn main() {
    print(f"{is_valid("([]{})") }");  // true
    print(f"{is_valid("([)]") }");    // false
}
```

### Concurrent File Processing

```brix
[io] fn process_file(path: String) -> !String {
    let content = read_file(path)!;
    content.to_upper()
}

fn main() -> !() {
    let (a, b, c) = await join(
        task { process_file("a.txt") },
        task { process_file("b.txt") },
        task { process_file("c.txt") },
    );

    print(f"A: {a}");
    print(f"B: {b}");
    print(f"C: {c}");
}
```

### Solana On-Chain Program

```brix
use brix_solana::{ RawAccount, Pubkey, TokenData };

// Entry point - raw runtime array from Solana runtime
[on_chain]
fn process_instruction(
    accounts: [RawAccount],
    data: [u8],
) -> !() {
    // Bind accounts to typed roles - signer/writable verified here at runtime
    // but guaranteed to happen - you cannot skip this
    let (payer, from, to) = accounts.bind::<(
        linear signer   Payer,
        linear writable From,
        linear writable To,
    )>()?;

    // Runtime checks that must stay runtime
    // Validated wrapper - compiler ensures these run before data access
    let from = from.require_owner(TOKEN_PROGRAM_ID)?;
    let to   = to.require_owner(TOKEN_PROGRAM_ID)?;

    // Parse instruction data
    let amount: u64 = data.parse()?;

    // Business logic - now statically safe
    transfer(payer, from, to, amount)?;
}

// Typed business logic function
// linear prevents same account being passed twice
// From and To are distinct types - wrong order is a compile error
fn transfer(
    payer: linear signer   Payer,
    from:  linear writable ValidatedFrom,
    to:    linear writable ValidatedTo,
    amount: u64,
) -> !() {
    if from.data.balance < amount {
        return Err(InsufficientFunds);
    }
    from.data.balance = from.data.balance - amount;
    to.data.balance   = to.data.balance   + amount;
}
```

---

## 25. Solana Account Model

Brix has a purpose-built account model for writing Solana on-chain programs. This section documents what Brix can and cannot guarantee at compile time, and how the type system helps catch real bugs.

### What is and is not possible at compile time

Solana account validation is inherently runtime-dependent. The compiler sees types - it does not see the Solana ledger.

**Cannot be compile-time - must stay runtime:**
- Account owner - which program owns this account is only known at runtime
- PDA seeds - whether seeds produce the right address depends on runtime inputs
- Signer bit - whether an account signed is a transaction-level runtime fact
- Rent exemption - lamport balance is runtime state
- Data layout correctness - whether bytes decode correctly is runtime data
- Account aliasing identity - whether two pointers refer to the same account is runtime

**Can be compile-time - what Brix enforces statically:**
- Same account cannot be passed twice to a function (linear types)
- Accounts cannot be used in the wrong role (role types)
- Validation cannot be skipped - Validated wrapper enforced by type system
- On-chain code cannot do IO, network, or randomness (effect system)
- Data cannot be accessed before owner check (Validated type)
- Account mutability is tracked - readonly accounts cannot be mutated

### Linear Account Types

`linear` is a special ownership concept for Solana accounts. A linear value moves on use - it cannot be copied, cloned, or passed twice. This prevents account aliasing at compile time.

```brix
fn transfer(
    from: linear writable From,
    to:   linear writable To,
) -> !() { ... }

let from = accounts[1];
let to   = accounts[2];

transfer(from, to);    // fine - each moved once
transfer(from, from);  // ERROR - from already moved - compile error
                       // catches double-spend bug statically
```

### Role Types

Each account gets a distinct type encoding its role. Even if two accounts hold the same data type, their role types are different - preventing wrong-order bugs.

```brix
struct Payer { lamports: u64 }
struct From  { data: TokenData }
struct To    { data: TokenData }

// From and To are different types even though both hold TokenData
fn transfer(
    payer: linear signer   Payer,
    from:  linear writable From,
    to:    linear writable To,
) -> !() { ... }

transfer(payer, from, to);    // fine
transfer(payer, to, from);    // ERROR - From expected, got To - compile error
                               // catches wrong account order statically
```

### Account Binding at Entry Point

Accounts arrive as a flat runtime array. `bind()` assigns roles and verifies basic constraints (signer, writable) at runtime - but the compiler guarantees this step cannot be skipped.

```brix
[on_chain]
fn process_instruction(
    accounts: [RawAccount],
    data: [u8],
) -> !() {

    // bind() - runtime verification but structurally required
    // returns typed linear accounts or propagates error
    let (payer, from, to) = accounts.bind::<(
        linear signer   Payer,     // signer bit checked here at runtime
        linear writable From,      // writable bit checked here at runtime
        linear writable To,        // writable bit checked here at runtime
    )>()?;

    // After bind() - statically safe to proceed
    transfer(payer, from, to)?;
}
```

### Validated Wrapper - Mandatory Owner Checks

Before accessing account data, the owner must be verified. The `Validated` wrapper type makes this structurally mandatory - the compiler prevents data access on unvalidated accounts.

```brix
// After bind() - account is Unvalidated
// You cannot access .data on an Unvalidated account
let from: Unvalidated<From> = ...;
from.data.balance = ...;  // ERROR - cannot access data on Unvalidated account

// After require_owner() - account becomes Validated
// Only now can you access .data
let from: Validated<From> = from.require_owner(TOKEN_PROGRAM_ID)?;
from.data.balance = ...;  // fine - owner was verified

// The full pattern
let from = from.require_owner(TOKEN_PROGRAM_ID)?;  // runtime - but required
let to   = to.require_owner(TOKEN_PROGRAM_ID)?;    // runtime - but required
transfer(payer, from, to)?;
```

### The Full Pattern

```brix
[on_chain]
fn process_instruction(
    accounts: [RawAccount],
    data: [u8],
) -> !() {

    // Step 1 - bind to typed roles (runtime, but structurally required)
    let (payer, from, to) = accounts.bind::<(
        linear signer   Payer,
        linear writable From,
        linear writable To,
    )>()?;

    // Step 2 - validate ownership (runtime, but structurally required)
    let from = from.require_owner(TOKEN_PROGRAM_ID)?;
    let to   = to.require_owner(TOKEN_PROGRAM_ID)?;

    // Step 3 - parse instruction data
    let amount: u64 = data.parse()?;

    // Step 4 - business logic (statically safe from here)
    transfer(payer, from, to, amount)?;
}

fn transfer(
    payer: linear signer   Payer,
    from:  linear writable Validated<From>,
    to:    linear writable Validated<To>,
    amount: u64,
) -> !() {
    if from.data.balance < amount {
        return Err(InsufficientFunds);
    }
    from.data.balance = from.data.balance - amount;
    to.data.balance   = to.data.balance   + amount;
}
```

### What Brix Catches vs What Rust + Anchor Catches

| Bug class | Rust + Anchor | Brix |
|-----------|--------------|------|
| Same account passed twice | Runtime panic | Compile error (linear types) |
| Wrong account order | Silent corruption | Compile error (role types) |
| Forgot signer check | Runtime exploit | bind() structurally required |
| Forgot writable check | Runtime exploit | bind() structurally required |
| Accessing data before owner check | Silent bug | Compile error (Validated wrapper) |
| IO in on-chain code | Runtime failure | Compile error (`[on_chain]` effect) |
| Randomness in on-chain code | Runtime failure | Compile error (`[deterministic]`) |
| Reentrancy via CPI | Runtime exploit | Linear types prevent mutable aliasing |

### What Brix Cannot Catch

Being honest about what remains runtime:

- PDA seed correctness - depends on runtime inputs
- Rent exemption - lamport balance is runtime state
- Account data layout beyond the declared struct - runtime data
- Cross-program account ownership chains - runtime ledger state
- Any check that requires reading the Solana ledger - runtime

### Honest Positioning

Brix does not eliminate runtime validation for Solana. It eliminates the class of bugs that come from:
- Forgetting to validate
- Validating in the wrong order
- Accidentally aliasing accounts
- Using accounts in the wrong role
- Writing non-deterministic on-chain code

The remaining runtime checks are made structurally mandatory - you cannot skip them because the type system will not let you access account data without going through the validation wrapper.

---

## 26. Unified CLI

```bash
brix new my-project       # create a new project
brix new --lib my-lib     # create a library
brix build                # compile
brix build --release      # compile with optimizations
brix run                  # compile and run
brix run -- arg1 arg2     # run with arguments
brix check                # type check without compiling
brix test                 # run all tests
brix test lexer           # run tests matching "lexer"
brix fmt                  # format all .br files
brix fmt --check          # check formatting only
brix add http             # add a dependency
brix add http@1.2.0       # add a specific version
brix remove http          # remove a dependency
brix update               # update all dependencies
brix doc                  # generate documentation
brix doc --open           # generate and open in browser
brix version              # print brix version
brix upgrade              # upgrade brix itself
```

### brix.toml

```toml
[project]
name = "my-project"
version = "0.1.0"
author = "Your Name"
license = "MIT OR Apache-2.0"

[dependencies]
http = "1.2.0"
json = "0.8.1"
```

---

## 27. Built-in Formatter

`brix fmt` - one official style, non-configurable.

| Rule | Value |
|------|-------|
| Indentation | 4 spaces |
| Max line length | 100 characters |
| Opening brace | Same line |
| Trailing commas | Always in multi-line |
| Blank lines between functions | 1 |
| Spaces around operators | Yes |

```brix
// Before
fn add(a:i32,b:i32)->i32{a+b}

// After brix fmt
fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

---

## 28. What Brix Removes from Rust

| Rust feature | Brix approach |
|-------------|--------------|
| Separate `impl` blocks | Methods live inside struct/enum |
| `format!()`, `println!()` macros | F-strings `f"Hello, {name}!"` |
| All macros (`!` syntax) | Regular functions and built-in features |
| Lifetime annotations (`'a`) | Four-layer ownership model - never written |
| `&` and `&mut` symbols | Fully replaced by ownership contracts - zero `&` in Brix |
| Implicit moves | Explicit `return move` - visible in code |
| Borrow-heavy APIs | Owned APIs by default - lifetimes structurally avoided |
| `pub(crate)`, `pub(super)` | Just `pub` or private |
| Turbofish `::<>` | Not needed - type inference handles it |
| `dyn Trait` vs `impl Trait` | Simplified - compiler decides |
| Trait system for operators | `op +`, `op ==` inside the struct |
| Separate `cargo` tool | Unified `brix` CLI |
| External formatter (`rustfmt`) | Built-in `brix fmt` |
| Manual multi-return via structs | Native multi-return and named returns |
| Repeated type annotations | Type declared once on the left |
| No default parameters | Default params + named arguments |
| External async runtime (Tokio) | Built-in async runtime |
| `tokio::spawn()` | `task { }` blocks |
| Runtime data races | Compile-time data race prevention |
| Single error handling style | `!T` for app code, `Result<T,E>` for libraries |
| `String::from("...")` | String literals are owned by default |
| No effect system | User-defined effects `[pure]`, `[io]`, `[network]` |
| No decorator system | `@decorator` with full lifecycle hooks |
| No on-chain safety | `[on_chain]` = `deterministic + account_io` - semantically correct |
| `on_chain: pure` confusion | Solana programs mutate accounts - `pure` was wrong - fixed to `deterministic + account_io` |
| Account aliasing (runtime panic) | Linear types - same account twice is a compile error |
| Wrong account order (silent bug) | Role types - From and To are distinct types |
| Skippable validation (runtime exploit) | Validated wrapper - compiler enforces validation before data access |
| `for i in 0..n` range syntax | Python-style `range(n)`, `enumerate()` |
| No cross-cutting concern tools | `@log`, `@retry`, `@cache`, `@measure` |
| Uninitialized variables undefined | Compiler tracks definite assignment |
| `s[0..3]` slice syntax | `.slice(0, 3)`, `.slice(-3)` Python-style |
| Integer literal suffixes `42i32` | Type on the left - no suffixes needed |

---

*Brix Language Spec - Draft v0.1*
*The Brix Foundation - May 2026*