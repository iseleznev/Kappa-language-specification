# Kappa Language Specification

MIT License

Copyright (c) 2024 Igor Seleznyov

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

---

## Overview

Kappa is a systems programming language combining Kotlin's readability with memory control at the Rust/Vale/Zig level. Key principles:

- Minimal syntactic noise
- Explicit memory control without borrow checker complexity
- Generational references as the primary safety mechanism
- Functional style with full performance control

---

## Primitive Types

### Signed Integers

| Type | Size | Range | Example |
|------|------|-------|---------|
| `tiny` | 8 bit | -128 .. 127 | `val t: tiny = -100` |
| `short` | 16 bit | -32768 .. 32767 | `val s: short = 1000` |
| `int` | 32 bit | -2³¹ .. 2³¹-1 | `val i: int = 42` |
| `long` | 64 bit | -2⁶³ .. 2⁶³-1 | `val l: long = 1_000_000L` |
| `long128` | 128 bit | -2¹²⁷ .. 2¹²⁷-1 | `val big: long128 = ...` |

### Unsigned Integers

| Type | Size | Range | Example |
|------|------|-------|---------|
| `byte` / `utiny` | 8 bit | 0 .. 255 | `val b: byte = 0xFF` |
| `ushort` | 16 bit | 0 .. 65535 | `val us: ushort = 60000` |
| `uint` | 32 bit | 0 .. 2³²-1 | `val ui: uint = 4_000_000_000` |
| `ulong` | 64 bit | 0 .. 2⁶⁴-1 | `val ul: ulong = 10_000_000_000UL` |
| `ulong128` | 128 bit | 0 .. 2¹²⁸-1 | `val huge: ulong128 = ...` |

### Floating Point

| Type | Size | Example |
|------|------|---------|
| `float` | 32 bit | `val f: float = 3.14f` |
| `double` | 64 bit | `val d: double = 3.14159265359` |

### Boolean and Bit Types

| Type | Size | Notes |
|------|------|-------|
| `bit` | 1 bit | Packed with adjacent bits into bytes |
| `bool` | 8 bit | Standard boolean type |
| `bool16` | 16 bit | For alignment with short |
| `bool32` | 32 bit | For alignment with int/float |
| `bool64` | 64 bit | For alignment with long/double |

```kappa
// bit accepts values 0, 1, false, true
var flag: bit = true
var other: bit = 1

// Compiler packs adjacent bits
struct Flags {
    var a: bit    // ┐
    var b: bit    // │ packed into 1 byte
    var c: bit    // │
    var d: bit    // ┘
}
```

### Other

| Type | Size | Example |
|------|------|---------|
| `char` | 32 bit (Unicode) | `val c: char = 'A'` |

---

## Basic Syntax

### Variables

```kappa
val name = "Igor"                   // immutable, type inferred
var count = 0                       // mutable
val age: int = 46                   // explicit type
```

### Functions

```kappa
fn add(a: int, b: int) -> int {
    a + b                           // implicit return
}

fn greet(name: string) {            // Unit returned implicitly
    println("Hello, $name!")
}

// Expression body
fn square(x: int) -> int = x * x

// Multiple return values
fn divide(a: int, b: int) -> (quotient: int, remainder: int) {
    (a / b, a mod b)
}

val (q, r) = divide(17, 5)          // destructuring
```

### Named Arguments

```kappa
fn createUser(name: string, age: int, city: string = "Unknown") -> User {
    User(name, age, city)
}

val user1 = createUser("Igor", 46, "Baku")
val user2 = createUser(name = "Igor", age = 46)
val user3 = createUser(age = 46, name = "Igor")      // order doesn't matter
val user4 = createUser("Igor", city = "Baku", age = 46)
```

---

## Structures

### Basic Declaration

```kappa
struct Point {
    var x: float
    var y: float
    
    fn distanceTo(other: Point) -> float {
        val dx = x - other.x
        val dy = y - other.y
        sqrt(dx * dx + dy * dy)
    }
    
    static fn origin() -> Point {
        Point(0.0, 0.0)
    }
}

val p = Point(3.0, 4.0)
println(p.x)
println(p.distanceTo(Point.origin()))
```

### Constructors

```kappa
// Primary constructor
struct Point(val x: float, val y: float)

// With default values
struct Config(
    val host: string = "localhost",
    val port: int = 8080,
    val timeout: int = 30
)

val c1 = Config()                   // all defaults
val c2 = Config(port = 3000)        // only port

// Constructor body with init
struct User(val name: string, val age: int) {
    var .createdAt: long
    
    init {
        require(age >= 0) { "Age cannot be negative" }
        createdAt = System.currentTimeMillis()
    }
}

// Secondary constructors
struct Point(val x: float, val y: float) {
    constructor(value: float) : this(value, value)
    constructor() : this(0.0, 0.0)
}
```

### Methods Outside Structure

```kappa
fn Point.manhattanDistance(other: Point) -> float {
    abs(x - other.x) + abs(y - other.y)
}

val p = Point(3.0, 4.0)
println(p.manhattanDistance(Point.origin()))
```

