# C++ Deep-Dive Notes — Chapters 16–20 (LearnCpp.com)
### Exhaustive, interview/OA-oriented — every lesson, every gotcha

---
# CHAPTER 16 — Dynamic Arrays: `std::vector`

## 16.1 Introduction to containers and arrays
- A **container** is a data type that holds a collection of other objects (its **elements**), providing storage/management and typically supporting operations like insertion, removal, and access. An **array** is a specific kind of container that stores its elements **contiguously in memory** and supports fast, direct access via a numeric **index/subscript**.
- C++ has THREE array-like types you'll encounter across Ch 16–17: `std::vector` (dynamic/resizable, this chapter), `std::array` (fixed-size, compile-time-known length, Ch 17), and **C-style arrays** (the inherited-from-C fixed-size array, also Ch 17) — all three are contiguous and subscriptable, differing mainly in resizability, safety, and modernity.

## 16.2 Introduction to `std::vector` and list constructors
- `std::vector<T>` (header `<vector>`) is the standard **dynamic array** — it can grow/shrink at runtime, manages its own memory automatically (RAII — no manual `new`/`delete` needed), and is generally the DEFAULT choice for "I need an array" in modern C++.
- Construction:
```cpp
std::vector<int> v1;            // empty vector
std::vector<int> v2{ 1, 2, 3 }; // list-init: 3 elements with values 1, 2, 3 — calls the initializer_list constructor
std::vector<int> v3(5);         // 5 default-initialized (zeroed) elements — direct-init, NOT the list ctor
std::vector<int> v4(5, 100);    // 5 elements, each initialized to 100
```
- **Critical gotcha (frequently tested)**: `std::vector<int> v{5};` and `std::vector<int> v(5);` mean COMPLETELY DIFFERENT things — brace-init `{5}` prefers the `initializer_list` constructor (a vector holding ONE element, value 5), while paren-init `(5)` calls a different constructor meaning "5 default-constructed elements." This is a direct consequence of the general rule (Ch 23.7 / 13.10) that braced-init prefers an `initializer_list` constructor over other viable constructors when one exists.
- CTAD (Ch 13.14) works with `std::vector`: `std::vector v{ 1, 2, 3 };` deduces `std::vector<int>` automatically.

## 16.3 `std::vector` and the unsigned length/subscript problem (a genuinely famous C++ design wart — very commonly discussed in interviews)
- `std::vector::size()` (and the `size()` of essentially every standard container) returns a nested typedef `size_type`, which for `std::vector` is (in practice, always) an alias for **`std::size_t`** — a large, UNSIGNED integral type. `operator[]` and `.at()` likewise expect their index parameter to be of this unsigned `size_type`.
- **Historical reasoning behind this design choice (worth knowing for a "why" interview question)**: made circa 1997 by the STL designers — negative indices are logically meaningless, unsigned types have one extra bit of positive range (mattered more in the 16-bit era), and unsigned indices only need ONE bounds check (`index < size`) instead of two (`index >= 0 && index < size`). **This is now broadly considered to have been a mistake** — even Bjarne Stroustrup has publicly said so — because it forces awkward signed/unsigned mixing throughout ordinary code.
- **The concrete problem**: if you use a normal (signed) `int` as your loop index/counter and pass it to `operator[]`/`.at()`, the compiler must perform an implicit **sign conversion** (int → `size_t`) — this is technically classified as a **narrowing conversion**, and the compiler will (or should) emit a warning about it at every single subscript operation, quickly flooding your build log with noise.
- **A genuinely dangerous variant of this problem**: iterating a vector backward using an unsigned index:
```cpp
for (std::size_t index{ arr.size() - 1 }; index >= 0; --index) { /* ... */ }
```
  This is a classic **infinite loop / out-of-bounds bug**: `index >= 0` is ALWAYS true for an unsigned type (it can never go negative — decrementing past 0 **wraps around** to the type's maximum value instead, e.g. `std::size_t(-1)` becomes a huge positive number) — so the loop never terminates normally and will index far out of bounds, undefined behavior. This exact pattern is a favorite "spot the bug" interview/OA question.
- **Mitigation strategies (all discussed, each with tradeoffs)**:
  1. Use `size_type` (or `auto`) explicitly for your loop variable to stay consistent with the container's own type.
  2. Use `.data()` (returns a raw C-style pointer to the underlying contiguous storage) and subscript THAT instead — C-style array/pointer subscripting accepts BOTH signed and unsigned indices without any sign-conversion warning, since it's just pointer arithmetic under the hood.
  3. `static_cast` your signed index to `size_type`/`size_t` explicitly at each use (technically correct, but clutters code).
  4. Use C++20's `std::ssize()` (a free function returning the container's length as a large SIGNED type, typically `std::ptrdiff_t`) instead of `.size()`/`std::size()` when you want a signed length to work with.
  5. Use unscoped enumerators as indices — since enumerators are implicitly `constexpr`, a **constexpr** conversion from signed to `size_t` is NOT considered narrowing (narrowing-conversion rules specifically exempt compile-time-constant conversions that are known to fit) — so indexing with an unscoped enum value avoids the warning cleanly (careful: this only holds for genuinely `constexpr` enumerator use, not a non-const runtime variable of enum type).
- Best-practice takeaway most consistently recommended: prefer using `.data()`-based indexing with signed loop variables when doing manual index arithmetic (especially reverse iteration), since it avoids sign-conversion warnings AND avoids the wraparound bug entirely, at very little readability cost.

