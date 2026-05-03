# Refactoring Patterns Catalog

> **Source:** Martin Fowler's "Refactoring: Improving the Design of Existing Code" (2nd Edition)  
> **Adapted for:** Rust-based 3D Slicer Project  
> **Purpose:** Complete reference for AI agents and developers to identify and apply refactorings

---

## Table of Contents

1. [Philosophy & Principles](#philosophy--principles)
2. [Code Smells (When to Refactor)](#code-smells-when-to-refactor)
3. [Refactoring Catalog](#refactoring-catalog)
   - [Basic Refactorings](#1-basic-refactorings)
   - [Encapsulation](#2-encapsulation)
   - [Moving Features](#3-moving-features)
   - [Organizing Data](#4-organizing-data)
   - [Simplifying Conditional Logic](#5-simplifying-conditional-logic)
   - [Refactoring APIs](#6-refactoring-apis)
   - [Dealing with Inheritance](#7-dealing-with-inheritance)
4. [Rust-Specific Patterns](#rust-specific-patterns)
5. [Quick Reference Matrix](#quick-reference-matrix)

---

## Philosophy & Principles

### What is Refactoring?

> "Refactoring is the process of changing a software system in a way that does not alter the external behavior of the code yet improves its internal structure."

**Key insight:** Refactoring is not about fixing bugs or adding features. It's about making code easier to understand and cheaper to modify.

### The Two Hats

When programming, you wear two hats:
1. **Adding functionality** — Don't change existing code structure
2. **Refactoring** — Don't add functionality, only restructure

Never wear both hats simultaneously. Switch between them consciously.

### When to Refactor

| Timing | Description |
|--------|-------------|
| **Preparatory** | Before adding a feature, refactor to make the addition easy |
| **Comprehension** | While reading code, refactor to clarify understanding |
| **Litter-Pickup** | When you see something wrong, fix it (camp site rule) |
| **Planned** | Dedicated time to address accumulated debt (rare, last resort) |

### The Rule of Three

> "The first time you do something, you just do it. The second time you do something similar, you wince at the duplication, but you do the duplicate thing anyway. The third time you do something similar, you refactor."

### Design Stamina Hypothesis

Good internal design increases development stamina over time. Without refactoring, each feature becomes progressively harder. With refactoring, velocity can remain constant or even increase.

---

## Code Smells (When to Refactor)

Code smells are indicators that refactoring may be beneficial. They are heuristics, not rules.

### 1. Mysterious Name

**What it is:** Names that don't communicate purpose clearly.

**Signs:**
- You need to read the implementation to understand what something does
- Names are abbreviated or use internal jargon
- You can't remember what a function does without looking it up

**Refactorings:**
- Change Function Declaration (rename)
- Rename Variable
- Rename Field

**Rust example:**
```rust
// Smell
fn proc_m(m: &Mesh, p: &Params) -> Vec<Layer> { ... }

// Better
fn slice_mesh_into_layers(mesh: &Mesh, slice_params: &SliceParams) -> Vec<Layer> { ... }
```

---

### 2. Duplicated Code

**What it is:** Same code structure appears in multiple places.

**Signs:**
- Copy-paste patterns in the codebase
- Similar logic with minor variations
- Parallel changes needed when fixing bugs

**Refactorings:**
- Extract Function
- Slide Statements (to group similar code)
- Pull Up Method (for sibling class duplication)

**Rust example:**
```rust
// Smell - duplicated validation in multiple functions
fn process_stl(path: &Path) -> Result<Mesh> {
    if !path.exists() { return Err(FileNotFound); }
    if path.extension() != Some("stl") { return Err(InvalidFormat); }
    // ... process
}

fn process_3mf(path: &Path) -> Result<Mesh> {
    if !path.exists() { return Err(FileNotFound); }
    if path.extension() != Some("3mf") { return Err(InvalidFormat); }
    // ... process
}

// Better - extract common validation
fn validate_file(path: &Path, expected_ext: &str) -> Result<()> {
    if !path.exists() { return Err(FileNotFound); }
    if path.extension() != Some(expected_ext) { return Err(InvalidFormat); }
    Ok(())
}
```

---

### 3. Long Function

**What it is:** Functions that do too much and are hard to understand.

**Signs:**
- Function body exceeds ~20-30 lines
- Multiple levels of abstraction in one function
- Comments explaining what sections do
- Many local variables and parameters

**Refactorings:**
- Extract Function (most common)
- Replace Temp with Query
- Introduce Parameter Object
- Replace Function with Command
- Decompose Conditional
- Replace Loop with Pipeline

**Heuristic:** If you feel the need to write a comment explaining what a block of code does, extract it into a function named after that comment.

**Rust example:**
```rust
// Smell - long function with comments as section markers
fn slice_layer(mesh: &Mesh, z: f64) -> Layer {
    // Find intersecting triangles
    let triangles: Vec<_> = mesh.triangles.iter()
        .filter(|t| t.intersects_z(z))
        .collect();
    
    // Calculate intersection segments
    let mut segments = Vec::new();
    for tri in &triangles {
        let seg = calculate_intersection(tri, z);
        segments.push(seg);
    }
    
    // Build contours from segments
    let mut contours = Vec::new();
    while !segments.is_empty() {
        let contour = build_contour(&mut segments);
        contours.push(contour);
    }
    
    // Classify contours as outer/inner
    // ... more code
    
    Layer { contours, z }
}

// Better - each comment becomes a function
fn slice_layer(mesh: &Mesh, z: f64) -> Layer {
    let triangles = find_intersecting_triangles(mesh, z);
    let segments = calculate_all_intersections(&triangles, z);
    let contours = build_contours_from_segments(segments);
    let classified = classify_contours(contours);
    Layer { contours: classified, z }
}
```

---

### 4. Long Parameter List

**What it is:** Functions with too many parameters.

**Signs:**
- More than 3-4 parameters
- Parameters that always appear together
- Boolean flags that change behavior

**Refactorings:**
- Replace Parameter with Query
- Preserve Whole Object
- Introduce Parameter Object
- Remove Flag Argument
- Combine Functions into Class

**Rust example:**
```rust
// Smell
fn generate_infill(
    polygon: &Polygon,
    density: f64,
    pattern: InfillPattern,
    angle: f64,
    line_width: f64,
    overlap: f64,
    connect_lines: bool,
) -> Vec<Path> { ... }

// Better - introduce parameter object
struct InfillParams {
    density: f64,
    pattern: InfillPattern,
    angle: f64,
    line_width: f64,
    overlap: f64,
    connect_lines: bool,
}

fn generate_infill(polygon: &Polygon, params: &InfillParams) -> Vec<Path> { ... }
```

---

### 5. Global Data

**What it is:** Data accessible from anywhere, modifiable by anyone.

**Signs:**
- `static mut` variables
- Singletons with mutable state
- Configuration accessed directly everywhere

**Refactorings:**
- Encapsulate Variable

**Rust approach:** Use dependency injection, pass configuration explicitly, or use `OnceCell`/`Lazy` for truly global immutable data.

---

### 6. Mutable Data

**What it is:** Data that can be changed from multiple places.

**Signs:**
- Shared mutable state
- Functions with side effects
- State that's hard to track

**Refactorings:**
- Encapsulate Variable
- Split Variable
- Slide Statements
- Extract Function
- Separate Query from Modifier
- Remove Setting Method
- Replace Derived Variable with Query
- Combine Functions into Class/Transform
- Change Reference to Value

**Rust advantage:** Ownership system helps, but interior mutability (`RefCell`, `Mutex`) can still create problems.

---

### 7. Divergent Change

**What it is:** One module changes for multiple unrelated reasons.

**Signs:**
- "I have to change these functions for database changes AND these other functions for UI changes"
- Module has multiple responsibilities

**Refactorings:**
- Split Phase
- Move Function
- Extract Function
- Extract Class (Extract Module in Rust)

---

### 8. Shotgun Surgery

**What it is:** One change requires edits in many different places.

**Signs:**
- Small feature requires touching many files
- Easy to miss a required change
- Related code is scattered

**Refactorings:**
- Move Function
- Move Field
- Combine Functions into Class
- Combine Functions into Transform
- Split Phase
- Inline Function/Class (to consolidate before re-extracting properly)

---

### 9. Feature Envy

**What it is:** A function uses more features from another module than its own.

**Signs:**
- Function calls many getters on another object
- Function processes data that "belongs" elsewhere
- Heavy use of another module's internals

**Refactorings:**
- Move Function
- Extract Function (then Move)

**Rust example:**
```rust
// Smell - this function envies Polygon's data
fn calculate_area(polygon: &Polygon) -> f64 {
    let points = polygon.points();
    let mut area = 0.0;
    for i in 0..points.len() {
        let j = (i + 1) % points.len();
        area += points[i].x * points[j].y;
        area -= points[j].x * points[i].y;
    }
    area.abs() / 2.0
}

// Better - move to Polygon
impl Polygon {
    fn area(&self) -> f64 {
        // ... same logic, but now it's where it belongs
    }
}
```

---

### 10. Data Clumps

**What it is:** Groups of data that appear together repeatedly.

**Signs:**
- Same 3-4 fields appear in multiple structs
- Same parameters passed together to multiple functions
- Deleting one piece would make others meaningless

**Refactorings:**
- Extract Class (create a struct for the clump)
- Introduce Parameter Object
- Preserve Whole Object

**Rust example:**
```rust
// Smell - x, y, z always appear together
fn translate(x: f64, y: f64, z: f64, dx: f64, dy: f64, dz: f64) -> (f64, f64, f64)

fn rotate(x: f64, y: f64, z: f64, angle: f64, axis_x: f64, axis_y: f64, axis_z: f64)

// Better
struct Point3D { x: f64, y: f64, z: f64 }
struct Vector3D { x: f64, y: f64, z: f64 }

fn translate(point: Point3D, delta: Vector3D) -> Point3D
fn rotate(point: Point3D, angle: f64, axis: Vector3D) -> Point3D
```

---

### 11. Primitive Obsession

**What it is:** Using primitives instead of small objects for simple tasks.

**Signs:**
- Strings used for structured data (phone numbers, file paths, currency)
- Numbers used where units matter
- Magic numbers/strings throughout code
- Range checks repeated everywhere

**Refactorings:**
- Replace Primitive with Object
- Replace Type Code with Subclasses
- Replace Conditional with Polymorphism
- Extract Class
- Introduce Parameter Object

**Rust example:**
```rust
// Smell - primitives everywhere
fn set_temperature(temp: f64) // Celsius? Fahrenheit? Kelvin?
fn set_layer_height(height: f64) // mm? inches? microns?

// Better - newtypes
struct Millimeters(f64);
struct Celsius(f64);

fn set_temperature(temp: Celsius)
fn set_layer_height(height: Millimeters)

// Even better with validation
struct LayerHeight(f64);
impl LayerHeight {
    fn new(mm: f64) -> Result<Self, Error> {
        if mm < 0.05 || mm > 1.0 {
            return Err(Error::InvalidLayerHeight);
        }
        Ok(Self(mm))
    }
}
```

---

### 12. Repeated Switches

**What it is:** Same switch/match on a type code appears in multiple places.

**Signs:**
- Multiple match statements on the same enum
- Adding a variant requires changes in many places
- Switch on type rather than behavior

**Refactorings:**
- Replace Conditional with Polymorphism

**Rust example:**
```rust
// Smell - same match repeated
fn get_speed(move_type: MoveType) -> f64 {
    match move_type {
        MoveType::Travel => 150.0,
        MoveType::Perimeter => 45.0,
        MoveType::Infill => 80.0,
    }
}

fn get_acceleration(move_type: MoveType) -> f64 {
    match move_type {
        MoveType::Travel => 3000.0,
        MoveType::Perimeter => 1000.0,
        MoveType::Infill => 2000.0,
    }
}

// Better - polymorphism via traits
trait MoveSettings {
    fn speed(&self) -> f64;
    fn acceleration(&self) -> f64;
}

struct TravelMove;
impl MoveSettings for TravelMove {
    fn speed(&self) -> f64 { 150.0 }
    fn acceleration(&self) -> f64 { 3000.0 }
}
// etc.
```

---

### 13. Loops

**What it is:** Imperative loops where functional pipelines would be clearer.

**Signs:**
- Loop with multiple responsibilities
- Loop that transforms data
- Loop that accumulates results

**Refactorings:**
- Replace Loop with Pipeline

**Rust example:**
```rust
// Smell
let mut valid_layers = Vec::new();
for layer in &layers {
    if layer.has_geometry() {
        let processed = process_layer(layer);
        valid_layers.push(processed);
    }
}

// Better
let valid_layers: Vec<_> = layers.iter()
    .filter(|l| l.has_geometry())
    .map(process_layer)
    .collect();
```

---

### 14. Lazy Element

**What it is:** A program element (function, struct, module) that doesn't do enough to justify its existence.

**Signs:**
- Function that just calls another function
- Struct with only one field
- Module with one small function
- Abstraction that adds no value

**Refactorings:**
- Inline Function
- Inline Class
- Collapse Hierarchy

---

### 15. Speculative Generality

**What it is:** "We might need this someday" abstractions that aren't actually used.

**Signs:**
- Abstract base classes with only one implementation
- Unused parameters "for future flexibility"
- Complex frameworks for simple tasks
- Type parameters that are never varied

**Refactorings:**
- Collapse Hierarchy
- Inline Function
- Inline Class
- Change Function Declaration (remove unused params)
- Remove Dead Code

---

### 16. Temporary Field

**What it is:** Fields that are only set in certain circumstances.

**Signs:**
- Fields that are `Option<T>` but only `None` temporarily
- Fields with special "unset" values
- Struct requires calling methods in specific order

**Refactorings:**
- Extract Class (move field and its logic together)
- Move Function
- Introduce Special Case

---

### 17. Message Chains

**What it is:** Long chains of object navigation: `a.b().c().d().e()`

**Signs:**
- Deep coupling to object structure
- Changes to intermediate objects break callers
- Long navigation paths

**Refactorings:**
- Hide Delegate
- Extract Function
- Move Function

---

### 18. Middle Man

**What it is:** A class that delegates almost everything to another class.

**Signs:**
- Most methods just call through to another object
- Wrapper that adds no value
- Excessive delegation

**Refactorings:**
- Remove Middle Man
- Inline Function
- Replace Superclass/Subclass with Delegate

---

### 19. Insider Trading

**What it is:** Modules that trade too much data, creating excessive coupling.

**Signs:**
- Two modules know too much about each other's internals
- Bidirectional dependencies
- Friend functions (where they exist)

**Refactorings:**
- Move Function
- Move Field
- Hide Delegate
- Replace Subclass/Superclass with Delegate

---

### 20. Large Class

**What it is:** A class/struct/module that does too much.

**Signs:**
- Too many fields (> 10-15)
- Too many methods (> 20-30)
- Fields used in separate subsets
- Common prefixes/suffixes suggesting groupings

**Refactorings:**
- Extract Class
- Extract Superclass
- Replace Type Code with Subclasses

---

### 21. Alternative Classes with Different Interfaces

**What it is:** Classes that do similar things but have different interfaces.

**Signs:**
- Two classes could be interchangeable but aren't
- Similar functionality with different method names
- Can't use polymorphism where you should

**Refactorings:**
- Change Function Declaration (make signatures match)
- Move Function (until protocols match)
- Extract Superclass

---

### 22. Data Class

**What it is:** Classes that have fields and getters/setters but no behavior.

**Signs:**
- Struct with only data, no methods
- All logic that uses the data lives elsewhere
- Just a "dumb" data holder

**Refactorings:**
- Encapsulate Record
- Remove Setting Method
- Move Function (move behavior into the data class)
- Extract Function

**Note:** Data classes are sometimes appropriate (DTOs, records from Split Phase).

---

### 23. Refused Bequest

**What it is:** Subclass doesn't want/use inherited functionality.

**Signs:**
- Override methods to do nothing
- Inherit from class but only use small portion
- "Is-a" relationship doesn't hold

**Refactorings:**
- Push Down Method/Field
- Replace Subclass with Delegate
- Replace Superclass with Delegate

---

### 24. Comments

**What it is:** Comments as deodorant for bad code.

**Signs:**
- Comments explaining what code does (not why)
- Long comments before complex blocks
- Comments that are out of date with code

**Refactorings:**
- Extract Function (named after the comment)
- Change Function Declaration (rename to be self-documenting)
- Introduce Assertion

**Good comments:** Why decisions were made, domain explanations, warnings about non-obvious consequences.

---

## Refactoring Catalog

### 1. Basic Refactorings

#### Extract Function
**Inverse:** Inline Function

**When:** You see a fragment of code that can be grouped together.

**Motivation:** The key is not function length but semantic distance between what the method does and how it does it. If you have to spend effort to understand what code is doing, extract it and name it.

**Mechanics:**
1. Create new function, name it after what it does (not how)
2. Copy extracted code into new function
3. Check for variables that are local to source but needed in extracted code
4. Pass needed variables as parameters
5. Compile/check for errors
6. Replace extracted code with function call
7. Test
8. Look for similar code that could use the new function

**Rust considerations:**
- Watch for ownership - may need to pass references or clone
- Consider returning `Result` if extracted code can fail
- Use `impl Trait` in return position for iterator methods

```rust
// Before
fn print_owing(invoice: &Invoice) {
    let mut outstanding = 0.0;
    
    println!("***********************");
    println!("**** Customer Owes ****");
    println!("***********************");
    
    for order in &invoice.orders {
        outstanding += order.amount;
    }
    
    println!("name: {}", invoice.customer);
    println!("amount: {}", outstanding);
}

// After
fn print_owing(invoice: &Invoice) {
    print_banner();
    let outstanding = calculate_outstanding(invoice);
    print_details(invoice, outstanding);
}

fn print_banner() {
    println!("***********************");
    println!("**** Customer Owes ****");
    println!("***********************");
}

fn calculate_outstanding(invoice: &Invoice) -> f64 {
    invoice.orders.iter().map(|o| o.amount).sum()
}

fn print_details(invoice: &Invoice, outstanding: f64) {
    println!("name: {}", invoice.customer);
    println!("amount: {}", outstanding);
}
```

---

#### Inline Function
**Inverse:** Extract Function

**When:** The body is as clear as the name, or you need to refactor multiple poorly-factored functions by first inlining them.

**Mechanics:**
1. Check this isn't a polymorphic method (can't inline if overridden)
2. Find all callers
3. Replace each call with the function body
4. Test after each replacement
5. Remove the function definition

```rust
// Before
fn get_rating(driver: &Driver) -> i32 {
    if more_than_five_late_deliveries(driver) { 2 } else { 1 }
}

fn more_than_five_late_deliveries(driver: &Driver) -> bool {
    driver.number_of_late_deliveries > 5
}

// After
fn get_rating(driver: &Driver) -> i32 {
    if driver.number_of_late_deliveries > 5 { 2 } else { 1 }
}
```

---

#### Extract Variable
**Inverse:** Inline Variable

**When:** An expression is complex and hard to read; a named variable would clarify intent.

**Mechanics:**
1. Ensure expression has no side effects
2. Declare immutable variable, set to expression
3. Replace original expression with variable
4. Test
5. Repeat for other occurrences

```rust
// Before
fn price(order: &Order) -> f64 {
    order.quantity as f64 * order.item_price 
        - f64::max(0.0, order.quantity as f64 - 500.0) * order.item_price * 0.05
        + f64::min(order.quantity as f64 * order.item_price * 0.1, 100.0)
}

// After
fn price(order: &Order) -> f64 {
    let base_price = order.quantity as f64 * order.item_price;
    let quantity_discount = f64::max(0.0, order.quantity as f64 - 500.0) * order.item_price * 0.05;
    let shipping = f64::min(base_price * 0.1, 100.0);
    base_price - quantity_discount + shipping
}
```

---

#### Inline Variable
**Inverse:** Extract Variable

**When:** The name doesn't communicate more than the expression itself, or the variable gets in the way of refactoring.

**Mechanics:**
1. Check right-hand side has no side effects
2. Make variable immutable if not already, test (to verify single assignment)
3. Find first reference, replace with expression
4. Test
5. Repeat for all references
6. Remove declaration

---

#### Change Function Declaration
**Also:** Rename Function, Add Parameter, Remove Parameter

**When:** A function's name doesn't reflect its purpose, or its parameters need adjustment.

**Simple Mechanics:**
1. If removing a parameter, ensure it's not used in body
2. Change declaration
3. Find all references, update them
4. Test

**Migration Mechanics (for complex changes):**
1. If needed, refactor body to prepare for extraction
2. Extract function body into new function with desired signature
3. Use temporary name if needed
4. Test
5. Inline old function into callers (one at a time)
6. Rename new function to original name
7. Test

---

#### Encapsulate Variable

**When:** Data is accessed from many places; you want to control access and enable future changes.

**Mechanics:**
1. Create encapsulating functions (getter/setter)
2. Run static checks
3. Replace each reference with function call
4. Test after each replacement
5. Restrict visibility of variable
6. Test

**Rust approach:** Use module privacy and provide accessor methods.

```rust
// Before
pub static mut DEFAULT_SETTINGS: Settings = Settings::default();

// After
mod settings {
    static DEFAULT_SETTINGS: OnceCell<Settings> = OnceCell::new();
    
    pub fn default_settings() -> &'static Settings {
        DEFAULT_SETTINGS.get_or_init(Settings::default)
    }
}
```

---

#### Rename Variable

**When:** A variable's name doesn't clearly communicate its purpose.

**Mechanics:**
1. If widely used, consider Encapsulate Variable first
2. Find all references, change each one
3. Test

---

#### Introduce Parameter Object

**When:** Multiple parameters frequently travel together.

**Mechanics:**
1. Create a new struct for the grouped parameters
2. Test
3. Use Change Function Declaration to add parameter of new type
4. Test
5. Modify callers to pass new struct, setting old params to struct values
6. Delete old parameters one by one, testing after each

```rust
// Before
fn amount_invoiced(start: Date, end: Date) -> Money
fn amount_received(start: Date, end: Date) -> Money
fn amount_overdue(start: Date, end: Date) -> Money

// After
struct DateRange { start: Date, end: Date }

fn amount_invoiced(range: &DateRange) -> Money
fn amount_received(range: &DateRange) -> Money
fn amount_overdue(range: &DateRange) -> Money
```

---

#### Combine Functions into Class

**When:** A group of functions operate on the same data.

**Mechanics:**
1. Use Encapsulate Record on the shared data
2. Move each function into the new class
3. Logic that manipulates the data can be extracted into methods

**Rust:** Create a struct with the data and implement methods.

---

#### Combine Functions into Transform

**When:** You need to derive data from source data in consistent ways.

**Mechanics:**
1. Create a transform function that clones/copies input
2. Move each derivation logic into transform
3. Each piece of derived data becomes a field on the output

**Rust:** Use builder pattern or create a "derived" struct.

---

#### Split Phase

**When:** Code deals with two different things (e.g., parsing then processing).

**Mechanics:**
1. Extract second phase into its own function
2. Test
3. Introduce intermediate data structure to pass between phases
4. Examine each parameter of extracted function, move to intermediate data
5. Apply Extract Function on first phase, returning intermediate data
6. Test

```rust
// Before: parsing and processing mixed
fn price_order(product_data: &str, quantity: i32, shipping: &str) -> f64 {
    let product: Product = parse(product_data); // parsing
    let shipping_method: Shipping = parse(shipping); // parsing
    
    // processing
    let base_price = product.price * quantity as f64;
    let discount = calculate_discount(quantity);
    let shipping_cost = calculate_shipping(shipping_method, quantity);
    base_price - discount + shipping_cost
}

// After: clear phases
fn price_order(product_data: &str, quantity: i32, shipping: &str) -> f64 {
    let order_data = parse_order(product_data, quantity, shipping);
    calculate_price(&order_data)
}

struct OrderData {
    product: Product,
    quantity: i32,
    shipping_method: Shipping,
}

fn parse_order(product_data: &str, quantity: i32, shipping: &str) -> OrderData {
    OrderData {
        product: parse(product_data),
        quantity,
        shipping_method: parse(shipping),
    }
}

fn calculate_price(data: &OrderData) -> f64 {
    let base_price = data.product.price * data.quantity as f64;
    let discount = calculate_discount(data.quantity);
    let shipping_cost = calculate_shipping(&data.shipping_method, data.quantity);
    base_price - discount + shipping_cost
}
```

---

### 2. Encapsulation

#### Encapsulate Record

**When:** You have a record structure that would benefit from being a proper object with behavior.

```rust
// Before
struct Organization {
    pub name: String,
    pub country: String,
}

// After
struct Organization {
    name: String,
    country: String,
}

impl Organization {
    pub fn name(&self) -> &str { &self.name }
    pub fn set_name(&mut self, name: String) { self.name = name; }
    pub fn country(&self) -> &str { &self.country }
    // country is read-only, no setter
}
```

---

#### Encapsulate Collection

**When:** A getter returns a collection that callers can modify directly.

```rust
// Before
impl Person {
    pub fn courses(&self) -> &Vec<Course> { &self.courses }
}
// Caller can do: person.courses().push(course); // Bad!

// After
impl Person {
    pub fn courses(&self) -> impl Iterator<Item = &Course> { 
        self.courses.iter() 
    }
    pub fn add_course(&mut self, course: Course) { 
        self.courses.push(course); 
    }
    pub fn remove_course(&mut self, course: &Course) { 
        self.courses.retain(|c| c != course); 
    }
}
```

---

#### Replace Primitive with Object

**When:** A primitive carries more meaning than just its value.

```rust
// Before
let priority: String = "high".to_string();

// After
#[derive(Clone, PartialEq, Eq, PartialOrd, Ord)]
struct Priority {
    value: String,
    level: i32,
}

impl Priority {
    pub fn high() -> Self { Self { value: "high".into(), level: 3 } }
    pub fn normal() -> Self { Self { value: "normal".into(), level: 2 } }
    pub fn low() -> Self { Self { value: "low".into(), level: 1 } }
    pub fn is_higher_than(&self, other: &Priority) -> bool { self.level > other.level }
}
```

---

#### Replace Temp with Query

**When:** A temporary variable holds the result of an expression that could be a method.

```rust
// Before
fn calculate_total(&self) -> f64 {
    let base_price = self.quantity as f64 * self.item_price;
    if base_price > 1000.0 {
        base_price * 0.95
    } else {
        base_price * 0.98
    }
}

// After
fn base_price(&self) -> f64 {
    self.quantity as f64 * self.item_price
}

fn calculate_total(&self) -> f64 {
    if self.base_price() > 1000.0 {
        self.base_price() * 0.95
    } else {
        self.base_price() * 0.98
    }
}
```

---

#### Extract Class

**When:** A class does too much and has data/methods that belong together as a separate concept.

```rust
// Before
struct Person {
    name: String,
    office_area_code: String,
    office_number: String,
}

impl Person {
    fn telephone_number(&self) -> String {
        format!("({}) {}", self.office_area_code, self.office_number)
    }
}

// After
struct TelephoneNumber {
    area_code: String,
    number: String,
}

impl TelephoneNumber {
    fn to_string(&self) -> String {
        format!("({}) {}", self.area_code, self.number)
    }
}

struct Person {
    name: String,
    office_telephone: TelephoneNumber,
}
```

---

#### Inline Class
**Inverse:** Extract Class

**When:** A class isn't doing enough to justify its existence.

---

#### Hide Delegate

**When:** Client code navigates through one object to get to another.

```rust
// Before
let manager = person.department().manager();

// After
impl Person {
    fn manager(&self) -> &Employee {
        self.department.manager()
    }
}
let manager = person.manager();
```

---

#### Remove Middle Man
**Inverse:** Hide Delegate

**When:** A class has too many delegating methods.

---

#### Substitute Algorithm

**When:** You want to replace an algorithm with a clearer or better one.

```rust
// Before
fn found_person(people: &[String]) -> Option<&String> {
    for person in people {
        if person == "Don" { return Some(person); }
        if person == "John" { return Some(person); }
        if person == "Kent" { return Some(person); }
    }
    None
}

// After
fn found_person(people: &[String]) -> Option<&String> {
    let candidates = ["Don", "John", "Kent"];
    people.iter().find(|p| candidates.contains(&p.as_str()))
}
```

---

### 3. Moving Features

#### Move Function

**When:** A function makes more sense in a different context/module.

**Mechanics:**
1. Examine what the function uses from its current context
2. Check if function is polymorphic
3. Copy function to target context
4. Adjust function to fit new context
5. Turn source function into delegating function
6. Test
7. Consider inlining source function

---

#### Move Field

**When:** A field is used more by another struct than its current home.

---

#### Move Statements into Function

**When:** Certain statements always appear with a function call.

---

#### Move Statements to Callers
**Inverse:** Move Statements into Function

**When:** A function does something that should be caller-specific.

---

#### Replace Inline Code with Function Call

**When:** Existing code does exactly what a well-named function would do.

```rust
// Before
let app_needs_update = !latest_version_matches_current();

// After  
let app_needs_update = needs_update(); // Uses the already-existing function
```

---

#### Slide Statements

**When:** Related code should be adjacent.

```rust
// Before
let pricing_plan = retrieve_pricing_plan();
let order = retrieve_order();
let base_charge = pricing_plan.base;
// ... intervening code
let charge = base_charge + pricing_plan.unit * order.units;

// After
let pricing_plan = retrieve_pricing_plan();
let base_charge = pricing_plan.base;
let order = retrieve_order();
let charge = base_charge + pricing_plan.unit * order.units;
```

---

#### Split Loop

**When:** A loop does multiple things.

```rust
// Before
let mut youngest_age = i32::MAX;
let mut total_salary = 0;
for person in &people {
    if person.age < youngest_age { youngest_age = person.age; }
    total_salary += person.salary;
}

// After
let youngest_age = people.iter().map(|p| p.age).min().unwrap_or(i32::MAX);
let total_salary: i32 = people.iter().map(|p| p.salary).sum();
```

---

#### Replace Loop with Pipeline

**When:** Loop logic can be expressed as a pipeline of operations.

```rust
// Before
let mut names = Vec::new();
for person in &people {
    if person.department == "Engineering" {
        names.push(&person.name);
    }
}

// After
let names: Vec<_> = people.iter()
    .filter(|p| p.department == "Engineering")
    .map(|p| &p.name)
    .collect();
```

---

#### Remove Dead Code

**When:** Code is never executed.

**Just delete it.** Version control remembers it if you need it back.

---

### 4. Organizing Data

#### Split Variable

**When:** A variable is assigned more than once (not counting loop counters or accumulating results).

```rust
// Before
let mut temp = 2.0 * (height + width);
println!("perimeter: {}", temp);
temp = height * width;
println!("area: {}", temp);

// After
let perimeter = 2.0 * (height + width);
println!("perimeter: {}", perimeter);
let area = height * width;
println!("area: {}", area);
```

---

#### Rename Field

**When:** A field name doesn't clearly communicate its purpose.

---

#### Replace Derived Variable with Query

**When:** A variable holds a value that could be calculated from other data.

```rust
// Before
struct ProductionPlan {
    adjustment: i32,
    production: i32, // derived from adjustments
}

// After
struct ProductionPlan {
    adjustments: Vec<Adjustment>,
}

impl ProductionPlan {
    fn production(&self) -> i32 {
        self.adjustments.iter().map(|a| a.amount).sum()
    }
}
```

---

#### Change Reference to Value

**When:** A reference object is small and should be immutable.

---

#### Change Value to Reference
**Inverse:** Change Reference to Value

**When:** You need to share state between multiple instances.

---

### 5. Simplifying Conditional Logic

#### Decompose Conditional

**When:** A complex conditional has complex condition and/or branches.

```rust
// Before
if date < SUMMER_START || date > SUMMER_END {
    charge = quantity * winter_rate + winter_service_charge;
} else {
    charge = quantity * summer_rate;
}

// After
if is_winter(&date) {
    charge = winter_charge(quantity);
} else {
    charge = summer_charge(quantity);
}

fn is_winter(date: &Date) -> bool { date < &SUMMER_START || date > &SUMMER_END }
fn winter_charge(quantity: i32) -> f64 { quantity as f64 * WINTER_RATE + WINTER_SERVICE_CHARGE }
fn summer_charge(quantity: i32) -> f64 { quantity as f64 * SUMMER_RATE }
```

---

#### Consolidate Conditional Expression

**When:** Multiple conditionals lead to the same result.

```rust
// Before
if employee.seniority < 2 { return 0.0; }
if employee.months_disabled > 12 { return 0.0; }
if employee.is_part_time { return 0.0; }

// After
if is_not_eligible_for_disability(&employee) { return 0.0; }

fn is_not_eligible_for_disability(emp: &Employee) -> bool {
    emp.seniority < 2 || emp.months_disabled > 12 || emp.is_part_time
}
```

---

#### Replace Nested Conditional with Guard Clauses

**When:** One branch is the normal case and others are exceptional.

```rust
// Before
fn pay_amount(employee: &Employee) -> f64 {
    let result;
    if employee.is_separated {
        result = separated_amount();
    } else {
        if employee.is_retired {
            result = retired_amount();
        } else {
            result = normal_pay_amount();
        }
    }
    result
}

// After
fn pay_amount(employee: &Employee) -> f64 {
    if employee.is_separated { return separated_amount(); }
    if employee.is_retired { return retired_amount(); }
    normal_pay_amount()
}
```

---

#### Replace Conditional with Polymorphism

**When:** You have a conditional that switches on type.

```rust
// Before
fn plumage(bird: &Bird) -> &str {
    match bird.kind {
        BirdKind::European => "average",
        BirdKind::African => if bird.number_of_coconuts > 2 { "tired" } else { "average" },
        BirdKind::Norwegian => if bird.voltage > 100 { "scorched" } else { "beautiful" },
    }
}

// After
trait Bird {
    fn plumage(&self) -> &str;
}

struct EuropeanSwallow;
impl Bird for EuropeanSwallow {
    fn plumage(&self) -> &str { "average" }
}

struct AfricanSwallow { number_of_coconuts: i32 }
impl Bird for AfricanSwallow {
    fn plumage(&self) -> &str {
        if self.number_of_coconuts > 2 { "tired" } else { "average" }
    }
}

struct NorwegianBlueParrot { voltage: i32 }
impl Bird for NorwegianBlueParrot {
    fn plumage(&self) -> &str {
        if self.voltage > 100 { "scorched" } else { "beautiful" }
    }
}
```

---

#### Introduce Special Case

**When:** Many places check for a special value (like null/None) and do the same thing.

```rust
// Before - scattered None checks
fn customer_name(site: &Site) -> String {
    match &site.customer {
        Some(c) => c.name.clone(),
        None => "occupant".to_string(),
    }
}

// After - special case object
impl Customer {
    fn unknown() -> Self {
        Customer { name: "occupant".to_string(), /* ... */ }
    }
    fn is_unknown(&self) -> bool { self.name == "occupant" }
}
```

---

#### Introduce Assertion

**When:** Code assumes something about state that should be explicit.

```rust
fn apply_discount(product: &Product, discount: f64) -> f64 {
    assert!(discount >= 0.0 && discount <= 1.0, "Discount must be between 0 and 1");
    product.base_price * (1.0 - discount)
}
```

---

### 6. Refactoring APIs

#### Separate Query from Modifier

**When:** A function both returns a value and changes state.

```rust
// Before
fn get_total_and_send_bill(&mut self) -> f64 {
    let total = self.calculate_total();
    self.send_bill(total);
    total
}

// After
fn total(&self) -> f64 { self.calculate_total() }
fn send_bill(&mut self) { /* ... */ }
```

---

#### Parameterize Function

**When:** Multiple functions do similar things with different values.

```rust
// Before
fn ten_percent_raise(person: &mut Person) { person.salary *= 1.10; }
fn five_percent_raise(person: &mut Person) { person.salary *= 1.05; }

// After
fn raise(person: &mut Person, factor: f64) { person.salary *= 1.0 + factor; }
```

---

#### Remove Flag Argument

**When:** A boolean argument controls which logic path to follow.

```rust
// Before
fn book_concert(customer: &Customer, is_premium: bool) {
    if is_premium {
        // premium booking logic
    } else {
        // regular booking logic
    }
}

// After
fn book_regular_concert(customer: &Customer) { /* ... */ }
fn book_premium_concert(customer: &Customer) { /* ... */ }
```

---

#### Preserve Whole Object

**When:** You pull several values from an object to pass them as separate arguments.

```rust
// Before
let low = room.days_temp_range.low;
let high = room.days_temp_range.high;
if plan.within_range(low, high) { /* ... */ }

// After
if plan.within_range(&room.days_temp_range) { /* ... */ }
```

---

#### Replace Parameter with Query

**When:** A parameter can be derived from other data available to the function.

---

#### Replace Query with Parameter
**Inverse:** Replace Parameter with Query

**When:** You want to remove a dependency from a function (making it pure).

---

#### Remove Setting Method

**When:** A field should be set only at construction time.

---

#### Replace Constructor with Factory Function

**When:** You need more flexibility than a constructor provides.

```rust
// Before
let employee = Employee::new(name, EmployeeType::Engineer);

// After
let employee = Employee::create_engineer(name);
```

---

#### Replace Function with Command

**When:** A function becomes complex and needs more structure.

```rust
// After - complex calculation becomes a command object
struct ScoreCalculator {
    candidate: Candidate,
    medical_exam: MedicalExam,
    scoring_guide: ScoringGuide,
}

impl ScoreCalculator {
    fn new(candidate: Candidate, medical_exam: MedicalExam, scoring_guide: ScoringGuide) -> Self {
        Self { candidate, medical_exam, scoring_guide }
    }
    
    fn execute(&self) -> i32 {
        // can now use private helper methods
        let health_score = self.score_health();
        let certification_score = self.score_certifications();
        health_score + certification_score
    }
    
    fn score_health(&self) -> i32 { /* ... */ }
    fn score_certifications(&self) -> i32 { /* ... */ }
}
```

---

#### Replace Command with Function
**Inverse:** Replace Function with Command

**When:** A command object is simple enough to be a function.

---

### 7. Dealing with Inheritance

#### Pull Up Method

**When:** Methods in sibling subclasses are identical.

---

#### Pull Up Field

**When:** Subclasses have duplicate fields.

---

#### Pull Up Constructor Body

**When:** Subclass constructors have common code.

---

#### Push Down Method
**Inverse:** Pull Up Method

**When:** A method is only relevant to one subclass.

---

#### Push Down Field
**Inverse:** Pull Up Field

**When:** A field is only used by one subclass.

---

#### Replace Type Code with Subclasses

**When:** A type code affects behavior.

```rust
// Before
enum EmployeeType { Engineer, Manager, Salesperson }
struct Employee {
    kind: EmployeeType,
    // ...
}

// After - using traits
trait Employee {
    fn calculate_bonus(&self) -> f64;
}

struct Engineer { /* ... */ }
impl Employee for Engineer {
    fn calculate_bonus(&self) -> f64 { /* ... */ }
}

struct Manager { /* ... */ }
impl Employee for Manager {
    fn calculate_bonus(&self) -> f64 { /* ... */ }
}
```

---

#### Remove Subclass
**Inverse:** Replace Type Code with Subclasses

**When:** Subclasses don't add enough value.

---

#### Extract Superclass

**When:** Two classes have similar features.

---

#### Collapse Hierarchy

**When:** A superclass and subclass are too similar to keep separate.

---

#### Replace Subclass with Delegate

**When:** You need to vary behavior independently of other variations.

```rust
// Before: inheritance
struct Booking { /* ... */ }
struct PremiumBooking { booking: Booking, extras: Extras }

// After: delegation
struct Booking {
    premium_delegate: Option<PremiumBookingDelegate>,
}

struct PremiumBookingDelegate {
    host_booking: *const Booking, // or Arc<Booking>
    extras: Extras,
}
```

---

#### Replace Superclass with Delegate

**When:** Inheritance isn't appropriate (subclass doesn't satisfy "is-a" relationship).

---

## Rust-Specific Patterns

### Ownership-Related Refactorings

#### Extract to Owned Type

**When:** A reference has lifetime issues; make it owned instead.

```rust
// Before - lifetime complexity
struct Parser<'a> {
    input: &'a str,
    tokens: Vec<Token<'a&gt;&gt;,
}

// After - owns its data
struct Parser {
    input: String,
    tokens: Vec<Token>,
}
```

---

#### Replace Clone with Reference

**When:** Excessive cloning; reference/borrow could work.

---

#### Introduce Arc/Rc for Shared Ownership

**When:** Multiple owners needed; cloning is too expensive.

---

### Error Handling Refactorings

#### Replace Panic with Result

**When:** A panic should be a recoverable error.

```rust
// Before
fn parse(s: &str) -> Config {
    serde_json::from_str(s).unwrap() // panics
}

// After
fn parse(s: &str) -> Result<Config, ConfigError> {
    serde_json::from_str(s).map_err(ConfigError::Parse)
}
```

---

#### Consolidate Error Types

**When:** Multiple error types could be unified with an enum.

```rust
// Before
fn process() -> Result<Output, Box<dyn Error&gt;&gt;

// After
#[derive(Debug, thiserror::Error)]
enum ProcessError {
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
    #[error("Parse error: {0}")]
    Parse(#[from] serde_json::Error),
}

fn process() -> Result<Output, ProcessError>
```

---

### Trait-Based Refactorings

#### Extract Trait

**When:** You need polymorphism or want to decouple implementations.

```rust
// Before - concrete types everywhere
fn slice(mesh: &TriangleMesh) -> Layers

// After - abstraction
trait Meshable {
    fn triangles(&self) -> &[Triangle];
}

fn slice(mesh: &impl Meshable) -> Layers
```

---

#### Replace Dynamic with Static Dispatch

**When:** `dyn Trait` overhead isn't needed.

```rust
// Before
fn process(handler: &dyn Handler) { /* ... */ }

// After
fn process<H: Handler>(handler: &H) { /* ... */ }
// Or
fn process(handler: impl Handler) { /* ... */ }
```

---

## Quick Reference Matrix

### Smell → Refactoring Mapping

| Smell | Primary Refactorings |
|-------|---------------------|
| Mysterious Name | Rename Function/Variable/Field |
| Duplicated Code | Extract Function, Pull Up Method, Slide Statements |
| Long Function | Extract Function, Replace Temp with Query, Decompose Conditional |
| Long Parameter List | Introduce Parameter Object, Preserve Whole Object, Remove Flag Argument |
| Global Data | Encapsulate Variable |
| Mutable Data | Encapsulate Variable, Split Variable, Separate Query from Modifier |
| Divergent Change | Split Phase, Move Function, Extract Class |
| Shotgun Surgery | Move Function/Field, Combine Functions into Class, Inline Function |
| Feature Envy | Move Function, Extract Function |
| Data Clumps | Extract Class, Introduce Parameter Object |
| Primitive Obsession | Replace Primitive with Object, Replace Type Code with Subclasses |
| Repeated Switches | Replace Conditional with Polymorphism |
| Loops | Replace Loop with Pipeline |
| Lazy Element | Inline Function/Class, Collapse Hierarchy |
| Speculative Generality | Collapse Hierarchy, Inline Function, Remove Dead Code |
| Temporary Field | Extract Class, Introduce Special Case |
| Message Chains | Hide Delegate, Extract Function, Move Function |
| Middle Man | Remove Middle Man, Inline Function |
| Insider Trading | Move Function/Field, Hide Delegate |
| Large Class | Extract Class, Extract Superclass, Replace Type Code with Subclasses |
| Data Class | Encapsulate Record, Move Function |
| Refused Bequest | Replace Subclass/Superclass with Delegate |
| Comments | Extract Function, Rename, Introduce Assertion |

---

## Refactoring Safety Checklist

Before refactoring:
- [ ] Tests pass
- [ ] Code compiles
- [ ] You understand what the code does
- [ ] You have a clear goal

During refactoring:
- [ ] Take small steps
- [ ] Compile after each change (Rust catches a lot!)
- [ ] Test frequently
- [ ] Commit often

After refactoring:
- [ ] All tests still pass
- [ ] Code is cleaner/clearer
- [ ] No functionality changed
- [ ] Consider if more refactoring is needed

---

*Catalog based on Martin Fowler's "Refactoring: Improving the Design of Existing Code" (2nd Edition), adapted for Rust and the Klipper Slicer project.*
