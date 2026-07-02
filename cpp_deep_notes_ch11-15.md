# C++ Deep-Dive Notes — Chapters 11–15 (LearnCpp.com)
### Exhaustive, interview/OA-oriented — every lesson, every gotcha

---
# CHAPTER 11 — Function Overloading and Function Templates

## 11.1 Introduction to function overloading
- Function overloading = multiple functions sharing the same name in the same scope, provided the compiler can **differentiate** them.
- Each identically-named function is called an **overload**; the whole set is the **overload set**.
- Motivation: lets you use one intuitive name (`add`, `print`, `max`) for logically-the-same operation across different types, instead of inventing `addInt`, `addDouble`, etc.
- Two separate mechanisms are involved, covered in the next two lessons:
  1. **Differentiation** — how the compiler tells overloads apart (compile-time check on the *declarations*).
  2. **Overload resolution** — how the compiler picks which overload a *specific call* should use.
- If overloads aren't properly differentiated → compile error at the declaration. If a call can't be resolved (or resolves ambiguously) → compile error at the call site.
- Operators are functions under the hood too — operator overloading (Ch 21) is function overloading applied to operator functions.

## 11.2 Function overload differentiation — exact rules
A function's **signature** = name + parameter list (number, order, types) + certain qualifiers. Two functions are differentiated (i.e., considered distinct overloads, not redefinitions) if they differ in:
1. **Number of parameters**: `f(int)` vs `f(int, int)`.
2. **Type of parameters** (not including typedefs/aliases — see below), including:
   - Ellipsis parameters count as a distinct/unique parameter type: `foo(int, int)` vs `foo(int, ...)` are differentiated.
3. Qualifiers: a `const` member function is differentiated from an otherwise-identical non-const member function, even with the same parameter list (this is exactly how `const` and non-const overloads of `operator[]` coexist).

**What does NOT differentiate:**
- **Return type is never used for differentiation.** `int f(); double f();` is a *redefinition* error, not a valid overload — the compiler has no reliable way to disambiguate purely by how the return value is used.
- **Typedefs / type aliases are not distinct types.** `typedef int Height; using Age = int;` — `print(int)`, `print(Height)`, `print(Age)` are all the SAME signature → redefinition error.
- **Top-level `const` on by-value parameters doesn't count.** `print(int)` and `print(const int)` are NOT differentiated (both just mean "pass an int by value"; the const only affects the parameter's mutability inside the function body, invisible to the caller) → redefinition error. (Note: this is different from `print(int&)` vs `print(const int&)`, which ARE differentiated — const on a reference/pointer parameter *is* part of the type.)

**Name mangling**: internally, the compiler encodes parameter info into each overload's compiled ("mangled") symbol name so the linker sees unique names — e.g. conceptually `fcn()` → `__fcn_v`, `fcn(int)` → `__fcn_i`. This is why you can't mix C and C++ overloaded function linkage without `extern "C"` caveats (C has no mangling/overloading).

## 11.3 Function overload resolution and ambiguous matches — the 6-step algorithm
When a call is made to an overloaded function, the compiler checks candidate functions in this **strict order** per argument, and picks the "best" overall match:

1. **Exact match** — including trivial conversions that don't change meaning: `T` ↔ `const T` (for by-value/rvalue-ref contexts), array-to-pointer decay, function-to-pointer decay.
2. **Match via numeric promotion** — small types promoted to larger "natural" types: `char/short/unscoped-enum` → `int`, `float` → `double`, `bool` → `int`. These are considered "safe"/preferred over conversions.
3. **Match via numeric conversion** — any other standard conversion between arithmetic types: `int`→`double`, `double`→`int`, `int`→`char`, etc. (lossy conversions are still allowed here, just lower priority).
4. **Match via user-defined conversion** — an implicit conversion supplied by a converting constructor or an overloaded typecast operator on a class type (Ch 14/21).
5. **Match via ellipsis** — matches a `...` parameter (last resort, covered fully in Ch 20).
6. **No match found** → compile error ("no matching function").

At each step, if exactly one function is the best match at the *earliest* successful step, that one is called. If **two or more** functions tie at the same best step → **ambiguous match** → compile error (the compiler does NOT arbitrarily pick one).

**Classic ambiguous examples:**
```cpp
void print(int x);
void print(double x);
print('a');   // char -> int is a promotion (step 2); char -> double is a conversion (step 3)
              // step 2 wins outright -> calls print(int), NOT ambiguous

void foo(unsigned int);
void foo(float);
foo(0);       // int -> unsigned int and int -> float are BOTH numeric conversions (step 3)
              // tie at step 3 -> AMBIGUOUS, compile error
```
- `5L` (long) vs overloads `print(int)`/`print(double)`: long→int and long→double are both numeric conversions (step 3) → ambiguous.

**Resolving ambiguity:**
1. Add a new overload with the exact parameter type of the call (gives an exact-match, step-1 candidate).
2. Explicitly `static_cast` the argument to force it toward the type you want: `foo(static_cast<unsigned int>(x));`
3. If calling a specific overload by literal, use a suffix (`5u`, `5.0f`) to change its type outright.