---

## Qualifier — Method Refinement

Qualifier is a method signature element that refines its behavior. Unlike annotations, a qualifier is part of the method name.

```kappa
// Syntax: fn Type.method.qualifier(args) -> ReturnType

fn Point.scale.uniform(factor: float) -> Point {
    Point(x * factor, y * factor)
}

fn Point.scale.separate(fx: float, fy: float) -> Point {
    Point(x * fx, y * fy)
}

val p = Point(10.0, 20.0)
val uniform = p.scale.uniform(2.0)          // Point(20.0, 40.0)
val separate = p.scale.separate(2.0, 0.5)   // Point(20.0, 10.0)
```

### Nested Qualifiers

```kappa
fn Atomic[T].cas.weak.relaxed(expected: T, new: T) -> bool
fn Atomic[T].cas.strong.acqrel(expected: T, new: T) -> bool

val counter = Atomic[int](0)
if (counter.cas.strong.acqrel(0, 1)) {
    println("Success")
}
```

### Practical Examples

```kappa
// HTTP client
fn HttpClient.request.get(path: string) -> Response
fn HttpClient.request.post(path: string, body: byte[]) -> Response

fn HttpClient.with.timeout(d: Duration) -> HttpClient
fn HttpClient.with.auth.bearer(token: string) -> HttpClient

val response = client
    .with.timeout(10.seconds)
    .with.auth.bearer(token)
    .request.get("/users")

// Logger
fn Logger.log.debug(msg: string)
fn Logger.log.info(msg: string)
fn Logger.log.error(msg: string)

logger.log.info("User logged in")
```

---

## Field Visibility

### Rules

| Declaration | Visibility |
|-------------|------------|
| `var x: T` | public (if no pub on others) |
| `var .x: T` | private (dot in declaration) |
| `pub var x: T` | public (others without pub become private) |
| `var x: T { } ->` | private (has getter) |
| `pub var x: T { } ->` | public through accessor |

### Examples

```kappa
// All fields public
struct Simple {
    var x: int                      // public
    var y: int                      // public
}

// Private fields via dot
struct User {
    val id: long                    // public
    var .passwordHash: string       // private
    var .failedAttempts: int        // private
}

fn User.authenticate(password: string) -> bool {
    // Inside methods dot is optional
    if (failedAttempts >= 5) {
        return false
    }
    hash(password) == passwordHash
}

// pub mode — others become private
struct Account {
    pub val id: long                // public
    pub var balance: decimal        // public
    var owner: string               // private (no pub)
}

// Properties with getter/setter
struct Temperature {
    pub var celsius: float
    
    pub var fahrenheit: float
        { celsius * 9.0 / 5.0 + 32.0 } ->           // getter
        <- { celsius = (value - 32.0) * 5.0 / 9.0 } // setter (value is keyword)
}
```

---

## References and Ownership

### Memory Model

Primitives are passed by value (copied). All other types (structures, strings, arrays) are passed by reference. By default, reference has type `ref` (generational reference).

```kappa
val account = Account(1, 1000.00, "Igor")   // ref Account
val name = "Hello"                           // ref string
val list = [1, 2, 3]                         // ref int[]
```

### Four Ownership Modes

```kappa
val account = Account(1, 1000.00, "Igor")

// ref — generational reference (default)
val ref r1 = account                // via keyword
val r2 = @account                   // via @ operator

// own — exclusive ownership (move semantics)
val own exclusive = Account(2, 500.00)

// borrow — compile-time check (Rust-style)
val borrow temp = account

// raw — raw pointer (unsafe)
val raw p1 = account                // via keyword
val p2 = &account                   // via & operator
```

### ref: Generational References

Primary safety mechanism. Each object has a generation counter. Reference stores expected generation. On dereference — runtime check for match.

```kappa
fn processRef(acc: Account) {       // Account = ref Account by default
    println(acc.balance)            // if object alive — OK
                                    // if freed — runtime panic
}

val account = Account(1, 1000.00, "Igor")
processRef(account)                 // pass ref
processRef(@account)                // explicitly create ref
```

**Advantages over borrow checker:**
- Easier to write complex structures (graphs, trees)
- No fighting with the compiler
- Minimal overhead (~1-3%)

### own: Exclusive Ownership

```kappa
fn createBuffer() -> own RingBuffer[int, 1024] {
    own RingBuffer[int, 1024]()
}

fn main() {
    var buffer = createBuffer()
    buffer.offer(42)
    
    val buffer2 = buffer            // move — buffer invalidated
    // buffer.offer(1)              // ERROR: buffer moved
    
    processBuffer(buffer2)          // ownership goes to function
}

fn processBuffer(buf: own RingBuffer[int, 1024]) {
    buf.offer(100)
    // At scope end — automatic free
}
```

### raw: Raw Pointers

```kappa
fn processRaw(acc: &Account) {
    unsafe {
        println(acc.balance)
    }
}

val account = Account(1, 1000.00, "Igor")
val raw ptr = account               // via keyword
processRaw(&account)                // via operator
```

### Copying vs Assignment

