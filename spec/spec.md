# Brix Language Specification - Draft v0.1

> **File extension:** `.br`
> **Paradigm:** Systems programming, statically typed, compiled
> **Memory model:** Ownership + borrow checker
> **Backend:** LLVM

---

## Design Philosophy

Brix takes the safety and performance of Rust and improves on these fronts:

1. **Simpler syntax** - less verbosity, methods live inside structs, built-in string interpolation
2. **Faster compile times** - simpler type system where possible, incremental by default
3. **Better developer experience** - clear error messages, helpful suggestions, minimal boilerplate
4. **Inferred lifetimes** - the compiler figures out lifetimes automatically; annotate only when genuinely ambiguous
5. **No macros** - everything macros do in Rust is expressible as regular functions or built-in language features
6. **Unified CLI** - one tool (`brix`) for everything: create, build, run, test, format, and manage packages
7. **Built-in formatter** - one official style, no configuration, no debates

---

## 1. Primitive Types

| Type | Description | Example |
|------|-------------|---------|
| `i8`, `i16`, `i32`, `i64`, `i128` | Signed integers | `let x: i32 = 42;` |
| `u8`, `u16`, `u32`, `u64`, `u128` | Unsigned integers | `let y: u64 = 100;` |
| `f32`, `f64` | Floating point | `let pi: f64 = 3.14;` |
| `bool` | Boolean | `let ok = true;` |
| `char` | Unicode character | `let c = 'A';` |
| `str` | String slice (borrowed) | `let s: &str = "hello";` |
| `String` | Owned string | `let s = String::from("hello");` |

---

## 2. Variables

Immutable by default. Use `mut` for mutable bindings. Type inference handles most cases.

```brix
// Immutable (default)
let x = 10;
let name = "Brix";

// Mutable
let mut counter = 0;
counter = counter + 1;

// Explicit type annotation (when needed)
let score: f64 = 99.5;

// Constants (compile-time, global scope allowed)
const MAX_SIZE: i32 = 1024;
```

---

## 3. Functions

Declared with `fn`. Return type after `->`. Last expression is the return value (no semicolon), or use `return` explicitly. Both styles are valid.

```brix
// Basic function
fn add(a: i32, b: i32) -> i32 {
    a + b
}

// Explicit return (also valid)
fn subtract(a: i32, b: i32) -> i32 {
    return a - b;
}

// No return value
fn greet(name: String) {
    print("Hello, ${name}!");
}

// Multiple parameters with type inference on return
fn max(a: i32, b: i32) -> i32 {
    if a > b { a } else { b }
}
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
// Infinite loop
loop {
    if done {
        break;
    }
}

// While loop
while count < 10 {
    count = count + 1;
}

// For loop (range)
for i in 0..10 {
    print("${i}");
}

// For loop (inclusive range)
for i in 0..=10 {
    print("${i}");
}

// For loop (over collection)
for item in items {
    print("${item}");
}

// Loop with break value
let result = loop {
    if found {
        break value;
    }
};
```

### Match

Pattern matching - must be exhaustive.

```brix
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
    _ => "Unknown",
};

// Match with destructuring
match point {
    Point { x: 0, y: 0 } => print("origin"),
    Point { x, y: 0 } => print("on x-axis at ${x}"),
    Point { x: 0, y } => print("on y-axis at ${y}"),
    Point { x, y } => print("at ${x}, ${y}"),
}
```

---

## 5. Strings & Interpolation

Brix has built-in string interpolation using `${}` syntax. No format macros needed.

```brix
let name = "Brix";
let version = 1;

// String interpolation
print("Welcome to ${name} v${version}!");

// Expressions inside interpolation
print("2 + 2 = ${2 + 2}");
print("Length: ${name.len()}");

// Multi-line strings
let text = "
    This is a
    multi-line string
";

// String operations
let greeting = "Hello, " + name;
let length = name.len();
let upper = name.to_upper();
let slice = name[0..3];          // "Bri"
let ch = name[0];                // 'B'
let contains = name.contains("ri"); // true
```