## 11.4 Deleted functions
- `= delete` on a function/overload marks it non-callable: `void print(char) = delete;`
- The deleted function **still participates in overload resolution** — if it's selected as the best match, you get a compile error citing the *deleted function*, not "no match found." This is intentional: it lets you explicitly forbid specific conversions/overloads (e.g., stop `print(5)` from silently promoting a `char` you actually wanted blocked, or make a class non-copyable via `T(const T&) = delete;`).
- Any function can be deleted, not just special member functions.

## 11.5 Default arguments
- Syntax: `void print(int x, int y = 10);` — caller can omit trailing arguments.
- **Rule: only the rightmost parameters can have defaults.** `void f(int a = 1, int b);` is illegal — once you default one parameter, all parameters to its right must also be defaulted.
- If a function is declared and defined separately (e.g. `.h` prototype + `.cpp` definition), the default value should be specified **only once**, conventionally in the forward declaration, so all call sites (which only see the header) know it — specifying it in both places is redundant and can cause a redefinition error if the values differ.
- **Default arguments and overload resolution don't mix well**: default-valued parameters are NOT used for differentiation purposes the way you might expect, and calls can become ambiguous:
```cpp
void print(long x);
void print(double x);
void print(int x = 0); // hypothetical 3rd overload
print(5); // ambiguous in certain configurations, because default-argument presence doesn't participate in step-based resolution logic the same as an exact int overload would
```
  (Concretely: the well-known example is `print(long)` + `print(double)` — calling `print(5)`, an `int`, is ambiguous because int→long and int→double are both numeric conversions, same as above; adding defaults doesn't fix an already-ambiguous numeric-conversion tie.)

## 11.6–11.7 Function templates & instantiation
- A **function template** is not a real function — it's a pattern/stencil the compiler uses to generate ("stamp out") real functions for specific types on demand.
```cpp
template <typename T>
T max(T a, T b) { return (a > b) ? a : b; }
```
- `template <typename T>` is the **template parameter declaration**; `T` is a **type template parameter** — a placeholder for a type to be determined later.
- **Function template instantiation**: the process of the compiler generating a concrete function from the template for a specific type. This happens automatically the first time the template is used with a given type — called **implicit instantiation**.
- Each distinct type used creates its OWN separately-compiled function (this is why it's "zero-cost abstraction" at runtime — no dynamic dispatch — but it increases compiled binary size, called **code bloat**, since `max<int>`, `max<double>`, etc. are all separate compiled functions).
- If the template body uses an operation the actual type doesn't support (e.g., trying `T x + 1` when `T = std::string` without a matching `operator+`), you get a compile error **only when that specific instantiation is attempted** — not when the template itself is written. This is why template error messages are often long and confusing (they point into the instantiation).
- Templates can be instantiated explicitly too: `template double max<double>(double, double);` (rarely needed directly, but good to recognize the syntax).

## 11.8 Function templates with multiple template types
- `template <typename T> T max(T a, T b)` requires BOTH arguments to resolve to the identical `T` — `max(2, 3.5)` fails to deduce a single `T` (ambiguous / deduction failure), since one argument suggests `int` and the other `double`.
- Fix: `template <typename T, typename U> auto max(T a, U b) -> ...` with multiple template parameters (needs some way to determine a sensible common return type — e.g. `auto` return type deduction, or `std::common_type_t<T, U>`).
- Alternatively, just explicitly specify the template argument: `max<double>(2, 3.5)` forces both to `double`.

## 11.9 Non-type template parameters
- A template parameter that is a **value**, not a type: `template <int N> int increment() { return N + 1; }`, called via `increment<5>()`.
- Must be resolvable at **compile time** — valid categories include integral types, enumerations, pointers/references to objects with static duration, `std::nullptr_t`, and (since C++20) floating point and some class types.
- Common real use: `std::array<T, N>` — `N` is a non-type template parameter baking the array's fixed size directly into the type.

## 11.10 Templates in multiple files (a frequent "why won't this link" interview question)
- Because instantiation is a **compile-time, per-translation-unit** process, the compiler needs the FULL definition of the template (not just a declaration/prototype) visible in every `.cpp` file that instantiates it.
- If you split a template's declaration into a `.h` and its definition into a `.cpp` (the normal non-template pattern), any other `.cpp` file that `#include`s only the header and tries to use the template will fail to **link** ("undefined reference") — the compiler had no definition to instantiate against in that translation unit.
- **Fix / standard practice**: put the entire template definition (not just declaration) in the header file, so it's visible everywhere it's used. (Alternative, less common: explicit instantiation in one .cpp for only the types you know you'll need — brittle, rarely used outside libraries.)

---

# CHAPTER 12 — Compound Types: References and Pointers

## 12.1 Introduction / compound types overview
- **Compound types** are types built from other (fundamental or compound) types: functions, arrays, pointers, references, enums, classes/structs. This chapter covers references and pointers specifically.

## 12.2 Value categories: lvalues and rvalues
- Every expression in C++ has both a **type** and a **value category** — these are independent properties.
- **Lvalue**: an expression that evaluates to a function or object with an **identity** (an identifiable memory address / identifier you can refer to again). Subtypes:
  - **Modifiable lvalue**: can be changed (e.g. `int x; x` is a modifiable lvalue).
  - **Non-modifiable lvalue**: an lvalue whose value can't be changed — typically because it's `const` or `constexpr` (e.g. `const int y = 5; y` is a non-modifiable lvalue).
- **Rvalue**: an expression that is NOT an lvalue — includes literals (except string literals, which are lvalues due to how they're stored), and the return values of functions/operators **when returned by value** (temporaries with no persistent identity).
- Interview-relevant subtlety: a *named* variable declared as an rvalue reference (`int&& ref = 5;`) is itself treated as an **lvalue** when you use `ref` in an expression — value category depends on how the expression is written/used, not on the declared type. (`ref` has type `int&&` but is an lvalue expression.)

## 12.3–12.5 Lvalue references
- **Lvalue reference** (`int&`): an alias for an existing object; declared with `&` after the type.
- Must be **initialized at the point of declaration** — `int& r;` alone (no initializer) is a compile error.
- **Cannot be reseated**: once bound, `r = someOtherVar;` assigns `someOtherVar`'s VALUE into the referenced object — it does not rebind `r` to refer to `someOtherVar`.
- A plain (non-const) lvalue reference is sometimes explicitly called an **"lvalue reference to non-const"** — it can only bind to a **modifiable lvalue**. It cannot bind to:
  - A non-modifiable (const) lvalue (would let you mutate a const object through the back door).
  - An rvalue/temporary (would let you mutate something with no real persistent identity — nonsensical semantically, and disallowed by the standard).
- **Dangling reference**: occurs when the referenced object is destroyed while the reference to it still exists (e.g. the object was a local variable that went out of scope). Using a dangling reference is undefined behavior — this is a top interview "spot the bug" pattern.

## 12.6 Const lvalue references
- `const int& r = x;` — an **lvalue reference to const**: treats the referenced object as const through this reference (even if the underlying object is not actually const elsewhere).
- **Binding rules — a const reference can bind to:**
  1. A modifiable lvalue
  2. A non-modifiable (const) lvalue
  3. An **rvalue** (temporary) — this is the special case a plain reference cannot do.
- When a const reference binds to a temporary/rvalue, C++ extends that **temporary's lifetime to match the reference's lifetime** (so it isn't immediately destroyed at the end of the full expression as rvalues normally are) — this is precisely why `const int& r = 5;` is legal and safe, while `int& r = 5;` is a compile error.
- **Best practice takeaway**: prefer `const T&` parameters for expensive-to-copy types you don't need to modify — it's strictly more flexible than a non-const reference (accepts all 3 categories above) while avoiding a copy.

## 12.7–12.8 Pass by (lvalue) reference
- Passing arguments by reference avoids copying (efficient for large/expensive objects) and, for non-const references, allows the callee to modify the caller's actual object (**out parameter** usage — see 12.13).
- Because non-const reference parameters can only bind to modifiable lvalues, a function `void f(int&)` **cannot** be called with a literal or temporary (`f(5)` is a compile error) — only `const T&` or by-value parameters can.
- Trade-off: pass-by-reference means the function can (accidentally) modify the caller's data if you forget `const` — this is why "const by default unless you intend to modify" is standard advice.

## 12.9 Introduction to pointers
- A **pointer** is an object that holds a memory address as its value (typically the address of another object). Declared with `*`: `int* p;`
- **Address-of operator `&`**: returns the memory address of its operand (as a pointer).
- **Dereference operator `*`** (when applied to a pointer, not in a declaration): returns the value at the pointed-to address, as an lvalue — `*p = 5;` writes through the pointer.
- Pointers should always be initialized — an uninitialized ("wild") pointer holds a garbage address; dereferencing it is undefined behavior. Default-initialize with `nullptr` if there's nothing to point at yet.
- Pointer size is fixed by the platform (commonly 8 bytes on 64-bit systems) regardless of the pointed-to type — a `char*` and a `double*` are the same size; only the *interpretation* of dereferenced data differs.

## 12.10 Null pointers
- `nullptr` (C++11+): the modern, type-safe null pointer literal — replaces the old C-style `NULL` macro (which was often just `0`, ambiguous with integer `0` in overload resolution) and raw `0`.
- Dereferencing a null pointer is undefined behavior (commonly crashes — "null pointer dereference," one of the most common real-world C++ bugs).
- Checking: `if (p) { ... }` or `if (p != nullptr) { ... }` — pointers convert to `bool` (`true` if non-null).

## 12.11 Pointers and const — the four combinations (classic interview memorization point)
Read declarations **right to left from the variable name** to parse them correctly:
1. **Pointer to non-const**: `int* p;` — can reseat p, can modify `*p`.
2. **Pointer to const** (`const int* p;` or `int const* p;`): p can be reseated to point elsewhere, but `*p` cannot be modified through p.
3. **Const pointer (to non-const)**: `int* const p = &x;` — p cannot be reseated once initialized, but `*p` CAN be modified.
4. **Const pointer to const**: `const int* const p = &x;` — neither p can be reseated, nor can `*p` be modified through it.
- A pointer-to-const can point at a non-const object (the const-ness is a restriction on what you can do *through this pointer*, not a property of the pointee itself) — commonly used for read-only function parameters analogous to `const T&`.

## 12.12 Pass by address
- Pass a pointer so the function can access/modify the caller's object via that address, or represent "no object" (pass `nullptr`) — a form of **optional pass-by-reference**.
- Downsides vs pass-by-reference: caller must remember to take the address (`&x`), function must remember to dereference, and a pointer parameter can be null (must be checked) whereas a reference is guaranteed bound to something. Modern C++ prefers references (or `std::optional`) where nullability isn't genuinely needed.
- **Pass by address by reference** (`int*& p`) is possible (lets a function reseat the caller's actual pointer variable) but rare/advanced.

## 12.13 In and out parameters
- **In parameter**: function only reads it — pass by value (cheap types) or const reference (expensive types).
- **Out parameter**: function's job is to write a result back into it for the caller — pass by (non-const) reference or pointer; caller reads the value after the call returns.
- **In/out parameter**: passed by reference, the function reads the existing value AND writes a new one (e.g. `void increment(int& x) { x = x + 1; }`).
- Style convention: some codebases use pointer parameters specifically to signal "this is an out parameter" at call sites (since `f(&x)` visually flags that x's address is being passed, whereas `f(x)` with a hidden reference parameter isn't visible from the call site) — a documented tradeoff, not a hard rule.

## 12.14 Return by reference and return by address
- Returning by reference avoids copying the return value — but is only safe if the referenced object OUTLIVES the function call: e.g., a reference/pointer parameter passed in, a static/global variable, or a member of an object that persists after the call.
- **Never return a reference/pointer to a local variable** — it's destroyed when the function returns, leaving the caller with a dangling reference/pointer; using it is undefined behavior (this is arguably THE most common C++ "spot the bug" interview question).
- Returning by address (`T*`) has the same lifetime rules, plus the caller must null-check.

## 12.15 `auto` and pointers/references — type deduction subtleties
- `auto` alone **drops references and top-level const** from the deduced type: given `const int& getRef();`, `auto x = getRef();` deduces `x` as plain `int` (a new copy), not `const int&`.
- To preserve the reference/const-ness, be explicit: `auto& x = getRef();` or `const auto& x = getRef();`.
- `auto*` explicitly deduces a pointer type (and can be combined with `const`: `const auto* p = ...;`).

## (12.x extension) `std::optional`
- `std::optional<T>` (header `<optional>`) represents "a `T`, or nothing" without needing sentinel values (`-1`, `nullptr`) or pointers — increasingly preferred over pass-by-address for optional return values / parameters in modern code.
- Key members: `has_value()` / `operator bool()`, `value()` (throws `std::bad_optional_access` if empty), `value_or(default)`, `operator*`/`operator->` (unchecked access).

---

# CHAPTER 13 — Compound Types: Enums and Structs

## 13.1–13.2 Unscoped enumerations
- `enum Color { red, green, blue };` — defines a new **program-defined type** with a fixed set of named integral constants (**enumerators**).
- Enumerators are placed into the **surrounding scope** (unscoped) — `red`, `green`, `blue` become names directly usable without qualification, which risks **naming collisions** with other identifiers in the same scope (a key motivation for scoped enums, below).
- By default, the first enumerator is `0`, and each subsequent one is the previous value **+1**. You can explicitly assign values, and later enumerators without explicit values continue incrementing from the last explicit one:
```cpp
enum Color { red = 5, green, blue }; // green = 6, blue = 7
```
- Unscoped enumerators **implicitly convert to `int`** (their underlying integral type) — this is convenient (can be used in switch/arithmetic) but also a safety hole (accidentally comparing/mixing unrelated enum types via their integer values, or assigning arbitrary ints).
- Underlying type defaults to `int` (implementation-defined exact size) but can be specified: `enum Color : std::uint8_t { ... };` (also lets you forward-declare unscoped enums with a fixed base).

## 13.3 Unscoped enumerator integer conversions
- Converting an `int` (or other integral) to an unscoped enum is NOT implicit — must use `static_cast` (e.g., `static_cast<Color>(1)`), since not every integer value necessarily corresponds to a defined enumerator (undefined enumerator values are technically allowed but semantically meaningless without care).

## 13.4 Converting enums to/from strings
- No automatic reflection in C++ — you write this manually, typically with a `switch` returning a `std::string_view`, or a lookup table (e.g., `std::array` indexed by the enum's underlying value, or a `std::map<Color, std::string>`).
- For *input* (string → enum), similarly manual: compare against known string values in an `if`/`switch` chain and return/assign the matching enumerator, or fail (e.g., set a stream to failure state, or return `std::optional<Color>`).

## 13.5 Overloading I/O operators for enums (preview of Ch 21 mechanics, applied here)
- To make `std::cout << myColor` work, overload `operator<<` as a **non-member function** (`std::ostream` is the left operand, so it can't be a member of your enum/class):
```cpp
std::ostream& operator<<(std::ostream& out, Color c) {
    switch (c) { case red: out << "red"; break; /* ... */ }
    return out; // return the left operand by reference to allow chaining: cout << a << b;
}
```
- For `operator>>` (extraction into the enum), the enum parameter is an **out parameter** — passed by reference so the function can write the parsed value back into it. On invalid input, conventionally put the stream into failure state (mirrors how built-in extraction failure works) so callers can detect the error the same way they check any other failed `cin >>`.

## 13.6 Scoped enumerations (`enum class`)
- `enum class Color { red, green, blue };` — the modern, safer form.
- Enumerators are scoped **inside** the enum's own namespace: must qualify as `Color::red`, not bare `red` — eliminates naming collisions.
- **No implicit conversion to (or from) `int`** — must `static_cast<int>(Color::red)` explicitly. This prevents accidentally comparing/mixing enumerators from unrelated enum types and catches type errors at compile time.
- Otherwise behaves like an unscoped enum (underlying type, explicit values, etc.) — `enum class` is generally preferred in modern C++ for anything beyond quick internal use.

## 13.7–13.9 Structs — basics
- `struct` groups related variables (**data members**) under one type/name.
- Members accessed via `.` on an object, `->` on a pointer to an object (`ptr->member` is shorthand for `(*ptr).member`).
- **The only fundamental default difference between `struct` and `class` in C++ is default member/base-class access**: `struct` members and inheritance default to `public`; `class` defaults to `private`. Both support member functions, constructors, access specifiers, inheritance, etc. — the choice is a style/semantic convention (struct for passive data aggregates, class for encapsulated objects with invariants), not a hard technical restriction.

## 13.10 Aggregate initialization
- A **struct/class is an aggregate** if it has: no user-declared/provided constructors, no private or protected non-static data members, no virtual functions, and (pre-C++17) no base classes (C++17 relaxed this to allow public, non-virtual base classes for aggregates).
- Aggregates can be initialized with a brace-init list matching member declaration order: `Point p{1, 2, 3};` — this directly initializes each member in order, no constructor call involved.
- Missing trailing initializers value-initialize (zero for fundamental types) the remaining members: `Point p{1};` sets the first member to 1, the rest to 0/default.
- **Designated initializers** (C++20): `Point p{.x = 1, .y = 2};` — must follow declaration order, can skip members (they get default/zero-init).

## 13.11 Default member initializers
- `struct Point { int x = 0; int y = 0; int z = 0; };` — provides a default value used when aggregate-init doesn't explicitly supply one for that member (works together with 13.10 — if you don't override x with an initializer, it falls back to its default member initializer rather than plain zero-init, if one is specified).

## 13.12 Passing and returning structs
- Passing: use **pass by (const) reference** for non-trivial structs to avoid copying, exactly the same reasoning as with classes (Ch 12) — pass by value only for small, cheap structs.
- Returning: return by value is idiomatic and safe (no dangling-reference risk) — the compiler applies **copy elision** (see Ch 14) in most cases, so there's typically no actual copy overhead despite the "by value" syntax.

## 13.13–13.15 Struct miscellany, class templates, CTAD
- **Struct member selection with pointers/references chains** (`obj.member.submember`, `ptr->member`) works exactly as expected, composing normally.
- **Class templates** applied to structs: `template <typename T> struct Pair { T first; T second; };` — generic aggregate type; instantiated like any other template on first use with a concrete type.
- **CTAD — Class Template Argument Deduction** (C++17+): lets you write `Pair p{1, 2};` instead of `Pair<int> p{1, 2};` — the compiler deduces the template argument(s) from the initializer, using either implicitly-generated deduction guides (from constructors, or aggregate members) or explicit user-provided **deduction guides** (`template <typename T> Pair(T, T) -> Pair<T>;`) when the implicit ones aren't sufficient.
- **Alias templates**: `template <typename T> using Vec = std::vector<T>;` — creates a template for a type alias, letting you parametrize an alias just like a class.

---

# CHAPTER 14 — Introduction to Classes

## 14.1–14.2 OOP and class basics
- **Object-oriented programming (OOP)**: models data together with the operations on that data as self-contained **objects**, rather than keeping data and functions that operate on it entirely separate (the older "procedural" style).
- Benefits typically cited: better modeling of real-world/domain concepts, encapsulation (controlled access), easier maintenance and reuse, clearer ownership of behavior.
- `class` keyword defines a new program-defined type combining **data members** (variables) and **member functions** (methods).
- A member function is called on an object using `.`/`->` and implicitly operates on that specific object's data (via the hidden `this` pointer, Ch 15).

## 14.3 Member functions
- Declared/defined inside the class body (or declared inside, defined outside using `ClassName::functionName`).
- Every non-static member function implicitly receives a pointer to the calling object as its first "hidden" parameter (`this`, formalized in Ch 15) — this is how `obj.getX()` knows which object's `x` to return.

## 14.4 Const member functions
- `int getX() const;` — the `const` after the parameter list applies to the **implicit object** (`this` is effectively `const ClassName*` inside this function) — the function promises not to modify any non-mutable member.
- **Const-correctness matters for overload resolution and callability**: a `const` object (or const reference/pointer to an object) can only have its `const` member functions called on it — attempting to call a non-const member function on a const object is a compile error.
- A class can have BOTH a const and non-const overload of the same-named function (differentiated by the const qualifier, per 11.2) — classic use: `T& operator[](size_t i); const T& operator[](size_t i) const;` so subscripting works correctly whether the container itself is const or not.
- `mutable` members are exempt from a const member function's modification restriction — used sparingly, e.g. for caching or internal counters that don't represent the object's "logical" state.

## 14.5 Public and private access specifiers
- `public:` — accessible from anywhere the object is visible.
- `private:` — accessible only from within the class's own member functions (and friends, 15.8).
- `protected:` — like private, but also accessible from derived classes (relevant starting Ch 24).
- Default access: `private` for `class`, `public` for `struct` (this is literally the only default difference between the two keywords in C++).
- Multiple access-specifier blocks can appear in any order/repeated in a class body; each subsequent member uses whichever specifier appeared most recently above it.

## 14.6 Access functions
- "Getters" (accessors) and "setters" (mutators) — public member functions providing controlled, indirect access to private data.
- Purpose (a classic interview theory question): lets the class enforce **invariants** (e.g. a setter can validate/clamp a value before storing it) and lets the internal representation change later without breaking code that depends on the class's public interface.
- Naming convention varies (`getX()`/`setX()`, or `x()`/`setX()`); not standardized by the language.

## 14.7 Member functions returning references to data members — brief note
- A getter can return by reference (`const T& getX() const { return m_x; }`) to avoid a copy for expensive-to-copy members — but must be careful the returned reference doesn't outlive the object (same dangling-reference rules as Ch 12).

## 14.8 The benefits of encapsulation (frequently asked theory question, memorize the 3 points)
1. **Easier to use correctly, harder to use incorrectly** — encapsulated classes provide a clean interface, hiding messy implementation details from callers.
2. **Helps protect data and prevent misuse** — invariants can be enforced centrally (via setters/constructors) instead of trusting every caller to keep data valid.
3. **Easier to change the internal implementation without breaking existing code that uses the class** — as long as the public interface (function signatures/behavior) stays the same, you can freely rewrite internals (e.g. change from an array to a `std::vector`) without touching calling code.

## 14.9 Introduction to constructors
- A **constructor** is a special member function, same name as the class, **no return type** (not even `void`), automatically invoked when an object of that type is created.
- Purpose: initialize the object's members to a valid, consistent state at creation.
- **Default constructor**: a constructor callable with no arguments (either genuinely zero parameters, or all parameters have default values).
- **Implicitly-generated default constructor**: if you declare **no constructors at all**, the compiler auto-generates a public default constructor that default-initializes members (calling default constructors for class-type members; leaving fundamental-type members uninitialized unless they have default member initializers). The moment you declare ANY constructor yourself, this auto-generation stops — you must explicitly provide a default constructor yourself if you still want one.

## 14.10 Constructor member initializer lists
- Syntax: `Point(int x, int y) : m_x(x), m_y(y) { }` — the part after `:` and before `{` is the **member initializer list**.
- **Strongly preferred over assigning inside the constructor body** because:
  - Members in the init list are **directly initialized** (constructed with that value from the start) rather than default-constructed and then reassigned — more efficient, especially for class-type members with non-trivial default constructors.
  - It's the **only** way to initialize `const` members, reference members, and members of a class type with no default constructor — these CANNOT be assigned to after default-construction, only initialized.
- **Order of execution is ALWAYS the members' declaration order in the class**, regardless of the order written in the initializer list. Writing the list out of order doesn't change execution order — it just makes the code misleading, and most compilers will warn (`-Wreorder` in GCC/Clang). If a later-declared member's initializer depends on an earlier-declared member (in the *list's* order but not the *class's* order), it may read an uninitialized value — a genuine bug, not just a style nit.

## 14.11 Non-static member initialization (default member initializers)
- Members can have a default value specified directly in the class definition: `int m_x = 0;` — used when a constructor's initializer list doesn't explicitly initialize that member, similar to struct default member initializers (Ch 13).

## 14.12 Delegating constructors
- One constructor can call another constructor of the same class from its member-initializer list: `Point() : Point(0, 0) { }` — avoids duplicating initialization logic across multiple constructors.
- **Constraint**: if a constructor delegates to another constructor, it **cannot also have other members in its initializer list** — the delegated-to constructor is solely responsible for member initialization; the delegating constructor's body can still contain additional logic that runs after the delegated constructor completes.

## 14.13 Temporary class objects
- Unnamed ("anonymous") objects created for the duration of a single expression, e.g. `Point(1,2).getX()` — the `Point(1,2)` temporary is constructed, used, and destroyed within that one expression/statement.

## 14.14 Copy constructors
- `Point(const Point& src);` — invoked whenever an object is copy-initialized from another object of the same type: initialization with `=` syntax at declaration, passing by value, returning by value (in some cases, subject to elision), and explicit copy syntax.
- If not user-defined, the compiler generates an **implicit memberwise copy constructor** — copies each member individually using ITS OWN copy constructor/copy semantics (fine for simple value-holding classes; dangerous/wrong for classes managing a raw resource like heap memory — see Rule of Three, Ch 21).
- You can explicitly disable copying: `Point(const Point&) = delete;` (used e.g. by `std::unique_ptr` to enforce exclusive ownership).

## 14.15 Copy elision
- **Copy elision**: the compiler is permitted — and in specific cases (C++17+) *required* — to skip constructing a temporary and then copying/moving it, instead constructing the object directly in its final destination.
- Mandatory case (C++17+): `return Point{1, 2};` where the returned expression is a **prvalue** of the same type as the function's return type — no copy/move constructor call happens at all, even conceptually (this is stronger than the pre-C++17 "Return Value Optimization" which was merely *permitted*).
- Practical consequence / interview gotcha: don't rely on side effects (like a print statement) inside a copy constructor firing a specific number of times — elision can (and often must) skip those calls entirely, even though the copy constructor is syntactically "used" in the code.

## 14.16 Converting constructors, explicit, and copy initialization
- A **converting constructor** is any non-explicit constructor that CAN be called with a single argument (whether it has exactly one parameter, or more with defaults for the rest) — it defines an implicit conversion from that argument's type to the class type.
```cpp
class Point { public: Point(int x) { ... } };
void f(Point p);
f(5); // 5 implicitly converted to Point via the converting constructor
```
- This can cause subtle bugs (unintended implicit conversions happening silently). Fix: mark the constructor `explicit`:
```cpp
class Point { public: explicit Point(int x) { ... } };
f(5);          // now a compile error
f(Point(5));   // must be explicit
f(static_cast<Point>(5)); // also fine, this is what static_cast invokes
```
- Best practice (frequently tested): make single-argument constructors `explicit` unless you specifically WANT implicit conversion behavior (e.g. sometimes desired for wrapper/proxy types).

## 14.17 Constexpr classes / aggregates
- Classes usable in constant expressions (`constexpr` contexts) if: all data members are of literal types, and the relevant constructor is itself `constexpr` (implicitly the case for simple aggregates, or explicitly declared for classes with user-provided constructors).
- Relevant for compile-time computation questions (e.g. building lookup tables / constant data at compile time using class types, not just fundamental types).

---

# CHAPTER 15 — More on Classes

## 15.1 The hidden `this` pointer
- Every non-static member function secretly receives an extra first parameter: a pointer to the object it was called on, named `this` (type `ClassName* const` — the pointer itself is const, i.e. can't be reseated within the function, though what it points to can be modified unless the function is also `const`, in which case `this` is `const ClassName* const`).
- Inside a member function, `m_x` and `this->m_x` refer to the same thing — the compiler implicitly inserts `this->` for unqualified member access.
- **Method chaining**: returning `*this` (by reference, `ClassName&`) from a member function lets calls be chained fluently:
```cpp
Employee& setName(std::string n) { m_name = n; return *this; }
Employee& setAge(int a) { m_age = a; return *this; }
// usage: emp.setName("Alice").setAge(30);
```

## 15.2 Classes and header files
- Standard organization: class **definition** (member variable declarations + member function declarations, and definitions of any trivial/inline functions) goes in a `.h` header; **non-trivial member function definitions** go in a matching `.cpp` file, qualified with `ClassName::functionName(...)`.
- Requires **include guards** (`#ifndef`/`#define`/`#endif`) or `#pragma once` in the header to prevent multiple-definition errors when the header is included in multiple translation units.
- **Exception**: template classes/functions must have their full definitions in the header (per Ch 11.10's reasoning) — they can't be split into a separate .cpp the normal way.

## 15.3 Nested types (member types)
- A class/struct/enum can be defined **inside** another class: e.g. `class Fruit { public: enum class Type { apple, banana }; ... };` — accessed as `Fruit::Type::apple`.
- Useful for tightly-coupled helper types that only make sense in the context of the containing class, keeping the global/enclosing namespace clean.
- Nested types have access to the outer class's private members if defined appropriately, and can be used before other class members are declared (unlike regular members, which generally must be declared before use within the same class — nested type declarations are looked up as a special case).

## 15.4 Destructors
- `~ClassName()` — a special member function, same name as class prefixed with `~`, **no parameters, no return type**, and a class can have **only one** destructor (no overloading destructors).
- Automatically invoked when an object's lifetime ends: going out of scope (automatic/local objects), explicit `delete` (dynamically allocated objects), or program termination for global/static objects.
- Used for **cleanup**: freeing dynamically allocated memory, closing file handles, releasing locks, etc. — this is the mechanical foundation of **RAII (Resource Acquisition Is Initialization)**: acquire a resource in the constructor, release it in the destructor, so resource lifetime is automatically tied to object lifetime and can't be "forgotten."
- If a class manages a resource (e.g. holds a raw pointer to heap-allocated memory) but you rely on the compiler's implicitly-generated destructor (which does nothing but call member destructors), that memory leaks. This ties directly into the **Rule of Three** (Ch 21): a custom destructor almost always implies you also need a custom copy constructor and copy-assignment operator, since the compiler-generated versions of those would do a shallow copy of the resource, leading to double-free or use-after-free.

## 15.5 (class invariants — brief)
- An **invariant** is a condition that must remain true throughout an object's lifetime for it to be in a valid state (e.g. a `Fraction` class's denominator must never be 0). Constructors and mutating member functions are responsible for enforcing invariants; well-encapsulated classes make it hard/impossible for external code to violate them directly.

## 15.6 Static member variables
- `static` on a data member: **one single copy shared across ALL instances** of the class, rather than a separate copy per object.
- Declared inside the class, but (pre-C++17, for non-const/non-inline statics) must be **defined exactly once outside the class**, typically in the matching `.cpp` file: `int Employee::s_count = 0;` — otherwise you get a linker error ("undefined reference").
- **C++17+**: `inline static` members can be defined directly inside the class body without a separate out-of-class definition (`inline static int s_count = 0;`) — cleaner, avoids the linker-definition dance. `static constexpr` members are also typically fine to define in-class since C++17 relaxes the rules for them.
- Classic use case: tracking the number of currently-live instances of a class (increment in constructor, decrement in destructor).
- Accessible via an instance (`obj.s_count`) OR via the class name directly (`Employee::s_count`) — the latter is preferred/clearer since it emphasizes the value isn't tied to any one object.

## 15.7 Static member functions
- `static` on a member function: the function has **no implicit `this`** and isn't associated with any specific object — callable directly via the class name: `Employee::getCount();`
- Because it has no `this`, a static member function **cannot directly access non-static members** (there's no object to reference) — it can only access other static members directly (or operate on an object explicitly passed to it as a parameter).
- Often paired with static member variables to provide a controlled public interface to shared class-level state (e.g. a static "get total instance count" accessor for a private static counter).

## 15.8 Friend functions and classes
- `friend` grants a specific non-member function, or an entire other class, direct access to a class's `private`/`protected` members — deliberately punching a hole in encapsulation for cases where two things are tightly logically coupled but can't naturally be a single class (e.g., `operator<<` overloads for a class, which must be non-member functions since `std::ostream` is the left operand — see Ch 21).
- Declared with `friend` inside the class granting access: `friend std::ostream& operator<<(std::ostream&, const Point&);` or `friend class OtherClass;`
- **Friendship is NOT mutual**: if class A declares class B a friend, B can access A's private members, but A does NOT automatically get access to B's private members (unless B separately also declares A a friend).
- **Friendship is NOT inherited/transitive**: if B is a friend of A, and C derives from B, C does not automatically inherit B's friendship with A. Similarly, being a friend of a friend doesn't confer access.
- Overuse of `friend` undermines encapsulation — use sparingly, generally limited to closely related helper functions/operator overloads.

## 15.9 Anonymous/unnamed and inline classes/namespaces — brief mention
- (Lightly covered territory around this point in the chapter sequence: nested/local class considerations, and using classes purely locally within a function — niche, low interview priority, but recognize that classes CAN be defined locally inside a function body, with restrictions like not being usable as template arguments in some contexts pre-C++11.)

## 15.10 Ref-qualified member functions (advanced, low frequency in interviews but worth recognizing)
- A member function can be **ref-qualified** with `&` or `&&` after its parameter list (and after any `const`), restricting which value category of object it can be called on:
```cpp
class T {
public:
    void f() &;   // only callable on lvalue objects
    void f() &&;  // only callable on rvalue objects (temporaries)
};
```
- Rare in typical interview/OA code, but useful to recognize if seen — mainly used in library-quality code to prevent misuse of a method on a temporary that's about to be destroyed (or conversely, to allow move-optimized overloads specifically for temporaries).

---

## Quick Cross-Chapter Traps to Memorize Before an OA/Interview

| Topic | The trap |
|---|---|
| Overload resolution | Return type never differentiates; typedef/alias never differentiates; top-level const on by-value params never differentiates |
| Ambiguous overloads | Numeric conversions (step 3) tie easily — e.g. `int` → `unsigned int` vs `int` → `float` |
| References | Can't be reseated; non-const ref can't bind an rvalue; const ref CAN (and extends the temporary's life) |
| Returning by reference | Never return a reference/pointer to a local variable — dangling |
| Pointer const combos | Read right-to-left: `const int* const p` = const pointer, const pointee |
| Aggregate init | Any user-declared constructor disqualifies a type from being an aggregate |
| Member-init-list order | Always follows declaration order, NOT list-written order |
| Copy elision | May skip copy/move ctor calls entirely (mandatory for prvalues since C++17) — don't rely on ctor side effects |
| `explicit` | Prevents unwanted single-argument implicit conversions |
| Destructors | No parameters, no overloading, only one per class; pairs with Rule of Three if managing a resource |
| Static members | One copy total, shared across instances; static functions have no `this` |
| `friend` | Not mutual, not inherited/transitive |
| Templates | Must be fully defined in headers, not split into .cpp (compile-time, per-TU instantiation) |