```kappa
val original = Account(1, 1000.00, "Igor")

val sameRef = original              // = reference to same object
val copy <= original                // <= deep copy

copy.balance = 500.00
println(original.balance)           // 1000.00 — unchanged
```

### Operator Summary

| Syntax | Meaning |
|--------|---------|
| `val a = b` | assignment (ref/move) |
| `val a <= b` | deep copy |
| `val ref a = b` | explicit ref |
| `@b` | create ref |
| `val raw a = b` | explicit raw |
| `&b` | create raw pointer |

---

## Memory Layout in Structures

### Primitives — Inline, Objects — By Reference

```kappa
struct Event {
    val id: ulong                   // offset 0, 8 bytes — value
    val timestamp: long             // offset 8, 8 bytes — value
    val date: ZonedDateTime         // offset 16, 8 bytes — ADDRESS
    val description: string         // offset 24, 8 bytes — ADDRESS
}
// Size: 32 bytes

// Object fields store addresses, not objects themselves!
val event = Event(1UL, 0L, ZonedDateTime.now(), "test")
val dateRef: ulong = event:[16]     // reading ADDRESS, not object
```

### inline — Storing Object Inside Structure

```kappa
struct Vertex {
    inline val position: Vec3       // 12 bytes inline
    inline val normal: Vec3         // 12 bytes inline
    inline val uv: Vec2             // 8 bytes inline
}
// Size: 32 bytes, all in one memory block

struct IpHeader {
    val versionIhl: byte
    val tos: byte
    val totalLength: ushort
    inline val srcAddr: IpAddress   // 4 bytes inline
    inline val dstAddr: IpAddress   // 4 bytes inline
}
```

### Summary: What's Stored in Fields

| Field Type | What's Stored | Size (64-bit) |
|------------|---------------|---------------|
| `tiny`, `byte`, `utiny` | value | 1 byte |
| `short`, `ushort` | value | 2 bytes |
| `int`, `uint`, `float` | value | 4 bytes |
| `long`, `ulong`, `double` | value | 8 bytes |
| `long128`, `ulong128` | value | 16 bytes |
| `bool` | value | 1 byte |
| `char` | value | 4 bytes |
| `string` | address (ref) | 8 bytes |
| `struct` | address (ref) | 8 bytes |
| `array[]` | address (ref) | 8 bytes |
| `inline struct` | value | struct size |

---

## Arrays

Arrays are low-level memory blocks. Collections (List, Map) are high-level structures with methods.

### Declaration

```kappa
// One-dimensional — type[](size)
val buffer = byte[](1024)
val numbers = int[](100)
val dynamic = byte[](sizeFromArg)   // runtime size

// Multi-dimensional — type[](size1, size2, ...)
val matrix = float[](4, 4)
val cube = int[](10, 10, 10)

// Literal with type inference
val days = ["Mon", "Tue", "Wed"]    // string[]
val primes = [2, 3, 5, 7, 11]       // int[]
```

### Element Access

```kappa
val numbers = int[](10)
numbers[0] = 42
val first = numbers[0]

val matrix = float[](4, 4)
matrix[0, 0] = 1.0
matrix[3, 3] = 1.0
val element = matrix[2, 1]
```

### Arrays in Constructors

```kappa
struct Buffer(capacity: int) {
    val .data: byte[]               // size not specified
    
    init {
        data = byte[](capacity)     // allocate in init
    }
}

val buf = Buffer(1024)
```

---

## Raw Byte Access :[offset]

Any reference can be interpreted as a byte array.

```kappa
val fullName = "Igor Seleznyov"
val firstByte = fullName:[0]        // byte

struct Account {
    val id: ulong128                // offset 0, 16 bytes
    val balance: long               // offset 16, 8 bytes
    val owner: string               // offset 24, 8 bytes (ADDRESS!)
}

val account = Account(Uuid.randomUuid7().long128(), 1000L, "Igor")
val balance: long = account:[16]    // read value
val ownerRef: ulong = account:[24]  // read ADDRESS of string, not string!

// Write
var buffer = byte[](100)
buffer:[0] = 0xFF
buffer:[8] = 1234567890L
```

---

## Null Safety

```kappa
var name: string? = null            // nullable
var age: int = 0                    // non-null

val length = name?.length           // int? — safe call
val len = name?.length ?: 0         // int — elvis
val len = name!!.length             // not-null assertion

if (name != null) {
    println(name.length)            // smart cast
}

name?.let { println("Name is $it") }

val city = user?.address?.city ?: "Unknown"
```

---

## Annotations

```kappa
[Deprecated("Use newMethod instead")]
[Inline]
fn oldMethod() { }

[Serializable]
struct User {
    [JsonName("user_id")]
    val id: long
    var name: string
}

fn process(
    [NotEmpty] name: string,
    [Range(0, 100)] percentage: int
) { }
```

---

## Interfaces and Traits

### Interfaces — Primary Contract