## 16.4 Passing `std::vector`
- Pass by **`const std::vector<T>&`** for read-only access (avoids an expensive full-container copy) — the standard, default-preferred approach.
- Pass by **non-const reference** (`std::vector<T>&`) specifically when the function needs to MODIFY the caller's vector in place.
- Passing by value copies the ENTIRE underlying dynamic array — expensive for anything beyond trivially small vectors; avoid unless you specifically need an independent, mutable local copy.
- Function templates can be written generically over the ELEMENT type: `template <typename T> void printVector(const std::vector<T>& v)`.

## 16.5 Returning `std::vector`, and an introduction to move semantics
- Returning a `std::vector` **by value** is idiomatic and (for move-capable types like `std::vector`) CHEAP — because a returned local variable is an rvalue, and `std::vector` supports move semantics (Ch 22), the compiler can MOVE the vector's internal buffer into the destination instead of deep-copying it, making return-by-value inexpensive (just a few pointer reassignments, not an O(n) element copy).
- Never return a vector by reference/pointer to a LOCAL vector — same dangling-reference rule as everywhere else (Ch 12.14) — the local is destroyed when the function returns.
- Consequently: expensive-to-copy, move-capable types like `std::vector`/`std::string` should still be PASSED by const reference (avoids the copy — there's nothing to "move from" when passing an existing caller-owned lvalue), but can be safely and cheaply RETURNED by value (the returned object is fresh/temporary — a move applies naturally). This asymmetry ("pass by reference, but return by value") is a frequently misunderstood point worth being able to explain clearly.
- Copy elision (Ch 14.15) may apply here too, potentially skipping even the move entirely in many cases — either way, returning containers by value is efficient in modern C++ and should not be avoided out of old "return by value is always expensive" instincts.

## 16.6–16.7 Arrays, loops, and sign challenge solutions
- Iterating a `std::vector` with a classic indexed `for` loop requires care around the sign issues from 16.3 — covered in depth above; 16.6/16.7 are largely dedicated to walking through those exact patterns (forward iteration is generally safe with either signed or unsigned; BACKWARD iteration with an unsigned index is the dangerous case).

## 16.8 Range-based for loops (for-each)
```cpp
for (const auto& element : vec) { /* read-only, no copy */ }
for (auto& element : vec) { /* can modify elements in place */ }
for (auto element : vec) { /* copies each element — fine for cheap types, wasteful for expensive ones */ }
```
- Range-based for loops SIDESTEP the sign-conversion/indexing problem entirely (16.3) — no manual index variable is involved at all, so this is the PREFERRED iteration style whenever you don't specifically need the index value itself.
- Use `const auto&` by default (avoids copying each element, prevents modification); use `auto&` specifically when you need to modify elements in place; use `auto` (by value) only for small, cheap-to-copy element types where you don't need a reference.

## 16.9 Array indexing and length using enumerators
- Covered above under 16.3's mitigation list — using UNSCOPED enum values as indices works cleanly (implicit `constexpr` conversion to `size_t`, non-narrowing); `enum class` values do NOT implicitly convert and would need an explicit `static_cast`.
- A common idiom: define an extra trailing enumerator (e.g. `max_students`) whose auto-incremented value conveniently equals the COUNT of the "real" enumerators before it — useful for sizing a fixed array/vector to exactly match an enum-labeled set of named indices, and for `static_assert`/`assert` checks that the array's length still matches the enum's count after future edits.

## 16.10 `std::vector` resizing and capacity
- **`size()`** = the number of elements CURRENTLY held (logical length). **`capacity()`** = the number of elements the vector can hold in its CURRENTLY allocated buffer before it needs to reallocate — capacity is always ≥ size.
- `resize(n)`: changes the vector's actual SIZE — adds default-constructed elements if growing, destroys/removes elements if shrinking. May trigger a reallocation if `n > capacity()`.
- `reserve(n)`: changes only the CAPACITY (pre-allocates buffer space) WITHOUT changing size or constructing any new elements — a pure performance optimization to avoid multiple reallocations when you know in advance you'll be adding many elements (e.g. via repeated `push_back`).
- **Reallocation mechanics**: when a vector must grow beyond its current capacity, it typically allocates a NEW, LARGER buffer (commonly ~1.5x–2x the old capacity, implementation-defined growth factor), moves/copies all existing elements into the new buffer, and frees the old one — this makes `push_back` **amortized O(1)** per call (occasional expensive reallocations, but rare enough that the AVERAGE cost per insertion stays constant) even though any INDIVIDUAL reallocating call is O(n).
- **Iterator/pointer/reference invalidation**: any operation that can trigger reallocation (`push_back`, `resize`, `reserve` when growing beyond current capacity, `insert`, etc.) INVALIDATES all existing iterators, pointers, and references into the vector's elements — a classic, frequently-tested source of subtle bugs (e.g. holding a reference to `vec[0]` across a `push_back` call that happens to trigger reallocation, then using that now-dangling reference).
- `shrink_to_fit()`: a (non-binding) request to reduce capacity down to match the current size, releasing unused reserved memory.

## 16.11 `std::vector` and stack behavior
- `std::vector` naturally supports efficient **LIFO (Last In, First Out) stack-like usage** via:
  - `push_back(value)` — appends to the end (amortized O(1), per 16.10).
  - `pop_back()` — removes the last element (O(1)) — note: does NOT return the removed value; retrieve it via `.back()` first if needed.
  - `back()` — accesses (peeks) the last element without removing it.
  - `empty()` — checks whether the vector currently has zero elements.
- The standard library also provides a dedicated **`std::stack`** adapter (built on top of `std::vector`/`std::deque` by default) offering exactly this LIFO-restricted interface (`push`, `pop`, `top`, `empty`, `size`) when you want to communicate stack-only usage intent explicitly rather than exposing the full `std::vector` interface.

## 16.12 `std::vector<bool>`
- `std::vector<bool>` is a **special-cased partial template specialization** in the standard library — it does NOT store actual `bool` elements in a normal contiguous array; instead, it uses a **bit-packed representation** (1 bit per logical element, 8x more memory-dense than an array of real `bool`s, which typically take a full byte each).
- **Consequence**: `operator[]` on a `std::vector<bool>` does NOT return an actual `bool&` (you can't take a real reference/address to a single bit) — it returns a **proxy object** (`std::vector<bool>::reference`) that mimics reference-like behavior but is a distinct, special type. This breaks certain generic code that expects `T&` from `operator[]`, and is widely considered one of the standard library's more notorious design mistakes — frequently cited in "what's a surprising C++ gotcha" interview discussions. If you need a true, ordinary bit-addressable-like array of booleans without this quirk, alternatives like `std::deque<bool>` or `std::bitset` are sometimes recommended instead, depending on the actual need.

---

# CHAPTER 17 — Fixed-Size Arrays: `std::array` and C-Style Arrays

## 17.1–17.2 `std::array` — introduction, length and indexing
- `std::array<T, N>` (header `<array>`) is a **fixed-size** array wrapper — `N` (the length) is a **non-type template parameter** (Ch 11.9/13.13) fixed at COMPILE time, baked directly into the type itself, so `std::array<int, 5>` and `std::array<int, 10>` are genuinely different types.
- Unlike `std::vector`, `std::array` has **NO dynamic resizing capability whatsoever** — its size is permanently fixed once the type is chosen. In exchange, it avoids heap allocation entirely (`std::vector` always heap-allocates its buffer; `std::array`'s storage typically lives directly wherever the `std::array` object itself lives — stack, or embedded in another object).
- **Fully supports `constexpr`** (unlike `std::vector`, which as of recent standards still generally cannot be used in `constexpr` contexts) — `std::array` is the array type to reach for whenever compile-time array usage is needed.
- Same unsigned `size_type`/sign-conversion considerations as `std::vector` (Ch 16.3) apply identically here, EXCEPT: since `std::array`'s length `N` must itself be a `constexpr` value, converting a signed constexpr value to `size_t` for that `N` is NOT a narrowing conversion (compile-time-constant conversions that provably fit aren't narrowing) — so declaring the array's SIZE with a signed constant is fine; it's specifically INDEXING with a non-constexpr signed runtime variable that still triggers the same narrowing-conversion warning as with `std::vector`.
- CTAD works: `std::array arr{ 1, 2, 3 };` deduces `std::array<int, 3>`.

## 17.3 Passing and returning `std::array`
- Pass by `const std::array<T, N>&` for read-only access — same reasoning as `std::vector`, and additionally: because the LENGTH is part of the type, a function templated to accept `std::array<T, N>&` generically must template over BOTH `T` and `N` if it needs to work with arrays of varying lengths.
- Because `std::array` doesn't own heap memory, it CAN be safely returned by value with genuinely low overhead relative to a C-style array (no manual copying loop needed, and it directly supports copy/move semantics as a regular class type) — though for very large N, this is still a real (stack) copy, unlike `std::vector`'s cheap pointer-move return.

## 17.4 `std::array` of class types, and brace elision
- `std::array<SomeClass, N>` works fine for holding class-type elements, following the same aggregate-initialization rules (Ch 13.10) as any other aggregate.
- **Brace elision**: because `std::array` is itself an aggregate wrapping an inner C-style array member, you're technically nesting braces (`std::array<int,3> a{ {1,2,3} };`), but C++ permits OMITTING the inner braces in the common case (`std::array<int,3> a{1,2,3};`) — a syntax convenience worth recognizing so nested-brace-looking code doesn't seem mysterious.

## 17.5 Arrays of references via `std::reference_wrapper`
- **C-style arrays, `std::array`, and `std::vector` cannot directly hold actual reference-typed elements** — references aren't reassignable/rebindable (Ch 12.5) and don't support the normal object semantics (default construction, assignment) containers rely on internally.
- **`std::reference_wrapper<T>`** (header `<functional>`) is the standard workaround — a small, copyable, reassignable class that WRAPS a reference, letting you store "reference-like" behavior inside ordinary containers: `std::array<std::reference_wrapper<int>, 3> arr{ x, y, z };`
- Convenience helper: `std::ref(x)` / `std::cref(x)` construct a `reference_wrapper` (mutable/const respectively) more concisely than spelling out the template explicitly.
- Access the underlying referenced object via `.get()`, or often implicitly via `reference_wrapper`'s own implicit conversion operator back to `T&`.

## 17.6 `std::array` and enumerations
- Combines Ch 16.9's enum-indexing idea with `std::array` specifically — a common, idiomatic C++ pattern for a fixed lookup table indexed by a meaningful enum (e.g. `std::array<std::string_view, Colors::max_colors> colorNames{ "red", "green", "blue" };`), giving compile-time-sized, compile-time-indexable, self-documenting tables.

## 17.7 Introduction to C-style arrays
- The array type inherited directly from C — fixed size, declared as `int arr[5];`, with the length typically needing to be a compile-time constant for a stack-allocated array.
- **No built-in `.size()`/`.length()` member function** (it's not a class at all, just raw contiguous memory) — length must be tracked separately or computed via `sizeof(arr) / sizeof(arr[0])` (fragile — only works correctly while `arr` is still a genuine array type, NOT after it has decayed to a pointer, see 17.8) or via `std::size()` (C++17, works correctly on C-style arrays specifically because the array's TYPE, including its length, is still known at that call site).
- Supports **aggregate initialization** (`int arr[3] = {1, 2, 3};`), can be declared `const`/`constexpr`, and its length can be OMITTED if an initializer list is provided (`int arr[] = {1,2,3};` — compiler infers length 3 from the initializer).
- Generally **discouraged in favor of `std::array`/`std::vector`** in modern C++ — no bounds checking, easy to accidentally decay to a pointer and lose length information (17.8), no member functions, awkward to pass/return correctly.

## 17.8 C-style array decay
- **Array decay**: in most expression contexts, a C-style array **implicitly converts ("decays") into a pointer** to its first element (`T*`), LOSING all information about its actual length in the process. This happens, for instance, whenever a C-style array is passed as a function argument to a parameter declared as a plain pointer/incomplete array-syntax parameter.
```cpp
void printSize(int arr[]) {           // this is secretly just `int* arr` — arr has DECAYED
    std::cout << sizeof(arr);         // prints sizeof(int*) — e.g. 8 — NOT the array's real byte size!
}
```
- **This is one of the single most classic C++ interview/OA gotchas**: `sizeof` on an actual array variable gives the array's TOTAL byte size; `sizeof` on a decayed pointer (including any array PARAMETER, since function parameters declared with array syntax are secretly just pointers) gives only the POINTER's size (commonly 8 bytes on 64-bit systems) — a function receiving "an array" parameter genuinely has NO way to determine that array's length from the parameter alone; the length must be passed separately as an explicit additional parameter.
- `std::array`/`std::vector` do NOT suffer from this problem — as proper class types, they carry their own length information (member function/member variable) regardless of how they're passed, which is a major practical reason they're preferred over C-style arrays in modern code.

## 17.9 Pointer arithmetic and subscripting
- Given a pointer `ptr` to an element of some type `T`, `ptr + n` computes a NEW address `n * sizeof(T)` bytes further in memory (i.e., the address of the element `n` positions later) — the compiler automatically scales the arithmetic by the pointed-to type's size, you don't manually multiply by `sizeof(T)` yourself.
- `ptr[n]` is DEFINED to be exactly equivalent to `*(ptr + n)` — subscripting is literally pointer arithmetic plus a dereference, under the hood; this is precisely why array indexing and pointer arithmetic are so closely related and interchangeable for C-style arrays.
- Since a C-style array decays to a pointer to its FIRST element, `arr[i]` and `*(arr + i)` are genuinely, syntactically interchangeable for C-style arrays (a very classic "explain why these are equivalent" interview question) — and even `i[arr]` technically compiles and works identically, since addition is commutative (`*(arr+i)` == `*(i+arr)`) — a fun, rarely-practical trivia point sometimes raised specifically to test genuine understanding of the underlying mechanism versus rote syntax memorization.
- **Guidance on when to use which**: prefer subscripting (`arr[i]`) when indexing relative to the START of the array (index 0) for clarity; pointer arithmetic (`*(ptr + n)`) reads more naturally when doing RELATIVE positioning from some other already-known pointer/element (not necessarily the array's start).

## 17.10–17.11 C-style strings and string symbolic constants
- A **C-style string** is simply a C-style array of `char`, terminated by a special **null terminator** character (`'\0'`, value 0) marking the logical end of the string — functions operating on C-style strings (`strlen`, `strcpy`, etc., from `<cstring>`) rely on scanning for this terminator rather than tracking length separately, which is the root cause of classic C string bugs (buffer overruns if the terminator is missing/overwritten, off-by-one errors leaving no room for it).
- String literals (`"hello"`) are themselves C-style (null-terminated) character arrays, stored with **static duration** — and, notably, are actually **lvalues** in C++ (an exception to the general rule that literals are rvalues, Ch 12.2) due to how they're stored.
- Modern C++ code should overwhelmingly prefer `std::string`/`std::string_view` over raw C-style strings — this material is largely important for (a) interoperating with C APIs/legacy code, and (b) understanding what's happening "under the hood" when discussing `std::string`'s relationship to `.c_str()`.

## 17.12–17.13 Multidimensional arrays (C-style and `std::array`)
- A C-style multidimensional array (`int arr[3][4];`) is stored in **row-major order** — elements of a given ROW are contiguous in memory; moving to the next row jumps forward by an entire row's worth of elements. (Contrast: some other languages, like Fortran, default to column-major order — worth knowing the term if asked to compare.)
- `std::array` doesn't natively support a clean multidimensional syntax the way C-style arrays do — the idiomatic modern approach is typically an `std::array` of `std::array`s (`std::array<std::array<int, 4>, 3>`), which is more verbose but retains all the usual `std::array` benefits (bounds-checkable via `.at()`, knows its own size, etc.) — a good pattern to recognize/be able to produce if asked to model a matrix/grid in an OA.

---

# CHAPTER 18 — Iterators and Algorithms

## 18.1 Sorting an array using selection sort
- **Selection sort** algorithm walk-through: repeatedly scan the UNSORTED remainder of the array to find its minimum (or maximum) element, then SWAP it into its correct final position at the front of the unsorted remainder — repeat for each position. `O(n²)` comparisons regardless of input order (always scans the full unsorted remainder each pass), `O(n)` swaps (at most one swap per pass) — the low swap count is selection sort's main practical advantage over, say, bubble sort, when swaps are comparatively expensive.
- `std::swap(a, b)` (header `<utility>` / `<algorithm>`) is the standard, idiomatic way to exchange two values — internally uses move semantics (Ch 22) where available, more efficient and less error-prone than manually writing a temp-variable swap by hand.
- This lesson is also a natural setup for discussing **algorithmic complexity/Big-O** in an interview context — selection sort is a classic "explain this algorithm and its complexity" OA warm-up question.

## 18.2 Introduction to iterators
- An **iterator** is an object that abstracts "a position within a container" and knows how to move to the NEXT (and sometimes previous) position, and how to access the element AT its current position — the general-purpose mechanism that lets generic algorithms and range-based for loops work uniformly across very different container implementations (contiguous arrays, linked lists, trees, etc.) without needing to know each container's internal structure.
- Core iterator operations (conceptually, mirroring pointer syntax deliberately): `*it` (dereference — access the current element), `++it` (advance to the next element), `it == otherIt` / `it != otherIt` (compare position, most commonly used to detect "have we reached the end").
- Every standard container exposes `.begin()` (an iterator to the FIRST element) and `.end()` (a "past-the-end" SENTINEL iterator — deliberately NOT a valid dereferenceable position, just a marker meaning "one past the last real element," used purely for the loop-termination comparison).
- **Range-based for loops (Ch 16.8) are literally syntactic sugar over this exact iterator pattern** — `for (auto& x : container) { ... }` compiles down to something conceptually equivalent to `for (auto it = container.begin(); it != container.end(); ++it) { auto& x = *it; ... }` — a very commonly asked "what does a range-based for loop actually do under the hood" interview question.
- A raw pointer into a C-style array or `std::array`'s underlying buffer technically already satisfies the basic iterator interface (supports `*`, `++`, `==`) — this is why some of the simplest STL algorithms can be, and historically were, demonstrated directly with raw pointers before iterators were formally introduced as their own abstraction.

## 18.3 Introduction to standard library algorithms (`<algorithm>`)
- The philosophy: rather than hand-writing loops for common operations (searching, sorting, counting, transforming) every single time, the standard library provides a large set of GENERIC, well-tested, iterator-based **algorithm** functions in `<algorithm>` that work uniformly across any compatible container.
- Frequently-used examples worth being comfortable naming/using in an OA setting:
  - `std::find(begin, end, value)` — returns an iterator to the first matching element, or `end` if not found.
  - `std::sort(begin, end)` (optionally with a custom comparator) — sorts the range in place, typically an efficient introsort/hybrid algorithm (better than the manually-written `O(n²)` selection sort from 18.1) — `O(n log n)` average/typical case.
  - `std::count(begin, end, value)` — counts matching elements.
  - `std::max_element` / `std::min_element` — returns an iterator to the largest/smallest element in a range.
  - `std::for_each(begin, end, function)` — applies a function/lambda to every element.
- **Best-practice guidance emphasized repeatedly across the course**: prefer reaching for an existing standard algorithm over hand-writing an equivalent loop whenever a suitable one exists — reduces bugs (these are extensively tested), often more efficient (implementations can leverage internal knowledge/optimizations), and communicates INTENT more clearly to a reader (`std::sort(...)` immediately says "this sorts," whereas an equivalent hand-rolled loop requires reading through the logic to infer the same intent) — a genuinely good talking point if asked "how do you write idiomatic modern C++."
- Many algorithms accept an optional comparator/predicate (often supplied as a lambda, Ch 20.6/20.7) to customize behavior, e.g. `std::sort(v.begin(), v.end(), [](int a, int b){ return a > b; });` for descending order.

## 18.4 Timing your code
- Standard, portable way to measure elapsed wall-clock time for benchmarking: `<chrono>`'s `std::chrono::steady_clock` (preferred for measuring durations/benchmarking specifically because, unlike `system_clock`, it's guaranteed monotonic — never jumps backward due to system clock adjustments) — capture a `time_point` before and after the code being measured, and compute the `duration` between them.
```cpp
#include <chrono>
auto start{ std::chrono::steady_clock::now() };
// ... code being timed ...
auto end{ std::chrono::steady_clock::now() };
std::chrono::duration<double> elapsed{ end - start };
```
- Relevant context/talking point: comparing a hand-written `O(n²)` sort against `std::sort`'s typically-`O(n log n)` performance empirically is exactly the kind of exercise this lesson sets up, tying back nicely to 18.1/18.3.

---

# CHAPTER 19 — Dynamic Allocation

## 19.1 Dynamic memory allocation with `new` and `delete`
- Three kinds of memory a C++ program uses: **static/global** (program lifetime), **automatic/stack** (local variables, automatically allocated/deallocated as scopes are entered/exited, very fast, but limited in size and lifetime tied strictly to scope), and **dynamic/heap** (explicitly requested at runtime via `new`, lives until explicitly `delete`d — flexible size and lifetime, but SLOWER to allocate and entirely the PROGRAMMER's responsibility to free).
```cpp
int* ptr{ new int };       // allocate a single int on the heap, uninitialized
int* ptr2{ new int{ 5 } }; // allocate and initialize to 5
delete ptr;                // free it — MUST be paired with every new
ptr = nullptr;             // good practice: null out the pointer after deleting, to avoid an accidental dangling-pointer use
```
- **Memory leak**: heap memory that's been allocated but never `delete`d, AND for which the program has lost every pointer that could be used to free it later (e.g. reassigning the only pointer to it, or it going out of scope) — the memory remains allocated for the rest of the program's run, unusable and unrecoverable, until the OS reclaims it at process exit.
- `delete` on a `nullptr` is explicitly well-defined and SAFE (a no-op) — no need to null-check before deleting.
- **Dangling pointer**: a pointer still holding the address of memory that has ALREADY been freed (via `delete`) — using (dereferencing) a dangling pointer is undefined behavior; this is why nulling a pointer out immediately after `delete` is recommended (a subsequent accidental dereference of a NULL pointer at least crashes predictably/detectably, rather than corrupting memory silently).
- **Double-free**: calling `delete` twice on the same already-freed pointer — undefined behavior, a very common and serious bug, and the exact issue that shallow-copy classes (Ch 21.13) and un-nulled moved-from smart pointer internals (Ch 22.3) are specifically designed to avoid.
- Modern C++ guidance (reinforced heavily by Ch 22's smart pointers): prefer `std::unique_ptr`/`std::shared_ptr` over manual `new`/`delete` wherever possible — this chapter's manual approach is foundational/historical understanding, but production code should lean on RAII-based smart pointers instead.

## 19.2 Dynamically allocating arrays
```cpp
int* arr{ new int[10] };   // allocate an array of 10 ints on the heap
delete[] arr;              // MUST use delete[] (not plain delete) to free an array allocation
```
- **Critical rule (frequently tested)**: `new[]` must ALWAYS be paired with `delete[]`, and plain `new` must ALWAYS be paired with plain `delete` — mixing them (`delete` on a `new[]`-allocated array, or `delete[]` on a plain `new`) is undefined behavior. This mismatch is a classic subtle bug and interview "what's wrong with this code" question.
- Unlike C-style STACK arrays, the array's LENGTH used for a dynamic `new[]` allocation does NOT need to be a compile-time constant — it can be any runtime-computed value, which is precisely the main practical reason to reach for dynamic array allocation over a C-style stack array in the first place (variable-length arrays are not standard C++ on the stack).
- `delete[]` correctly calls the destructor for EVERY element in the array before freeing the underlying memory block (relevant for arrays of class-type objects, not just primitives).

## 19.3 Destructors (recap in the dynamic-allocation context)
- Reiterates and applies Ch 15.4's destructor material specifically to classes that manage dynamically-allocated resources — a class holding a `new`'d pointer member should free it in its own destructor, tying resource lifetime to object lifetime (RAII), so users of the class don't need to manually manage that inner pointer themselves. Sets up the motivation directly leading into Ch 21's Rule of Three and Ch 22's smart pointers.

## 19.4 Pointers to pointers and dynamic multidimensional arrays
- A **pointer to a pointer** (`int** ptr;`) holds the address of ANOTHER pointer variable — two levels of indirection. Used, among other things, to implement dynamically-allocated 2D arrays where each "row" is itself a separately, dynamically allocated 1D array:
```cpp
int** arr2d{ new int*[rows] };       // array of row-pointers
for (int i = 0; i < rows; ++i)
    arr2d[i] = new int[cols];        // each row separately allocated
// cleanup, in reverse order:
for (int i = 0; i < rows; ++i)
    delete[] arr2d[i];
delete[] arr2d;
```
- This "array of separately-allocated rows" approach means the rows are NOT guaranteed contiguous in memory with each other (unlike a true fixed 2D C-style array, which IS one single contiguous block) — a subtle but real difference worth knowing (e.g. relevant to cache-locality discussions, or to why you can't just treat this structure as one flat pointer the way you could a real contiguous 2D array).
- Modern C++ strongly prefers `std::vector<std::vector<T>>` (or a single flat `std::vector<T>` with manual row/col index math for genuine contiguity) over this manual double-pointer approach, for RAII safety and to avoid the multi-step allocation/deallocation bookkeeping shown above.

## 19.5 Void pointers
- `void*` — a pointer that can hold the address of ANY data type, but carries NO type information about what it actually points to — consequently, it **cannot be dereferenced directly** (the compiler has no idea how many bytes to read or how to interpret them) — it must first be explicitly cast to a concrete pointer type (`static_cast<int*>(voidPtr)`) before dereferencing.
- Historically used in C-style generic programming (e.g. `memcpy`'s signature, C-style callback APIs) before C++ templates provided a type-safe alternative — in modern C++, templates (and `std::any`/`std::variant` for genuinely type-erased storage) are almost always preferred over `void*` for new code; recognizing `void*` is mostly important for reading/interoperating with legacy C APIs.

---

# CHAPTER 20 — Functions

## 20.1 Function pointers
- Just as a normal pointer holds the address of a variable, a **function pointer** holds the address of a FUNCTION, and can be used to CALL that function indirectly through the pointer:
```cpp
int add(int x, int y) { return x + y; }
int (*fcnPtr)(int, int) { &add }; // note the parentheses around *fcnPtr — required for correct precedence
int result{ fcnPtr(3, 4) };       // or (*fcnPtr)(3, 4) — both call syntaxes are valid and equivalent
```
- The function pointer's declared TYPE must match the target function's signature exactly (parameter types AND return type) — this is effectively function-pointer "overload resolution" happening at the point of address-taking.
- **Primary use case**: passing a function AS A PARAMETER to another function, enabling **callback**-style patterns — e.g. passing a custom comparator function into a sorting routine (directly relevant to `std::sort`'s comparator parameter, Ch 18.3) — a foundational concept that DIRECTLY motivates why lambdas (20.6/20.7) and `std::function` were later added as more convenient, flexible alternatives.
- `using` type aliases (Ch 10.7) are commonly used to make function pointer types more readable, since raw function pointer syntax is notoriously awkward: `using ValidateFcn = bool(*)(int);`

## 20.2 The stack and the heap
- **The call stack**: a region of memory that automatically manages **function call frames** — each function call PUSHES a new "stack frame" (holding that call's parameters, local variables, and return address) onto the stack; returning from the function POPS that frame off. This automatic push/pop behavior is precisely what makes local variable lifetime (automatic duration, Ch 7) work — and precisely why local variables become invalid the instant their function returns (their storage was just popped off the stack — see the classic dangling-reference bug from Ch 12.14/16.5).
- **Stack overflow**: occurs when the call stack grows beyond its (typically fairly small, often just a few MB) allotted size — most commonly caused by excessively deep or INFINITE recursion (each recursive call pushes another frame without ever popping) — results in a crash. A frequently-tested "what could cause this" / "how would you fix this recursive function" interview scenario.
- **The heap**: the region used for dynamically-allocated memory (`new`/`delete`, Ch 19) — much larger available capacity than the stack, but SLOWER to allocate/deallocate from (involves more complex bookkeeping to track free/used blocks), and NOT automatically managed — the programmer (or a smart pointer, Ch 22) is responsible for freeing it explicitly.
- **Stack vs heap — the classic side-by-side comparison (very frequently asked as a direct interview question)**:

| | Stack | Heap |
|---|---|---|
| Speed | Very fast (simple pointer bump) | Slower (more complex allocation bookkeeping) |
| Size | Small, fixed at program start | Large, limited mainly by system memory |
| Lifetime | Automatic — tied to scope | Manual — until explicitly freed (or smart-pointer-managed) |
| Management | Automatic (compiler-managed) | Manual (`new`/`delete`, or RAII/smart pointers) |
| Typical failure mode | Stack overflow (usually from deep/infinite recursion) | Memory leaks, dangling pointers, fragmentation |

## 20.3 Recursion
- A **recursive function** calls itself (directly, or indirectly through another function) to solve a problem by breaking it into smaller sub-instances of the SAME problem.
- **Every correct recursive function needs**:
  1. A **base case** — a condition under which the function returns a result directly, WITHOUT recursing further (this is what stops the recursion).
  2. A **recursive case** — where the function calls itself with a "smaller"/simpler argument that provably moves toward the base case.
- Missing or unreachable base case → infinite recursion → stack overflow (directly ties back to 20.2).
- **Recursion vs iteration tradeoff (a classic interview discussion point)**: recursive solutions can be more elegant/readable for naturally recursive problems (tree traversal, some divide-and-conquer algorithms), but carry real overhead — EACH recursive call pushes a new stack frame (function-call overhead: parameter passing, return address bookkeeping) — an equivalent iterative (loop-based) solution is typically faster and uses constant stack space. Some recursive patterns (specifically **tail recursion**, where the recursive call is the very LAST operation performed, with no further work needed after it returns) CAN be optimized by some compilers into an equivalent loop internally ("tail call optimization") — but C++ does NOT guarantee this optimization will happen, so relying on it for correctness (e.g. to avoid stack overflow on deep recursion) is risky/non-portable.
- Classic recursive examples worth being fluent in for an OA: factorial, Fibonacci (and the strong contrast between naive exponential recursive Fibonacci vs. an efficient iterative/memoized version — a very common complexity-analysis interview question), binary search, tree/graph traversal.

## 20.4 Command line arguments
- `int main(int argc, char* argv[])` — the standard way a C++ program receives arguments passed to it from the command line when invoked.
- `argc` ("argument count"): the number of command-line arguments, ALWAYS at least 1 (by convention, `argv[0]` is the program's own name/invocation path).
- `argv` ("argument vector"): a C-style array of C-style strings (`char*`), one per argument — `argv[0]` is the program name, `argv[1]` through `argv[argc-1]` are the actual user-supplied arguments, and `argv[argc]` is guaranteed to be a null pointer (a sentinel, mirroring the null-terminator convention for individual C-strings).
- Since each `argv[i]` is a raw C-style string, converting a numeric argument (e.g. `"42"`) into an actual `int`/`double` requires manual parsing (`std::stoi`, `std::stod`, or a `std::stringstream`, Ch 13.5-adjacent material) — command-line arguments arrive as TEXT regardless of their intended semantic type.

## 20.5 Ellipsis (and why to avoid them)
- `void foo(int count, ...);` — the ellipsis (`...`) parameter accepts a VARIABLE, UNSPECIFIED number/type of additional trailing arguments — inherited from C (this is exactly how legacy functions like `printf` accept a variable argument list).
- **Why they're generally discouraged in modern C++ (a good "explain a C++ pitfall" interview topic)**: ellipsis parameters provide **NO type safety whatsoever** — the compiler does not check that the arguments passed match what the function actually expects to receive; the function body must manually use macros (`va_start`, `va_arg`, `va_end` from `<cstdarg>`) to extract arguments, entirely trusting that the CALLER passed the right number and right types in the right order — mismatches are undefined behavior, not a compile error. This is precisely the kind of bug class that got printf-style format-string vulnerabilities into real-world software.
- Modern C++ alternatives that provide the same "variable number of arguments" flexibility WITH type safety: function templates with **variadic template parameters** (`template <typename... Args> void foo(Args... args)`, C++11+) or simply passing a `std::vector`/`std::initializer_list` (Ch 23.7) when all the variable arguments share a common type.

## 20.6 Introduction to lambdas (anonymous functions)
- A **lambda expression** defines an anonymous, inline, callable function object directly at the point of use — extremely common as a way to pass small custom logic (e.g. comparators, predicates) into standard algorithms (Ch 18.3) without needing to define a separate named function elsewhere:
```cpp
auto isEven = [](int x) { return x % 2 == 0; };
std::sort(v.begin(), v.end(), [](int a, int b) { return a > b; }); // descending-order comparator, inline
```
- Anatomy: `[capture-clause](parameters) -> returnType { body }` — the return type is usually omitted and auto-deduced from the body (like `auto` function return-type deduction, Ch 10.9); explicit return type is only needed occasionally (e.g. multiple return statements with mismatched-but-convertible types).
- **A lambda's actual TYPE is a unique, compiler-generated, unnamed CLASS TYPE** — this is precisely why lambdas are typically stored using `auto` (you can't spell out their real type) — and this is DIRECTLY connected to Ch 21.10's `operator()` functor material: a lambda is, under the hood, essentially syntactic sugar for the compiler auto-generating a small functor class whose `operator()` implements the lambda's body, and whose member variables (if any) are populated according to the lambda's CAPTURE clause. Recognizing this "a lambda IS a functor" equivalence is a very commonly-tested conceptual link between these two chapters.
- `std::function<ReturnType(ParamTypes...)>` (header `<functional>`) is the standard, TYPE-ERASED way to store/pass around ANY callable (lambda, function pointer, or functor) matching a given signature, when you need a uniform type to hold different kinds of callables (e.g. a class member variable holding a customizable callback) — comes with some runtime overhead (type erasure, possible heap allocation for captures) compared to a lambda passed directly as a template parameter (which the compiler can inline/optimize much more aggressively).

## 20.7 Lambda captures
- The **capture clause** (`[...]`) specifies which surrounding (enclosing-scope) variables the lambda's body is allowed to access, and HOW:
  - `[]` — captures nothing; the lambda can only use its own parameters and globals.
  - `[x]` — captures `x` **BY VALUE** (a COPY of `x`'s value at the point the lambda is CREATED, not when it's later called — a subtlety worth flagging).
  - `[&x]` — captures `x` **BY REFERENCE** (the lambda sees LIVE updates to the real `x`; but this also means the lambda must not outlive `x`, or you get a dangling reference — exactly analogous to Ch 12.14's dangling-reference concerns).
  - `[=]` — captures ALL used surrounding variables by value (implicit, blanket capture).
  - `[&]` — captures ALL used surrounding variables by reference (implicit, blanket capture).
  - `[=, &y]` / `[&, x]` — mixed: a default capture mode plus explicit overrides for specific named variables.
- **A very commonly tested gotcha**: capturing by value captures the variable's value AT THE TIME THE LAMBDA IS DEFINED — subsequent changes to the original variable AFTER the lambda is created are NOT reflected inside the lambda (since it holds its own independent copy) — contrast directly with capture-by-reference, which always sees the CURRENT (possibly since-modified) value.
- By default, a lambda's captured-by-value variables are effectively `const` inside the lambda body (the generated functor's `operator()` is `const` by default) — attempting to modify a by-value-captured variable inside the lambda is a compile error UNLESS the lambda is explicitly marked **`mutable`**: `[x]() mutable { x++; };` — even then, note that `mutable` only allows modifying the lambda's OWN internal copy; it still has zero effect on the original outside variable.
- **Dangling-reference risk with capture-by-reference is a real, frequently-cited danger**, especially for lambdas that are STORED and called later (e.g. as a callback, or returned from a function) rather than used and discarded immediately — if the referenced variable goes out of scope before the lambda is actually invoked, calling it is undefined behavior, exactly mirroring the general dangling-reference theme running throughout the whole "chapters 12–25" range covered so far. **Prefer capture-by-value for lambdas that might outlive the scope they were created in**, and reserve capture-by-reference for lambdas used strictly within the same, still-live scope (e.g. passed immediately into a `std::sort` call and not stored anywhere).

---

## Quick Cross-Chapter Traps to Memorize (Ch 16–20)

| Topic | The trap |
|---|---|
| `std::vector<int> v{5}` vs `v(5)` | Brace = 1 element valued 5 (initializer_list ctor); parens = 5 default elements |
| Unsigned `size_type` | Backward loops with an unsigned index and `index >= 0` never terminate (wraps to huge positive instead of going negative) |
| Reallocation | `push_back`/`resize`/`reserve`(growing) invalidate ALL existing iterators/pointers/references into the vector |
| `std::vector<bool>` | Doesn't store real `bool`s — bit-packed, `operator[]` returns a proxy, not a real `bool&` |
| C-style array decay | A C-style array parameter is secretly just a pointer — `sizeof` on it gives pointer size, not array size |
| Pointer arithmetic | `arr[i]` is literally defined as `*(arr + i)` — that's why array subscripting and pointer offsets are interchangeable |
| `new`/`delete` pairing | `new[]` must pair with `delete[]`; mixing with plain `new`/`delete` is undefined behavior |
| Dangling/leaked pointers | Forgetting to null a pointer after `delete`, or losing the only pointer to allocated memory before freeing it |
| Stack overflow | Usually caused by deep/infinite recursion filling up the (small, fixed) call stack |
| Ellipsis (`...`) | Zero type safety — caller/callee argument mismatch is undefined behavior, not a compile error |
| Lambda = functor | A lambda is sugar for a compiler-generated class with `operator()`; capture clause populates its members |
| Lambda capture-by-value | Captures the value AT CREATION TIME — later changes to the original variable aren't seen inside the lambda |
| Lambda capture-by-reference | Same dangling-reference risk as any reference — dangerous for lambdas stored/called after their scope ends |