---

## 6. Arrays

Fixed-size and dynamic arrays with bounds checking.

```brix
// Fixed-size array
let nums: [i32; 5] = [1, 2, 3, 4, 5];

// Dynamic array (Vec equivalent)
let mut items: [i32] = [1, 2, 3];
items.push(4);
items.pop();

// Access
let first = nums[0];
let length = nums.len();

// Slicing
let slice = nums[1..3];  // [2, 3]

// Iteration
for num in nums {
    print("${num}");
}

// Array methods
let sorted = nums.sorted();
let reversed = nums.reversed();
let found = nums.contains(3);
let index = nums.index_of(3);     // Option<i32>
let mapped = nums.map(fn(x) { x * 2 });
let filtered = nums.filter(fn(x) { x > 2 });
```

---

## 7. Structs

Methods live **inside** the struct - no separate `impl` block.

```brix
struct Point {
    x: f64,
    y: f64,

    // Constructor (associated function)
    fn new(x: f64, y: f64) -> Point {
        Point { x, y }
    }

    // Method (takes self)
    fn distance(&self, other: &Point) -> f64 {
        let dx = self.x - other.x;
        let dy = self.y - other.y;
        (dx * dx + dy * dy).sqrt()
    }

    // Mutable method
    fn translate(&mut self, dx: f64, dy: f64) {
        self.x = self.x + dx;
        self.y = self.y + dy;
    }

    // Method that consumes self
    fn into_tuple(self) -> (f64, f64) {
        (self.x, self.y)
    }
}

// Usage
let mut p = Point::new(3.0, 4.0);
let origin = Point::new(0.0, 0.0);
let dist = p.distance(&origin);
p.translate(1.0, 1.0);
```

---

## 8. Enums

Tagged unions with optional data. Pattern matching works with enums.

```brix
// Simple enum
enum Direction {
    North,
    South,
    East,
    West,
}

// Enum with data
enum Shape {
    Circle(f64),
    Rectangle(f64, f64),
    Triangle(f64, f64, f64),

    // Methods inside enum too
    fn area(&self) -> f64 {
        match self {
            Shape::Circle(r) => 3.14159 * r * r,
            Shape::Rectangle(w, h) => w * h,
            Shape::Triangle(a, b, c) => {
                let s = (a + b + c) / 2.0;
                (s * (s - a) * (s - b) * (s - c)).sqrt()
            }
        }
    }
}

// Usage
let s = Shape::Circle(5.0);
print("Area: ${s.area()}");
```

---

## 9. Option & Result

Built-in types for null safety and error handling. Simpler syntax with `?` operator and `try` blocks.

```brix
// Option - replaces null
enum Option<T> {
    Some(T),
    None,
}

// Result - for error handling
enum Result<T, E> {
    Ok(T),
    Err(E),
}

// Using Option
fn find(haystack: [i32], needle: i32) -> Option<i32> {
    for i in 0..haystack.len() {
        if haystack[i] == needle {
            return Some(i);
        }
    }
    None
}

// Using the ? operator (propagates errors up)
fn read_number(input: String) -> Result<i32, String> {
    let parsed = input.parse_i32()?;  // returns Err early if parse fails
    Ok(parsed * 2)
}

// Try block - catch errors in a scope
fn process() -> Result<String, Error> {
    let result = try {
        let file = open("data.txt")?;
        let content = file.read()?;
        let parsed = content.parse()?;
        parsed
    };
    result
}

// Pattern matching on Option/Result
match find(nums, 42) {
    Some(index) => print("Found at ${index}"),
    None => print("Not found"),
}

// Unwrap shortcuts
let val = maybe_value.unwrap();           // panics if None
let val = maybe_value.unwrap_or(0);       // default if None
let val = maybe_value.unwrap_or_else(fn() { compute() });
```