```kappa
interface Closeable {
    fn close()
}

interface Reader {
    fn read(buffer: byte[]) -> int
}

interface ReadWriter : Reader, Writer

interface Comparable[T] {
    fn compareTo(other: T) -> int
    fn lessThan(other: T) -> bool = compareTo(other) < 0
}

struct File : Closeable, ReadWriter {
    override fn close() { }
    override fn read(buffer: byte[]) -> int { }
}
```

### Traits — Extensible Behavior

```kappa
trait Hash {
    fn hash() -> ulong
}

trait Eq {
    fn eq(other: Self) -> bool
}

// Implementation for own type
impl Hash for Point {
    fn hash() -> ulong {
        var h = 17UL
        h = h * 31 + x.toBits()
        h * 31 + y.toBits()
    }
}

// Implementation for foreign type
impl Hash for string {
    fn hash() -> ulong { /* ... */ }
}

// Conditional implementation
impl[T: Hash] Hash for List[T] {
    fn hash() -> ulong {
        var h = 17UL
        for (item in this) {
            h = h * 31 + item.hash()
        }
        h
    }
}
```

### Trait Bounds

```kappa
fn sort[T: Ord + Copy](list: List[T]) -> List[T] { }

fn merge[K, V, M](left: M, right: M) -> M
where
    K: Hash + Eq,
    M: Collection[Pair[K, V]] + Growable[Pair[K, V]]
{ }
```

### Sealed Traits

```kappa
sealed trait Result[T, E] {
    struct Ok(value: T)
    struct Err(error: E)
}

fn handle(result: Result[int, string]) -> int {
    return match result {
        Ok(value) -> value
        Err(msg) -> {
            println("Error: $msg")
            -1
        }
    }
}
```

---

## Operator Overloading

### Arithmetic

```kappa
struct Vec2(val x: float, val y: float) {
    operator fn +(other: Vec2) -> Vec2 = Vec2(x + other.x, y + other.y)
    operator fn -(other: Vec2) -> Vec2 = Vec2(x - other.x, y - other.y)
    operator fn *(scalar: float) -> Vec2 = Vec2(x * scalar, y * scalar)
    operator fn unary-() -> Vec2 = Vec2(-x, -y)
}

val a = Vec2(1.0, 2.0)
val b = Vec2(3.0, 4.0)
val sum = a + b                     // Vec2(4.0, 6.0)
val neg = -a                        // Vec2(-1.0, -2.0)
```

### Indexing

```kappa
struct Matrix[T](val rows: int, val cols: int) {
    var .data: T[]
    
    init { data = T[](rows * cols) }
    
    operator fn [](row: int, col: int) -> T {
        data[row * cols + col]
    }
    
    operator fn []=(row: int, col: int, value: T) {
        data[row * cols + col] = value
    }
}

val m = Matrix[float](4, 4)
m[0, 0] = 1.0
val element = m[2, 1]
```

### Call Operator

```kappa
struct Multiplier(val factor: int) {
    operator fn ()(value: int) -> int {
        value * factor
    }
}

val double = Multiplier(2)
val result = double(21)             // 42
```

### in Operator

```kappa
struct Range(val start: int, val end: int) {
    operator fn contains(value: int) -> bool {
        value >= start && value < end
    }
}

if (5 in Range(0, 10)) {
    println("In range")
}
```

### Iteration

```kappa
struct Range(val start: int, val end: int) {
    operator fn iterator() -> RangeIterator {
        RangeIterator(start, end)
    }
}

for (i in Range(0, 10)) {
    println(i)
}
```

### Operator Summary

| Operator | Method |
|----------|--------|
| `a + b` | `operator fn +(other: T)` |
| `a - b` | `operator fn -(other: T)` |
| `a * b` | `operator fn *(other: T)` |
| `a / b` | `operator fn /(other: T)` |
| `a % b` | `operator fn %(other: T)` |
| `-a` | `operator fn unary-()` |
| `!a` | `operator fn unary!()` |
| `a == b` | `operator fn ==(other: T)` |
| `a < b` | `operator fn <(other: T)` |
| `a[i]` | `operator fn [](index: I)` |
| `a[i] = v` | `operator fn []=(index: I, value: V)` |
| `a(x)` | `operator fn ()(arg: T)` |
| `x in a` | `operator fn contains(x: T)` |
| `a += b` | `operator fn +=(other: T)` |
| `for x in a` | `operator fn iterator()` |

---

## Control Flow

### if/when Expressions

```kappa
val max = if (a > b) a else b

val result = when {
    x < 0 -> "negative"
    x == 0 -> "zero"
    else -> "positive"
}

fn classify(x: int) -> string {
    return if (x == 0) "zero" else "non-zero"
}
```

### match

```kappa
fn process(result: Result[int, string]) -> int {
    return match result {
        Ok(value) -> value * 2
        Err(msg) -> {
            println("Error: $msg")
            0
        }
    }
}
```

### Loops

```kappa
// Range — two equivalent forms
for (i in 0 until 10) { }        // 0, 1, 2, ..., 9
for (i in 0..10) { }             // 0, 1, 2, ..., 9 (exclusive end)

// Inclusive end
for (i in 0..=10) { }            // 0, 1, 2, ..., 10

// With step
for (i in 0..100 step 10) { }    // 0, 10, 20, ..., 90

// Reverse
for (i in 10 downTo 0) { }       // 10, 9, 8, ..., 0
for (i in 10..0 step -1) { }     // alternative

// Collection iteration
for (item in list) { }
for ((key, value) in map) { }

// While
while (condition) { }

// Infinite loop
loop {
    if (done) break
}

// Repeat N times
repeat(10) { println("Hello") }
```

#### Range Syntax

| Syntax | Description | Example Values |
|--------|-------------|----------------|
| `0 until n` | [0, n) exclusive | 0, 1, ..., n-1 |
| `0..n` | [0, n) exclusive | 0, 1, ..., n-1 |
| `0..=n` | [0, n] inclusive | 0, 1, ..., n |
| `n downTo 0` | [n, 0] reverse | n, n-1, ..., 0 |
| `a..b step k` | with step k | a, a+k, a+2k, ... |

---

## Pipe and Composition

### Pipe Forward |>

```kappa
// value |> func is equivalent to func(value)

val result = data
    |> filter { it > 0 }
    |> map { it * 2 }
    |> sum
```

### Pipe Backward <|

```kappa
// func <| value is equivalent to func(value)

println <| format <| calculate <| input
```

### Forward Composition >>

```kappa
// (f >> g)(x) is equivalent to g(f(x))

val process = validate >> transform >> save
val result = process(user)
```

### Backward Composition <<

```kappa
// (f << g)(x) is equivalent to f(g(x))

val process = save << transform << validate
```

### Practical Example

```kappa
val parseAndValidate = trim >> parseJson[User] >> validateUser
val enrichAndSave = addTimestamp >> addMetadata >> saveToDb
val fullPipeline = parseAndValidate >> enrichAndSave

rawInput |> fullPipeline
```

---

## Error Handling

### The error Keyword

`error` — like `return`, but returns an error. The compiler automatically adds a hidden parameter for the error if `error` is used in a function. If there's no `error` in the function — no hidden parameter is added (optimization).

```kappa
fn divide(a: int, b: int) -> int {
    if (b == 0) {
        error DivisionError(message = "Cannot divide by zero")
    }
    a / b
}

struct DivisionError {
    val message: string
}

// Or a primitive as error code
fn parsePositive(s: string) -> int {
    val n = s.toInt()
    if (n < 0) {
        error -1
    }
    n
}
```

### catch Syntax

Code after `catch` goes WITHOUT curly braces. Curly braces are only in the error handler.

```kappa
// After variable
val result = divide(10, 2)
catch result { err ->
    println("Error: ${err.message}")
    return
}

// Short form with default
val result = divide(10, 0)
catch result { 0 }

// Inline form
val result = divide(a, b) catch { 0 }

// On method call
file.close() { err -> sout << err.message }
```

### catch Without Expression

If `catch` is called without specifying a variable or expression, it checks the last returned error. In this case, a single-line handler without curly braces is allowed:

```kappa
val file = File.open(path)
val content = file.read.all()
val data = parse(content)
catch err -> logger.error(err.message)  // checks last error

// Equivalent to
catch err -> {
    logger.error(err.message)
}

// Or with multi-line handler
catch { err ->
    logger.error(err.message)
    metrics.increment("errors")
    return Data.empty()
}

// Useful at end of operation block
fn loadData(path: string) -> Data {
    val file = File.open(path)
    val content = file.read.all()
    val parsed = parse(content)
    catch err -> error LoadError("Failed to load: ${err.message}")
    
    transform(parsed)
}

// Or for logging without interruption
fn process() {
    step1()
    step2()
    step3()
    catch err -> logger.warn("Some step failed: $err")  // log but continue
}
```

### catch Before Code Block

```kappa
// catch wraps multiple operations
// Code goes without curly braces, until the handler
catch
val file = File.open.readOnly(path)
val buffer, size = file.read.all()
process(buffer)
{ err -> Data.default() }

// With indentation for readability (indentation is optional)
catch
    val file = File.open.readOnly(path)
    val buffer, size = file.read.all()
    process(buffer)
{ err -> Data.default() }

// Single-line catch
catch val content = file.read.all() { err -> byte[](0) }
```

### Multi-line Handler

```kappa
catch
val file = File.open(path)
val data = file.read.all()
transform(data)
{ err ->
    match err {
        is IoError -> handleIo(err)
        is MemoryError -> handleMemory(err)
        else -> handleUnknown(err)
    }
}
```

### Multiple Return and catch

```kappa
// Method returns multiple values
fn File.read.all() -> (buffer: byte[], size: int)

// catch can be specified for any variable
// Compiler understands it refers to hidden error variable
val buffer, size = file.read.all()
catch buffer { err -> 
    sout << "Read failed: " << err.message
    return
}
// or equivalently
catch size { err -> ... }
```

### Automatic Propagation

```kappa
// If no catch is written — error propagates automatically
fn calculate(a: int, b: int) -> int {
    val x = divide(a, b)            // if error — exit immediately
    val y = divide(x, 2)            // if error — exit immediately
    x + y
}
```