---

## 10. Blueprints & Contracts

Brix introduces two concepts for polymorphism - **Blueprints** and **Contracts**.

A **Blueprint** defines shared behavior that types must implement (like Rust traits).
A **Contract** defines a structural interface that types must satisfy (like Go interfaces).

### Blueprints (behavior-based, explicit)

Types explicitly declare they follow a blueprint. Blueprints can have default implementations.

```brix
blueprint Printable {
    fn to_string(&self) -> String;

    // Default implementation
    fn print(&self) {
        print("${self.to_string()}");
    }
}

struct Point {
    x: f64,
    y: f64,

    // Implement the blueprint inside the struct
    use Printable {
        fn to_string(&self) -> String {
            "(${self.x}, ${self.y})"
        }
        // print() is inherited from default
    }
}

// Use as a constraint
fn display(item: &Printable) {
    item.print();
}
```

### Contracts (structure-based, implicit)

If a type has the right methods, it satisfies the contract automatically - no explicit declaration needed. Like Go interfaces.

```brix
contract Measurable {
    fn len(&self) -> i32;
    fn is_empty(&self) -> bool;
}

// Any struct that has len() and is_empty() satisfies Measurable
struct Stack {
    items: [i32],

    fn len(&self) -> i32 {
        self.items.len()
    }

    fn is_empty(&self) -> bool {
        self.items.len() == 0
    }
}

// Stack automatically satisfies Measurable - no declaration needed
fn report(thing: &Measurable) {
    print("Length: ${thing.len()}, Empty: ${thing.is_empty()}");
}
```

### When to use which?

| | Blueprint | Contract |
|--|-----------|----------|
| Declaration | Explicit (`use Blueprint`) | Implicit (just have the methods) |
| Default methods | Yes | No |
| Best for | Core abstractions, library APIs | Duck typing, flexible composition |
| Similar to | Rust traits | Go interfaces |

---

## 11. Generics

```brix
// Generic function
fn first<T>(items: [T]) -> Option<T> {
    if items.len() > 0 {
        Some(items[0])
    } else {
        None
    }
}

// Generic struct
struct Pair<A, B> {
    first: A,
    second: B,

    fn new(first: A, second: B) -> Pair<A, B> {
        Pair { first, second }
    }
}

// Generic with blueprint constraint
fn largest<T: Comparable>(items: [T]) -> T {
    let mut max = items[0];
    for item in items[1..] {
        if item > max {
            max = item;
        }
    }
    max
}

// Multiple constraints
fn process<T: Printable + Comparable>(item: T) {
    item.print();
}
```

---

## 12. Ownership & Borrowing

Same rules as Rust - this is what makes Brix memory-safe without a GC.

```brix
// Ownership - each value has one owner
let s1 = String::from("hello");
let s2 = s1;          // s1 is moved, no longer valid
// print("${s1}");    // ERROR: s1 was moved

// Borrowing - read-only reference
fn length(s: &String) -> i32 {
    s.len()
}

let s = String::from("hello");
let len = length(&s);  // borrow s, don't move it
print("${s}");         // s is still valid

// Mutable borrowing - only one at a time
fn push_char(s: &mut String, c: char) {
    s.push(c);
}

let mut s = String::from("hello");
push_char(&mut s, '!');

// Rules:
// 1. Each value has exactly one owner
// 2. You can have EITHER one &mut OR any number of & at a time
// 3. References must always be valid (no dangling pointers)
```

---

## 13. Tuples

```brix
let pair = (1, "hello");
let (num, text) = pair;       // destructuring

// Access by index
let first = pair.0;
let second = pair.1;

// Function returning tuple
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
```

---

## 14. Comments

```brix
// Single-line comment

/* 
   Multi-line
   comment 
*/

/// Documentation comment (for generating docs)
/// This function adds two numbers.
fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

---

## 15. Modules & Imports

```brix
// Declaring a module
mod math {
    fn square(x: i32) -> i32 {
        x * x
    }