### Nested catch

```kappa
catch
val config = catch
    loadFromFile("config.json")
{ err -> 
    logger.warn("File not found, trying env")
    loadFromEnv()
}
val connection = connect(config)
process(connection)
{ err -> 
    logger.error("Processing failed")
    Result.empty() 
}
```

### Helpers

```kappa
val size = file.size() orDefault 0L

val content = file.read.all() orElse { err ->
    logger.warn("Using cache: $err")
    cache.get(path)
}

// orPanic — for prototypes and tests
val config = loadConfig() orPanic
```

### defer

```kappa
// Single-line — without curly braces
defer file.close()

// Multi-line — with curly braces
defer {
    file.close()
    logger.debug("File closed")
}

// Usage example
fn processFile(path: string) -> Data {
    val file = File.open(path)
    defer file.close()              // executes on both return and error
    
    val content = file.read.all()
    process(content)
}
```

### Standard Errors (CommonError)

All standard errors implement the `CommonError` interface:

```kappa
interface CommonError {
    val code: int
    val message: string
}
```

#### ArithmeticError

```kappa
sealed trait ArithmeticError : CommonError {
    object DivisionByZero : ArithmeticError {
        override val code = 1
        override val message = "Division by zero"
    }
    object IntegerOverflow : ArithmeticError {
        override val code = 2
        override val message = "Integer overflow"
    }
    object IntegerUnderflow : ArithmeticError {
        override val code = 3
        override val message = "Integer underflow"
    }
}
```

#### MemoryError

```kappa
sealed trait MemoryError : CommonError {
    object OutOfMemory : MemoryError {
        override val code = 100
        override val message = "Out of memory"
    }
    object StackOverflow : MemoryError {
        override val code = 101
        override val message = "Stack overflow"
    }
    object NullPointer : MemoryError {
        override val code = 103
        override val message = "Null pointer dereference"
    }
}
```

#### IndexError

```kappa
sealed trait IndexError : CommonError {
    struct OutOfBounds(val index: int, val size: int) : IndexError {
        override val code = 200
        override val message = "Index $index out of bounds for size $size"
    }
}
```

#### IoError

```kappa
sealed trait IoError : CommonError {
    struct NotFound(val path: string) : IoError {
        override val code = 400
        override val message = "File not found: $path"
    }
    struct PermissionDenied(val path: string) : IoError {
        override val code = 401
        override val message = "Permission denied: $path"
    }
    struct Timeout(val operation: string, val duration: Duration) : IoError {
        override val code = 404
        override val message = "$operation timed out after $duration"
    }
}
```

#### ConcurrencyError

```kappa
sealed trait ConcurrencyError : CommonError {
    object Deadlock : ConcurrencyError {
        override val code = 500
        override val message = "Deadlock detected"
    }
    object ThreadInterrupted : ConcurrencyError {
        override val code = 502
        override val message = "Thread interrupted"
    }
}
```

### Automatic Errors

The compiler automatically generates standard errors:

```kappa
// Division by zero
val x = 10 / 0                      // automatically error ArithmeticError.DivisionByZero

// Index out of bounds
val arr = int[](10)
val v = arr[15]                     // automatically error IndexError.OutOfBounds(15, 10)

// Handling
val result = catch 10 / divisor { err ->
    match err {
        is ArithmeticError.DivisionByZero -> 0
        else -> -1
    }
}
```

### panic — For Unrecoverable Situations

```kappa
fn unreachable() -> Nothing {
    panic("Should never reach here")
}

// Catching panic
val result = catch panic {
    dangerousOperation()
}
```

### StackTrace

```kappa
// Explicit capture — only when needed
val trace = StackTrace.capture()
trace.print()

// With depth limit
val shortTrace = StackTrace.capture(maxDepth = 10)

// Addresses only (fast), symbolize later
val addresses = StackTrace.captureAddresses(maxDepth = 50)
val symbolized = addresses.symbolize()
```

### Summary

| Element | Curly Braces |
|---------|--------------|
| Code after `catch` | ❌ No |
| Error handler | ✅ Yes `{ err -> ... }` |
| `defer` single-line | ❌ No |
| `defer` multi-line | ✅ Yes |

| Keyword | Description |
|---------|-------------|
| `error value` | Return error from function |
| `catch x { err -> }` | Handle error after variable |
| `catch ... { err -> }` | Handle code block |
| `expr catch { default }` | Inline handling |
| `method() { err -> }` | Handle on call |
| `catch err -> handler` | Handle last error |
| `orDefault value` | Default on error |
| `orElse { err -> value }` | Compute default |
| `orPanic` | Panic on error |
| `panic(msg)` | Unrecoverable error |
| `defer expr` | Single-line cleanup |
| `defer { }` | Multi-line cleanup |

---

## Atomics and Memory Ordering

### Atomic[T]