    fn cube(x: i32) -> i32 {
        x * x * x
    }
}

// Using a module
use math::square;
let result = square(5);

// Or use the full path
let result = math::cube(3);

// Importing from standard library
use std::io;
use std::collections::HashMap;

// Importing multiple items
use std::io::{read, write};

// Public visibility (private by default)
pub fn public_function() {
    // visible outside the module
}
```

---

## 16. Closures

```brix
// Basic closure
let double = fn(x: i32) -> i32 { x * 2 };
let result = double(5);  // 10

// Closure with type inference
let add_one = fn(x) { x + 1 };

// Closures capture environment
let offset = 10;
let add_offset = fn(x) { x + offset };

// Passing closures to functions
let nums = [1, 2, 3, 4, 5];
let doubled = nums.map(fn(x) { x * 2 });
let evens = nums.filter(fn(x) { x % 2 == 0 });

// Short closure syntax for single expressions
let doubled = nums.map(|x| x * 2);
let evens = nums.filter(|x| x % 2 == 0);
```

---

## 17. Keywords (Reserved)

```
let         mut         const       fn          return
if          else        match       for         while
loop        break       continue    in          
struct      enum        mod         use         pub
self        &           &mut        
blueprint   contract    
try         Option      Some        None
Result      Ok          Err
true        false
as          type        where
```

---

## 18. Operator Precedence (highest to lowest)

| Precedence | Operators | Description |
|------------|-----------|-------------|
| 1 | `()` `[]` `.` `::` | Grouping, indexing, field access, path |
| 2 | `!` `-` `&` `&mut` `*` | Unary: not, negate, reference, deref |
| 3 | `*` `/` `%` | Multiplication, division, modulo |
| 4 | `+` `-` | Addition, subtraction |
| 5 | `<<` `>>` | Bit shifts |
| 6 | `<` `>` `<=` `>=` | Comparison |
| 7 | `==` `!=` | Equality |
| 8 | `&` | Bitwise AND |
| 9 | `^` | Bitwise XOR |
| 10 | `\|` | Bitwise OR |
| 11 | `&&` | Logical AND |
| 12 | `\|\|` | Logical OR |
| 13 | `?` | Error propagation |
| 14 | `=` `+=` `-=` `*=` `/=` | Assignment |

---

## 19. Example Programs

### Hello World

```brix
fn main() {
    print("Hello, World!");
}
```

### FizzBuzz

```brix
fn main() {
    for i in 1..=100 {
        let result = match (i % 3, i % 5) {
            (0, 0) => "FizzBuzz",
            (0, _) => "Fizz",
            (_, 0) => "Buzz",
            _ => "${i}",
        };
        print(result);
    }
}
```

### Two Sum

```brix
fn two_sum(nums: [i32], target: i32) -> Option<(i32, i32)> {
    for i in 0..nums.len() {
        for j in (i + 1)..nums.len() {
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
        Some((i, j)) => print("Found: [${i}, ${j}]"),
        None => print("No solution"),
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

    fn to_string(&self) -> String {
        match self {
            Node::Cons(val, next) => "${val} -> ${next.to_string()}",
            Node::Nil => "Nil",
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

    fn push(&mut self, value: i32) {
        self.items.push(value);
    }

    fn pop(&mut self) -> Option<i32> {
        self.items.pop()
    }

    fn peek(&self) -> Option<i32> {
        if self.items.len() > 0 {
            Some(self.items[self.items.len() - 1])
        } else {
            None
        }
    }

    fn is_empty(&self) -> bool {
        self.items.len() == 0
    }
}

fn main() {
    let mut s = Stack::new();
    s.push(10);
    s.push(20);
    s.push(30);

    match s.pop() {
        Some(val) => print("Popped: ${val}"),
        None => print("Empty"),
    }

    print("Top: ${s.peek().unwrap_or(0)}");
}
```

### Valid Parentheses

```brix
fn is_valid(input: String) -> bool {
    let mut stack: [char] = [];

    for ch in input {
        match ch {
            '(' | '[' | '{' => stack.push(ch),
            ')' => {
                if stack.pop() != Some('(') { return false; }
            }
            ']' => {
                if stack.pop() != Some('[') { return false; }
            }
            '}' => {
                if stack.pop() != Some('{') { return false; }
            }
            _ => {}
        }
    }

    stack.is_empty()
}

fn main() {
    print("${is_valid(\"([]{})\") }");   // true
    print("${is_valid(\"([)]\") }");     // false
}
```

---

## 20. Multi-Return Values

Functions can return multiple values directly without wrapping them in a struct. Both plain tuples and named returns are supported.

### Plain tuple return

```brix
fn min_max(nums: [i32]) -> (i32, i32) {
    let mut min = nums[0];
    let mut max = nums[0];
    for n in nums {
        if n < min { min = n; }
        if n > max { max = n; }
    }
    (min, max)
}

fn main() {
    let (lo, hi) = min_max([3, 1, 4, 1, 5]);
    print("min: ${lo}, max: ${hi}");
}
```

### Named return values

Named returns make the meaning of each value explicit without defining a struct.

```brix
fn divide(a: i32, b: i32) -> (result: i32, remainder: i32) {
    (result: a / b, remainder: a % b)
}

fn main() {
    let (result, remainder) = divide(10, 3);

    // Or access by name
    let d = divide(10, 3);
    print("${d.result} remainder ${d.remainder}");
}
```

Named returns are preferred when the values have meaningful names. Plain tuples are fine for quick, obvious pairs.

---

## 21. Inferred Lifetimes

Brix handles lifetimes automatically in the vast majority of cases. Unlike Rust, you never write `'a` unless the compiler genuinely cannot determine the lifetime relationship itself.

```brix
// Brix - no lifetime annotations needed
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() { x } else { y }
}
```

```rust
// Rust equivalent - requires manual lifetime annotation
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

The compiler infers lifetimes by analyzing the relationship between inputs and outputs. Annotation is only required when the situation is genuinely ambiguous and the compiler cannot make a safe assumption.

---

## 22. No Macros

Brix has no macro system. Everything that requires macros in Rust is handled by regular language features in Brix.

| Rust macro | Brix equivalent |
|------------|----------------|
| `println!("hello {}", name)` | `print("hello ${name}")` - string interpolation |
| `format!("v{}", version)` | `"v${version}"` - string interpolation |
| `vec![1, 2, 3]` | `[1, 2, 3]` - array literal |
| `assert!(condition)` | `assert(condition)` - built-in function |
| `assert_eq!(a, b)` | `assert_eq(a, b)` - built-in function |
| `panic!("msg")` | `panic("msg")` - built-in function |
| `todo!()` | `todo()` - built-in function |
| `unreachable!()` | `unreachable()` - built-in function |
| `include_str!("file")` | `@include_str("file")` - compiler directive |
| `derive(Debug)` | `derive Debug` - built-in, no macro |

The goal is simple: if you see a function call in Brix, it behaves like a function call. No special syntax, no hidden code generation, no separate macro language to learn.

---

## 23. Unified CLI

Brix ships with a single CLI tool that handles everything - compiling, running, testing, formatting, and package management. No separate tools, no configuration files for basic tasks.

```bash
# Project management
brix new my-project          # create a new project
brix new --lib my-lib        # create a new library

# Building and running
brix build                   # compile the project
brix build --release         # compile with optimizations
brix run                     # compile and run
brix run -- arg1 arg2        # compile and run with arguments
brix check                   # type check without compiling (fast)

# Testing
brix test                    # run all tests
brix test lexer              # run tests matching "lexer"

# Formatting
brix fmt                     # format all .br files in the project
brix fmt --check             # check formatting without changing files

# Package management
brix add http                # add a dependency
brix add http@1.2.0          # add a specific version
brix remove http             # remove a dependency
brix update                  # update all dependencies

# Documentation
brix doc                     # generate documentation
brix doc --open              # generate and open in browser

# Toolchain
brix version                 # print brix version
brix upgrade                 # upgrade brix to the latest version
```

### Project structure created by `brix new`

```
my-project/
├── brix.toml          # project manifest (name, version, dependencies)
├── .gitignore
├── README.md
└── src/
    └── main.br        # entry point
```

### `brix.toml` format

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

## 24. Built-in Formatter

`brix fmt` is the official Brix formatter - built into the CLI, opinionated, and non-configurable by design. One way to format Brix code. No `.prettierrc`, no debates.

### Formatting rules

| Rule | Value |
|------|-------|
| Indentation | 4 spaces |
| Max line length | 100 characters |
| Opening brace | Same line |
| Trailing commas | Always in multi-line |
| Blank lines between functions | 1 |
| Blank lines between struct methods | 1 |
| Spaces around operators | Yes |
| Space after keywords | Yes (`if (` → `if (`) |

### Example - unformatted

```brix
fn add(a:i32,b:i32)->i32{
a+b
}
struct Point{x:f64,y:f64,fn new(x:f64,y:f64)->Point{Point{x,y}}}
```

### Example - after `brix fmt`

```brix
fn add(a: i32, b: i32) -> i32 {
    a + b
}

struct Point {
    x: f64,
    y: f64,

    fn new(x: f64, y: f64) -> Point {
        Point { x, y }
    }
}
```

Like Go's `gofmt` - the formatter is the authority. There is no argument about style in a Brix codebase.

---

## 25. No Repeated Type Annotations

If a binding has a type annotation on the left side, the right side never repeats it. The annotation on the left is the single source of truth.

```brix
// Rust - type repeated twice
let mut items: Vec<i32> = Vec::new();
let parser: Box<dyn Parser> = Box::new(MyParser::new());

// Brix - type declared once on the left, right side just provides the value
let mut items: [i32] = [];
let parser: Parser = MyParser::new();
```

This applies to all bindings:

```brix
// Collections
let mut scores: [i32] = [];
let mut map: HashMap<str, i32> = {};

// Struct construction - no need to repeat the type name on the right
let p: Point = { x: 1.0, y: 2.0 };

// Generics - type already declared on the left
let result: Result<i32, String> = Ok(42);
let maybe: Option<i32> = Some(10);
```

The rule: **write the type once, on the left. Never repeat it on the right.**

---

## 26. Async / Await with Built-in Runtime

Brix has `async/await` built into the language with a built-in runtime. No external crates, no choosing between Tokio and async-std, no runtime configuration - it just works.

### Basic usage

```brix
async fn fetch_data(url: String) -> Result<String, Error> {
    let response = await http.get(url)?;
    await response.text()
}

fn main() {
    let data = await fetch_data("https://api.example.com/data");
    print("${data}");
}
```

### Spawning concurrent tasks

```brix
// Spawn a task and run it concurrently
let handle = spawn(async {
    await do_something();
});

await handle;
```

### Running tasks in parallel

```brix
// Both tasks run at the same time, result arrives when both complete
let (a, b) = await join(
    fetch_data("https://api.example.com/a"),
    fetch_data("https://api.example.com/b"),
);
```

### Timeout built in

```brix
// No external crate needed for timeouts
let result = await fetch_data("https://api.example.com").timeout(5000);

match result {
    Ok(data) => print("${data}"),
    Err(TimeoutError) => print("request timed out"),
    Err(e) => print("error: ${e}"),
}
```

### Postfix await for chaining

`await` is postfix - it goes after the expression, making chains readable left to right:

```brix
// Postfix - reads naturally as a pipeline
let body = http.get(url).await?.json().await?;

// vs prefix (harder to read with chaining)
let body = await (await http.get(url))?.json()?;
```

### What the built-in runtime handles

- Task scheduling and execution
- Async I/O (network, files, timers)
- Spawning and joining concurrent tasks
- Timeouts and cancellation
- Thread pool management

### Comparison with Rust

| | Rust | Brix |
|--|------|------|
| Runtime | External (Tokio, async-std) | Built-in |
| Setup | `#[tokio::main]`, Cargo deps | Nothing - just use `async/await` |
| `await` position | Postfix (`.await`) | Postfix (`await expr`) |
| Spawn | `tokio::spawn()` | `spawn()` built-in |
| Join | `tokio::join!()` macro | `join()` built-in function |
| Timeout | External crate | Built-in `.timeout()` method |

---

## 28. Task Blocks - Brix Concurrency

Brix uses `task` blocks as its native concurrency primitive. Lightweight, runtime-scheduled, and unique to Brix - not borrowed from any other language.

### Fire and forget

```brix
task {
    process_file("data.csv");
}
```

### Task that returns a value

```brix
let handle = task {
    fetch_data("https://api.example.com")
};

let result = await handle;
```

### Named tasks (for debugging and tracing)

```brix
task "file-processor" {
    process_file("data.csv");
}
```

### Running multiple tasks in parallel

```brix
let (a, b) = await join(
    task { fetch_data("url1") },
    task { fetch_data("url2") },
);
```

### Task with timeout

```brix
let handle = task {
    fetch_data("https://api.example.com")
};

let result = await handle.timeout(5000);
```

### Comparison

| | Rust | Go | Brix |
|--|------|----|------|
| Concurrency primitive | `tokio::spawn()` | `go func()` | `task { }` |
| Returns a value | `JoinHandle<T>` | Channel | `let handle = task { }` |
| Built-in runtime | No | Yes | Yes |
| Named tasks | No | No | Yes |

---

## 29. Compile-Time Data Race Prevention

Brix prevents data races at compile time. If two `task` blocks can access the same mutable memory simultaneously, the compiler rejects it - no runtime crashes, no undefined behavior.

```brix
let mut counter = 0;

task { counter = counter + 1; }  // ERROR: data race - mutable access across tasks
task { counter = counter + 1; }  // compiler catches this at compile time
```

**Correct approaches:**

```brix
// Atomic values - safe concurrent mutation
let counter = Atomic::new(0);
task { counter.increment(); }   // fine
task { counter.increment(); }   // fine

// Channels - communicate by sending values between tasks
let (sender, receiver) = channel();

task {
    sender.send(compute_result());
};

let result = receiver.recv();

// Mutex - explicit locking for shared state
let data = Mutex::new([]);

task {
    let mut lock = data.lock();
    lock.push(42);
};
```

The rule is simple: **shared mutable state across tasks must go through a safe primitive** - `Atomic`, `Mutex`, or channels. The compiler enforces this, not the programmer's discipline.

---

## 30. Error Handling - `!T` and `Result<T, E>`

Brix has two tiers of error handling for different use cases:

### `!T` - any-error fallible (for application code)

Use `!T` when a function can fail but the caller doesn't need to know the specific error type. Errors propagate automatically with `!`. Great for application code, scripts, and glue code.

```brix
// Return type !T means "returns T or any error"
fn read_file(path: String) -> !String {
    let f = open(path)!;      // ! propagates error upward automatically
    f.read_all()!
}

fn parse_config(path: String) -> !Config {
    let content = read_file(path)!;
    let config = json.parse(content)!;
    config
}

// main can be fallible too
fn main() -> !() {
    let config = parse_config("config.json")!;
    print("Loaded: ${config.name}");
}
```

### `Result<T, E>` - typed errors (for library code)

Use `Result<T, E>` when the caller needs to handle specific error cases. Best for libraries where consumers need to know exactly what can go wrong.

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

// Caller handles specific errors
match parse_int(input) {
    Ok(n) => print("Parsed: ${n}"),
    Err(ParseError::Empty) => print("input was empty"),
    Err(ParseError::Overflow) => print("number too large"),
    Err(ParseError::InvalidFormat(s)) => print("bad format: ${s}"),
}
```

### When to use which

| | `!T` | `Result<T, E>` |
|--|------|----------------|
| Use for | Application code, scripts | Library code, public APIs |
| Caller handles specific errors | No | Yes |
| Error propagation | Automatic with `!` | Manual with `?` or `match` |
| Verbosity | Minimal | Explicit |
| Equivalent to | Swift `throws`, Zig `!T` | Rust `Result<T, E>` |

### Mixing both

```brix
// Library exposes typed errors
fn fetch(url: String) -> Result<Response, HttpError> { ... }

// Application code wraps it with !T - doesn't care about specific HttpError
fn load_data() -> !String {
    let response = fetch("https://api.example.com")!;  // HttpError becomes any-error
    response.text()!
}
```

---

## 31. F-Strings - String Interpolation

Brix uses f-strings for string interpolation - familiar to Python and Kotlin developers. The `f` prefix makes it visually explicit that a string contains expressions.

### Basic usage

```brix
let name = "Brix";
let version = 1;

let msg = f"Hello, {name}!";
let full = f"Welcome to {name} v{version}";
```

### Expressions inside f-strings

Any valid Brix expression works inside `{}`:

```brix
let a = 2;
let b = 3;

print(f"Sum: {a + b}");               // Sum: 5
print(f"Upper: {name.to_upper()}");   // Upper: BRIX
print(f"Length: {name.len()}");       // Length: 4
print(f"2 + 2 = {2 + 2}");           // 2 + 2 = 4
```

### F-strings vs plain strings

```brix
// Plain string - no interpolation
let s = "Hello, world!";

// F-string - interpolation enabled
let s = f"Hello, {name}!";

// Multi-line f-string
let msg = f"
    Name: {name}
    Version: {version}
    Status: {if ready { "ready" } else { "not ready" }}
";
```

### Replaces all Rust string macros

| Rust | Brix |
|------|------|
| `format!("Hello, {}!", name)` | `f"Hello, {name}!"` |
| `println!("Hello, {}!", name)` | `print(f"Hello, {name}!")` |
| `format!("2+2={}", 2+2)` | `f"2+2={2+2}"` |
| `format!("{:?}", value)` | `f"{value.debug()}"` |
| `format!("{:.2}", pi)` | `f"{pi:.2}"` |

---

## 32. What Brix Removes from Rust

| Rust feature | Brix approach |
|-------------|--------------|
| Separate `impl` blocks | Methods live inside struct/enum |
| `format!()`, `println!()` macros | F-strings `f"Hello, {name}!"` |
| All macros (`!` syntax) | Regular functions and built-in features |
| Lifetime annotations (`'a`) | Inferred by compiler; annotate only when ambiguous |
| `pub(crate)`, `pub(super)` | Just `pub` (public) or nothing (private) |
| Turbofish `::<>` | Not needed - type inference handles it |
| `dyn Trait` vs `impl Trait` | Simplified - compiler decides boxing |
| Complex trait bounds with `where` | Simplified constraint syntax |
| Separate `cargo` tool | Unified `brix` CLI |
| External formatter (`rustfmt`) | Built-in `brix fmt` |
| Manual multi-return via structs | Native multi-return and named returns |
| Repeated type annotations | Type declared once on the left |
| External async runtime (Tokio) | Built-in async runtime |
| `tokio::spawn()` / goroutines | `task { }` blocks |
| Runtime data races | Compile-time data race prevention |
| Single error handling style | `!T` for app code, `Result<T,E>` for libraries |

---

*Brix Language Spec - Draft v0.1*
*The Brix Foundation - May 2026*