```kappa
val counter = Atomic[int](0)

// Load/Store with memory ordering via qualifier
val value = counter.load.relaxed
val value = counter.load.acquire
val value = counter.load.seqcst

counter.store.relaxed(42)
counter.store.release(42)
counter.store.seqcst(42)

// CAS
counter.cas.weak.relaxed(0, 1)
counter.cas.strong.acqrel(0, 1)

// Fetch-and-modify
val old = counter.fetchAdd.acqrel(1)
val old = counter.fetchSub.release(1)
val old = counter.exchange.acqrel(100)
```

### Volatile[T]

```kappa
// Volatile — visibility guarantee between threads, no CAS
// Used instead of volatile keyword

struct SharedState {
    val flag: Volatile[bool] = Volatile(false)
    val counter: Volatile[int] = Volatile(0)
}

val state = SharedState()
state.flag.set(true)
val f = state.flag.get()

// With explicit ordering via qualifier
state.counter.get.relaxed
state.counter.get.acquire
state.counter.set.release(42)

// Volatile vs Atomic:
// Volatile[T] — visibility only (get/set)
// Atomic[T] — visibility + atomic operations (CAS, fetchAdd)
```

### Memory Fences

```kappa
Cpu.fence.acquire()
Cpu.fence.release()
Cpu.fence.acqrel()
Cpu.fence.seqcst()
Cpu.fence.compiler()                // compiler only
```

---

## Threads

### Creation and Management

```kappa
val thread = Thread {
    println("Hello from thread!")
}

val worker = Thread(
    name = "Worker-1",
    stackSize = 2.megabytes,
    priority = ThreadPriority.High
) {
    processQueue()
}

thread.start()
thread.join()

val completed = thread.join(timeout = 5.seconds)
```

### Thread State

```kappa
thread.state                        // ThreadState
thread.isAlive                      // bool
thread.isInterrupted                // bool

thread.interrupt()

Thread.currentThread()
Thread.sleep(100.millis)
Thread.yield()
Thread.availableProcessors()
```

### ThreadLocal

```kappa
val threadLocal = ThreadLocal[Connection]()
threadLocal.set(createConnection())
val conn = threadLocal.get()

// With initializer
val requestId = ThreadLocal { Uuid.randomUuid7() }
```

---

## CPU Primitives

### Cache Operations

```kappa
Cpu.prefetch.read(address)
Cpu.prefetch.write(address)
Cpu.cache.flush(address)
Cpu.cacheLineSize                   // usually 64 bytes
```

### CPU Information

```kappa
Cpu.info.vendor
Cpu.info.cores
Cpu.info.threads
Cpu.supports.avx2
Cpu.supports.neon

Cpu.pause()                         // spin-wait hint
Cpu.rdtsc()                         // timestamp counter
```

---

## Synchronization

```kappa
// Mutex
val mutex = Mutex()
mutex.withLock { /* critical section */ }

// ReadWriteLock
val rwLock = ReadWriteLock()
rwLock.readLock { /* read */ }
rwLock.writeLock { /* write */ }

// Semaphore
val semaphore = Semaphore(permits = 10)
semaphore.acquire()
semaphore.release()

// CountDownLatch
val latch = CountDownLatch(count = 5)
latch.countDown()
latch.await()

// CyclicBarrier
val barrier = CyclicBarrier(parties = 4)
barrier.await()
```

---

## Asynchrony and Coroutines

### Future and async/await

```kappa
async fn fetchUser(id: long) -> User {
    val response = httpClient.get("/users/$id").await()
    parseJson[User](response.body)
}

async fn fetchAll(ids: List[long]) -> List[User] {
    val futures = ids.map { id -> fetchUser(id) }
    Future.all(futures).await()
}
```

### coro — Coroutines

```kappa
coro fn generateNumbers() -> int {
    var i = 0
    while (true) {
        yield i
        i = i + 1
    }
}

val gen = generateNumbers()
println(gen.next())                 // 0
println(gen.next())                 // 1

for (n in generateNumbers().take(10)) {
    println(n)
}
```

### Channels

```kappa
val channel = Channel[int](capacity = 10)

async fn produce(ch: Channel[int]) {
    for (i in 0 until 100) {
        ch.send(i).await()
    }
    ch.close()
}

async fn consume(ch: Channel[int]) {
    for (item in ch) {
        println("Received: $item")
    }
}
```

### select

```kappa
async fn multiplex(ch1: Channel[int], ch2: Channel[string]) {
    loop {
        select {
            ch1.receive() -> { value -> println("Int: $value") }
            ch2.receive() -> { value -> println("String: $value") }
            timeout(1.seconds) -> { break }
        }
    }
}
```

### Structured Concurrency

```kappa
async fn processAll(items: List[Item]) -> List[Result] {
    scope { s ->
        val futures = items.map { item ->
            s.spawn { process(item) }
        }
        futures.map { it.await() }
    }
}
```

---

## Complete Example: SPSC Ring Buffer

```kappa
struct RingBuffer[T](capacity: int) {
    var .buffer: T?[]
    var .readOffset: Atomic[int] = Atomic(0)
    var .writeOffset: Atomic[int] = Atomic(0)
    pub val capacity: int = capacity
    
    init {
        buffer = T?[](capacity)
    }
    
    pub val size: int {
        val read = readOffset.load.acquire
        val write = writeOffset.load.acquire
        if (write >= read) write - read else capacity - read + write
    } ->
    
    pub val isEmpty: bool { size == 0 } ->
    
    pub val isFull: bool { size == capacity - 1 } ->
    
    pub fn offer(value: T) -> bool {
        val currentWrite = writeOffset.load.relaxed
        val nextWrite = (currentWrite + 1) mod capacity
        val currentRead = readOffset.load.acquire
        
        if (nextWrite == currentRead) {
            return false
        }
        
        buffer[currentWrite] = value
        writeOffset.store.release(nextWrite)
        true
    }
    
    pub fn poll() -> T? {
        val currentRead = readOffset.load.relaxed
        val currentWrite = writeOffset.load.acquire
        
        if (currentRead == currentWrite) {
            return null
        }
        
        val value = buffer[currentRead]
        buffer[currentRead] = null
        readOffset.store.release((currentRead + 1) mod capacity)
        value
    }
    
    pub fn peek() -> T? {
        if (isEmpty) return null
        buffer[readOffset.load.acquire]
    }
    
    pub fn clear() {
        while (poll() != null) { }
    }
}

// Extension with qualifier
fn RingBuffer[T].batch.offer(items: List[T]) -> int {
    var count = 0
    for (item in items) {
        if (!offer(item)) break
        count = count + 1
    }
    count
}

fn RingBuffer[T].batch.poll(maxItems: int) -> List[T] {
    val result = mutableListOf[T]()
    repeat(maxItems) {
        val item = poll() ?: return result
        result.add(item)
    }
    result
}

// Usage
fn main() {
    val ring = RingBuffer[string](100)
    
    ring.offer("Hello")
    ring.offer("World")
    
    println(ring.size)                  // 2
    
    ring.poll()?.let { println(it) }    // "Hello"
    
    // Batch
    ring.batch.offer(["a", "b", "c"])
    val items = ring.batch.poll(10)
    
    // Pipe
    [1, 2, 3, 4, 5]
        |> filter { it mod 2 == 0 }
        |> map { it.toString() }
        |> forEach { ring.offer(it) }
    
    ring.clear()
}
```

---

## Syntax Summary

| Element | Syntax |
|---------|--------|
| Variables | `val x = 1`, `var y = 2` |
| Types | `int`, `string?`, `List[T]` |
| Functions | `fn name(args) -> Type { }` |
| Structures | `struct Name { }` |
| Constructor | `struct Name(val x: T) { init { } }` |
| Interfaces | `interface Name { }` |
| Traits | `trait Name { }` |
| Implementation | `impl Trait for Type { }` |
| Annotations | `[Annotation]` |
| Private fields | `var .name: Type` |
| Public fields | `pub var name: Type` |
| Inline fields | `inline val point: Point` |
| Getter | `{ expression } ->` |
| Setter | `<- { code with value }` |
| Qualifier | `fn Type.method.qualifier()` |
| Ref (keyword) | `val ref a = b` |
| Ref (operator) | `@value` |
| Raw (keyword) | `val raw a = b` |
| Raw (operator) | `&value` |
| Copy | `val a <= b` |
| Nullable | `Type?`, `?.`, `?:`, `!!` |
| Array | `type[](size)`, `[1, 2, 3]` |
| Raw bytes | `value:[offset]` |
| Error | `error value` |
| Handle after variable | `catch varName { handler }` |
| Handle block | `catch` code `{ handler }` |
| Inline handling | `expr catch { default }` |
| Handle on call | `method() { err -> handler }` |
| Handle last error | `catch err -> handler` |
| defer single-line | `defer expr` |
| defer multi-line | `defer { ... }` |
| Pipe forward | `x \|> f` |
| Pipe backward | `f <\| x` |
| Compose forward | `f >> g` |
| Compose backward | `f << g` |
| Range exclusive | `0..n`, `0 until n` |
| Range inclusive | `0..=n` |
| Range reverse | `n downTo 0` |
| Range with step | `0..n step k` |
| Modulo | `a mod b` |
| Operator | `operator fn +(other: T)` |
| Indexing | `operator fn [](i: int)` |
| Atomics | `Atomic[T]`, `Volatile[T]` |
| Memory ordering | `atomic.load.acquire`, `atomic.store.release` |
| Async functions | `async fn name() -> T` |
| Await | `future.await()` |
| Coroutines | `coro fn name() -> T` |
| Yield from coroutine | `yield value` |

## Standard Errors (CommonError)

| Type | Examples |
|------|----------|
| `ArithmeticError` | `DivisionByZero`, `IntegerOverflow`, `IntegerUnderflow` |
| `MemoryError` | `OutOfMemory`, `StackOverflow`, `NullPointer` |
| `IndexError` | `OutOfBounds`, `NegativeIndex` |
| `TypeError` | `CastFailed`, `InvalidConversion` |
| `IoError` | `NotFound`, `PermissionDenied`, `Timeout` |
| `ConcurrencyError` | `Deadlock`, `LockTimeout`, `ThreadInterrupted` |
