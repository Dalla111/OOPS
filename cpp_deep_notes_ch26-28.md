# C++ Deep-Dive Notes — Chapters 26–28 (LearnCpp.com)
### Exhaustive, interview/OA-oriented — every lesson, every gotcha

---
# CHAPTER 26 — Templates and Classes (Advanced)

## 26.1 Template classes
- Recaps and extends Ch 13.13/15.5's introduction to class templates, focusing specifically on a fully worked-out generic CONTAINER example (a templated dynamic `Array<T>` class), and formalizing the ORGANIZATIONAL rule already flagged in Ch 11.10.
- **Critical, heavily-emphasized rule (directly extends Ch 11.10 to class templates)**: with NORMAL (non-template) classes, the standard practice is class definition in a `.h` header, member function DEFINITIONS in a matching `.cpp` file. **This does NOT work for template classes** — splitting a template class's member function definitions into a separate `.cpp` file causes a LINKER error ("undefined reference"), because the compiler only instantiates a template member function in a translation unit where it's ACTUALLY USED — a separate `.cpp` file containing only the definitions (with no actual usage triggering instantiation) never causes the compiler to generate any real code for it, so there's nothing for the linker to find when `main.cpp` (which DOES use it) needs it.
- **Fix**: template class member functions, even when "split out" for organizational clarity, must still ultimately be VISIBLE (via `#include`) in the header, or otherwise fully defined in a way included wherever the template is used — typically, member functions are defined either directly inside the class body, or just below it in the SAME header file (each definition outside the class body needs ITS OWN `template <typename T>` declaration repeated: `template <typename T> T& Array<T>::operator[](int index) { ... }`).
- **Naming detail worth recognizing**: inside the class's own member function definitions, the class template's name used WITHOUT explicit template arguments (`Array` rather than `Array<T>`) implicitly refers to "the current instantiation" — e.g. a copy constructor can be written taking `const Array&` rather than needing to spell out `const Array<T>&`, since within that context they mean the same thing.

## 26.2 Template non-type parameters (class-template context) / class templates with static members
- Extends Ch 11.9/17.1's non-type template parameter material specifically into the CLASS-template context — e.g. `template <typename T, int size> class StaticArray { T m_array[size]{}; ... };`, directly foreshadowing 26.5's StaticArray example used for partial specialization.
- **Static members of a class template are worth flagging as a distinct gotcha**: a static member declared inside a class TEMPLATE is NOT shared across ALL instantiations of the template — rather, EACH distinct instantiation (e.g. `Foo<int>` vs `Foo<double>`) gets its OWN, completely SEPARATE copy of that static member, since each instantiation is genuinely a distinct, independently-compiled class. This is a frequently-tested "does `Foo<int>::count` and `Foo<double>::count` refer to the same variable" conceptual question — they do NOT; each specialization of the template has independent static state.

## 26.3 Function template specialization
- **Full (explicit) specialization**: overriding a function template's GENERIC behavior with a hand-written, completely custom implementation for ONE SPECIFIC type:
```cpp
template <typename T>
void print(T value) { cout << value; }

template <>  // explicit specialization marker — empty angle brackets
void print<double>(double value) { cout << std::scientific << value; }  // custom double behavior
```
- Once this specialization exists, calling `print(5.5)` (deducing `T = double`) uses the SPECIALIZED version instead of instantiating the generic template for `double`.
- **Important, frequently-tested detail**: explicit (full) specializations are **NOT implicitly `inline`** the way template definitions normally are (recall Ch 7.9/11.10's ODR-exemption discussion for templates) — if a full specialization's definition is placed in a HEADER included by multiple `.cpp` files, it must be EXPLICITLY marked `inline` to avoid an ODR/multiple-definition linker error, unlike a normal (un-specialized) template function, which gets this exemption automatically.
- This is DISTINCT from simply providing a plain (non-template) OVERLOAD with the exact matching signature — both approaches can look superficially similar for a single-argument case, but explicit specialization vs. overloading have different overload-resolution priority rules and different applicability (specialization only works for an EXISTING template; overloading works independently).

## 26.4 Class template specialization
- The SAME specialization concept, applied to an ENTIRE CLASS (or a single MEMBER FUNCTION of a class template) rather than a standalone function template:
```cpp
template <typename T>
class Storage {
    T m_value{};
public:
    Storage(T value) : m_value{value} {}
    void print() { cout << m_value; }
};

template<>  // specializing just ONE member function for T=double
void Storage<double>::print() { cout << std::scientific << m_value; }
```
- You can specialize just a SINGLE member function (as above, leaving the rest of `Storage<double>` using the generic template's other members), or specialize the ENTIRE class body for a specific type (providing a completely independent, from-scratch implementation of every member for that one type) — useful when a type needs a genuinely different INTERNAL REPRESENTATION, not just different behavior in one function (the canonical real-world example: `std::vector<bool>`'s specialized bit-packed internal storage, already flagged in the Ch 16.12 notes, is precisely a class template full specialization).
- **Design principle worth citing**: when fully specializing a class, best practice is to KEEP THE PUBLIC INTERFACE IDENTICAL to the generic template's interface — callers should be able to use either version (generic or specialized) completely interchangeably without caring which one they actually got, even though the internal implementation may be completely different.

## 26.5 Partial template specialization
- **Partial specialization**: specializing a class template for a SUBSET/PATTERN of possible template arguments (e.g., "any pointer type", "any array of a specific element type but with size left generic"), rather than one single fully-concrete type as in 26.4's FULL specialization.
```cpp
template <typename T, int size>
class StaticArray { T m_array[size]{}; /* ... */ };

template <int size>  // partially specialize: T is fixed to char, size stays templated
class StaticArray<char, size> { /* custom char-specific implementation */ };
```
- **THE critical, extremely-frequently-tested rule (already flagged in earlier notes, formally the subject of this entire lesson)**: **partial template specialization is ONLY allowed for CLASS templates — it is explicitly, permanently DISALLOWED for FUNCTION templates** in standard C++ (`template <typename T> void print(StaticArray<T, size>&)` cannot be partially specialized the same way).
- **The standard workaround for the "partial specialization" need on FUNCTIONS**: since a function CANNOT be partially specialized, but CAN be freely OVERLOADED (Ch 11.1), you achieve the same practical effect by writing an additional, more-specific OVERLOADED function template instead of attempting a (disallowed) partial specialization:
```cpp
template <typename T, int size>
void print(const StaticArray<T, size>& array) { /* generic version */ }

template <int size>  // this is a NEW OVERLOAD (legal), NOT a partial specialization (which would be illegal here)
void print(const StaticArray<char, size>& array) { /* char-specific version, e.g. print as a string */ }
```
  This distinction — "you CAN'T partially specialize a function template, but you CAN achieve the same practical outcome via overloading" — is possibly the single most commonly quizzed fact from this entire chapter.
- **Partial specialization of MEMBER functions is notably awkward**: you generally CANNOT partially specialize just ONE member function of a class template while leaving the rest generic (unlike FULL specialization, 26.4, where specializing a single member function while leaving the rest generic IS allowed) — attempting this typically requires either fully partially-specializing the ENTIRE class (duplicating all the other members' code, which is not reusable) or restructuring the design (e.g. via inheritance/composition tricks) to avoid the need altogether. This awkwardness is explicitly acknowledged as a genuine limitation of the language in this area.

## 26.6 Partial template specialization for pointers
- A specific, very commonly-cited PRACTICAL application of 26.5's partial specialization: providing a specialized version of a template for when the template type parameter `T` is itself A POINTER TYPE (`T*`), as opposed to any single concrete pointer type:
```cpp
template <typename T>
class Storage { T m_value; /* generic: works for value types */ };

template <typename T>            // partial specialization: matches ANY pointer type T*
class Storage<T*> { T* m_value;  /* pointer-specific behavior, e.g. might dereference before printing, or manage pointed-to lifetime differently */ };
```
- Motivating use case explicitly highlighted: a generic container storing a POINTER shouldn't necessarily behave identically to one storing a plain VALUE (e.g., printing a `Storage<int*>` naively via the generic template would print a raw memory address, not the pointed-to int's actual value — a specialized `Storage<T*>` can be written to dereference first, giving more useful/expected behavior specifically for the pointer case) — illustrating WHY partial (rather than full) specialization is valuable: it captures an entire CATEGORY of types ("any pointer") with one specialization, rather than needing a separate full specialization for every individual concrete pointer type (`int*`, `double*`, `MyClass*`, ...).

**Likely interview Qs for Ch 26:** Why does splitting a template class's member functions into a separate .cpp file cause a linker error? Is a static member of a class template shared across different instantiations (e.g. `Foo<int>` and `Foo<double>`)? Why can't you partially specialize a function template, and what do you do instead? What's the difference between full and partial specialization?

---
# CHAPTER 27 — Exceptions

## 27.1 The need for exceptions
- Recap/motivation: return-code-based error handling (Ch 9.4) tightly INTERMINGLES a function's normal control flow with its error-handling logic — every caller must remember to check the return code, error codes can be silently ignored, and propagating an error up through MULTIPLE levels of nested function calls requires every intermediate function to explicitly check and re-propagate the failure, cluttering code at every level.
- **Exceptions decouple error DETECTION from error HANDLING**: the code that DETECTS a problem can simply `throw`, without needing to know (or care) how — or even WHETHER — the error will eventually be handled; the code that actually HANDLES the error (a `catch` block) can be located FAR UP the call stack, with all the intervening layers of function calls needing NO explicit error-checking code at all.

## 27.2 Basic exception handling — `throw`, `try`, `catch`
```cpp
try {
    throw 4.5;                  // 'throw' raises an exception (here, of type double)
    cout << "never prints";     // unreachable — control leaves immediately upon throw
} catch (double x) {            // 'catch' handles exceptions of a matching type
    cerr << "Caught: " << x;
}
```
- **Mechanics, precisely**: a `throw` statement immediately raises an exception of whatever type the thrown expression has; control jumps AT ONCE to the nearest ENCLOSING `try` block (skipping over any remaining code in the current scope, exactly like the "never prints" line above); the compiler then checks each `catch` handler attached to that try block, IN ORDER, for one whose type matches the thrown exception; the FIRST matching handler runs.
- ANY type can be thrown — not just classes; can be an `int`, a `double`, a `std::string`, or (much more commonly in good practice) a class type, especially one derived from `std::exception` (27.5).
- A `try` block can have MULTIPLE `catch` handlers attached, each for a DIFFERENT exception type — the compiler checks them TOP TO BOTTOM and uses the FIRST one that matches (ORDER MATTERS — a broader/base-class catch listed BEFORE a more specific derived-class catch will "steal" the match, a frequently-tested gotcha covered further in 27.5).

## 27.3 Exceptions, functions, and stack unwinding
- **A crucial capability exceptions provide that return codes cannot**: an exception thrown deep inside a NESTED function call (several levels down) can be caught by a `try` block ANYWHERE up the call stack — the exception PROPAGATES UPWARD automatically through every intervening function call, without those intermediate functions needing any explicit error-checking/propagation code at all.
- **Stack unwinding**: as an exception propagates upward searching for a matching handler, every function call frame between the `throw` and the eventual matching `catch` is "unwound" — meaning each of those frames' LOCAL OBJECTS are properly destructed (in the normal reverse-of-construction order, Ch 24.3) as the stack unwinds through them, exactly as if each function had returned normally. **This is precisely why RAII (Ch 15.4/22.5-22.6) is considered essential/idiomatic for writing EXCEPTION-SAFE C++ code**: local RAII objects (smart pointers, `std::string`, `std::vector`, file handles, etc.) will have their destructors correctly invoked during unwinding, safely releasing their resources, even though the function never reaches its normal end/return statement.
- **If NO matching handler is found ANYWHERE up the entire call stack** (including all the way up through `main()`), the program calls `std::terminate()` (closely related to `std::abort()`, Ch 8.12) — the program crashes with an "unhandled exception" error, and — critically — **stack unwinding does NOT necessarily complete properly in this specific failure case** (local objects' destructors are NOT guaranteed to run in this scenario, since std::terminate can be invoked directly without necessarily performing a full, guaranteed unwind first) — this is a genuinely nuanced point (implementation-defined details vary) but the safe TAKEAWAY is: an uncaught exception is a serious, RAII-safety-undermining failure mode, not just an inconvenience — always ensure your program has appropriate top-level catch handlers.

## 27.4 Uncaught exceptions and catch-all handlers
- **`catch (...)`**: a "catch-all" handler using ellipsis syntax — matches ANY exception type whatsoever, regardless of what was thrown. Useful as a LAST-RESORT safety net (e.g. at/near the top of `main()`) to prevent std::terminate from being invoked due to a genuinely unanticipated exception type, though it provides NO information about what was actually caught (you can't inspect the exception's value/type through `...`) — typically paired with at minimum a log message and a graceful shutdown, rather than attempting to meaningfully "handle" the unknown error.
- Common pattern:
```cpp
try {
    riskyOperation();
} catch (const std::exception& e) {
    cerr << "Known error: " << e.what();
} catch (...) {
    cerr << "Unknown exception caught!";
}
```

## 27.5 Exceptions, classes, and inheritance
- **`std::exception`** (header `<exception>`) is the standard library's small INTERFACE-style base class — essentially all standard-library-thrown exceptions (as of C++20, roughly 28 distinct exception classes, per the standard) derive from it, giving a UNIFORM way to catch "something went wrong in the standard library" without needing to know the exact specific type: `catch (const std::exception& e) { cerr << e.what(); }` catches essentially ANY standard exception, and `.what()` (a virtual member function) returns a human-readable description string.
- Common concrete standard exception types worth recognizing: `std::runtime_error`/`std::logic_error` (and their further subclasses like `std::out_of_range`, `std::invalid_argument`) from `<stdexcept>`, `std::bad_alloc` (thrown by `operator new` on allocation failure), `std::bad_cast` (thrown by a failed REFERENCE-form `dynamic_cast`, Ch 25.10).
- **A CATCH HANDLER TAKING A BASE CLASS REFERENCE will also catch exceptions of ANY DERIVED class** — this is exactly the same polymorphic-reference behavior from Ch 25.2 applied to exception handling: `catch (const std::exception& e)` catches a thrown `std::runtime_error`, `std::out_of_range`, or any other class deriving (even indirectly) from `std::exception`, since a reference to a base class can bind to a derived object.
- **CRITICAL, frequently-tested ordering rule**: when MULTIPLE catch handlers are attached to one try block for a HIERARCHY of exception types, **more DERIVED (specific) exception types MUST be listed BEFORE more BASE (general) ones** — because the compiler checks handlers top-to-bottom and stops at the FIRST match, a base-class catch listed FIRST would "steal" ALL derived-type exceptions too (since a derived object IS-A base object), NEVER letting the more specific handler(s) below it ever run — a classic "why doesn't my specific catch block ever execute" bug and interview question:
```cpp
try { throw std::out_of_range("oops"); }
catch (const std::exception& e) { cerr << "generic"; }      // WRONG ORDER: this catches EVERYTHING first
catch (const std::out_of_range& e) { cerr << "specific"; }  // UNREACHABLE — never actually runs
```
- **Writing your own custom exception classes** deriving from `std::exception` (or a more specific standard subclass like `std::runtime_error`) is standard, idiomatic practice — it lets you attach whatever additional context/data your specific error needs (beyond just a string message) while still being catchable generically via `std::exception&` by code that doesn't care about the specifics.
- **Preferring RAII members over manual resource management inside a class that might throw** (explicitly emphasized): if a class's constructor performs manual resource acquisition (e.g. raw `new`) and then THROWS partway through (e.g. after successfully allocating one resource but failing to acquire a second), the class itself becomes responsible for tricky manual cleanup of the partially-constructed state. Delegating resource ownership to RAII-COMPLIANT MEMBERS instead (e.g. a `std::unique_ptr` member instead of a raw pointer member) means that if the constructor throws, already-constructed member subobjects (like that unique_ptr) still have their OWN destructors correctly invoked as part of the failed object's teardown — no manual cleanup code needed in the class itself.

## 27.6 Rethrowing exceptions
- Inside a `catch` block, you can choose to `throw;` (with NO expression) to RE-THROW the SAME exception currently being handled, propagating it further UP the call stack to an even-higher-level handler — useful when a catch block wants to do some LOCAL cleanup/logging but ultimately still let a higher-level handler deal with the actual error.
```cpp
catch (const std::exception& e) {
    logError(e.what());
    throw;  // re-throw the SAME original exception object (preserves its original type/data)
}
```
- **Critical, frequently-tested distinction**: bare `throw;` (rethrowing) preserves the ORIGINAL exception's full information (including its actual DERIVED type, even if it was caught via a base-class reference) — whereas `throw e;` (re-throwing by explicitly naming the caught variable) creates a NEW exception by COPYING `e`, which — if `e` is a base-class reference/type — can cause **object slicing** (Ch 25.9) if the original exception was actually of some MORE-DERIVED type: the re-thrown copy would be sliced down to just the base type, losing derived-specific information. Always prefer bare `throw;` for rethrowing the exact original exception.
- A newly `throw`n exception INSIDE a catch block is considered to occur OUTSIDE the try block that catch block itself belongs to — so it will NOT be caught by any OTHER catch handler attached to that SAME try block; it propagates further up the stack, exactly like a fresh throw would from ordinary code.

## 27.7 Function try blocks
- A specialized syntax allowing a `try`/`catch` to wrap an ENTIRE function body (most notably useful for CONSTRUCTOR member-initializer lists, which normally can't be wrapped by a try block placed inside the function body in the usual way, since initializer-list execution happens BEFORE the body):
```cpp
class Foo {
public:
    Foo(int x) try : m_resource(x) {  // function try block wrapping the constructor
        // constructor body
    } catch (...) {
        // handle exceptions from the member-initializer list OR the constructor body
        throw; // constructors generally MUST rethrow (or throw something else) — see below
    }
};
```
- **Important, frequently-cited limitation**: a function-try-block's catch clause attached to a CONSTRUCTOR cannot "swallow" the exception and let construction proceed normally — the object is considered to have FAILED to construct regardless of what the catch block does; the catch block can only perform cleanup/logging and then MUST (implicitly, if it doesn't explicitly throw something else) rethrow the original exception (or a new one) once it finishes — you can't recover and return a "successfully constructed" object from this handler.
- Relatively niche/rarely used outside this constructor-initializer-list use case, but worth recognizing the syntax and its specific constructor-related purpose if it appears in an interview/OA.

## 27.8 Exception dangers and downsides
- Explicitly-catalogued CAUTIONS worth being able to recite for a "what are the risks/costs of using exceptions" theory question:
  1. **Cleanup obligations don't disappear just because you're using exceptions** — RESOURCES ACQUIRED via manual `new`/raw handles still must be properly released during unwinding, and NAIVE manual cleanup code is easy to get wrong across all the various paths an exception might take; the recommended fix (heavily emphasized) is to prefer STACK-ALLOCATED RAII objects / smart pointers (`std::unique_ptr`, etc.) specifically BECAUSE their destructors are GUARANTEED to run correctly during unwinding, removing the need for manual, error-prone cleanup code scattered through every possible throwing code path.
  2. **Destructors should (almost) NEVER throw exceptions** — if a destructor throws WHILE the stack is ALREADY unwinding due to a DIFFERENT, earlier exception (e.g., an exception thrown inside a catch block or during unwinding itself), the program calls `std::terminate()` IMMEDIATELY, since C++ has no defined mechanism for handling two simultaneous, competing exceptions. Destructors are implicitly `noexcept` by default in modern C++ specifically to help enforce/formalize this expectation (27.9 covers `noexcept` fully).
  3. **Performance overhead**, though generally SMALL and largely confined to the actual throw/unwind path itself in modern implementations (the "zero-cost exceptions" model used by most modern compilers imposes essentially NO overhead on the NON-throwing/happy path, at the cost of a comparatively more expensive throw/catch operation when an exception genuinely occurs) — this nuance ("cheap when not thrown, more expensive when actually thrown") is itself a good, commonly-tested talking point.
  4. **Exception SPECIFICATIONS/design complexity**: knowing exactly WHICH exceptions a given function might throw isn't always obvious from its signature alone (unlike, say, Java's checked exceptions) — this can make exception-safety reasoning genuinely harder in large codebases, a fair critique to acknowledge in a balanced discussion.

## 27.9 Exception specifications and `noexcept`
- `noexcept` (a function specifier, distinct from a runtime exception-handling mechanism itself) DOCUMENTS — and, more importantly, semi-ENFORCES — that a function is NOT expected to throw ANY exception:
```cpp
void mustNotThrow() noexcept { /* ... */ }
```
- **If a `noexcept` function DOES somehow throw anyway** (e.g. it calls something else that throws, and doesn't catch it internally), `std::terminate()` is called IMMEDIATELY — the exception does NOT propagate normally to an outer try/catch the way it would from a non-noexcept function; this is a genuinely important, frequently-tested behavioral difference (marking something `noexcept` isn't just documentation — violating that promise is fatal, not merely "surprising").
- **Why mark something `noexcept`, concretely (beyond documentation)**: certain STANDARD LIBRARY operations specifically check whether a type's MOVE constructor/move assignment is `noexcept` to decide whether it's SAFE to use moves in a context that must offer a strong exception-safety guarantee — e.g. `std::vector`'s reallocation logic (Ch 16.10) will only use MOVE semantics on its elements during reallocation if the element type's move constructor is `noexcept`; otherwise, it conservatively falls back to using the (slower, but exception-safety-preserving) COPY constructor instead, specifically to avoid ending up with a partially-moved, inconsistent vector state if a mid-reallocation move were to throw partway through. This is precisely why `noexcept` matters PRACTICALLY, not just as documentation — MARKING your move constructor/assignment `noexcept` (when it genuinely can't throw, which is the common case for typical resource-stealing moves) directly enables real performance optimizations elsewhere in the standard library.
- Destructors are IMPLICITLY `noexcept` by default (reinforcing 27.8's point #2) unless you explicitly opt out (`noexcept(false)`, rarely done).

## 27.10 `std::move_if_noexcept`
- A standard library utility (already flagged briefly in the Ch 22 notes) that conditionally returns either an rvalue-cast (enabling a MOVE) OR an lvalue (forcing a COPY) of its argument, DEPENDING on whether the argument's type has a `noexcept` move constructor:
- **Precise logic**: if `T`'s move constructor IS `noexcept` (or `T` has NO copy constructor at all, meaning there's no safer fallback anyway), `std::move_if_noexcept` behaves like `std::move` (enables a move). Otherwise (move constructor exists but is NOT noexcept, and a copy constructor IS available as a fallback), it returns an lvalue instead, forcing a COPY rather than risking a potentially-throwing move.
- **Purpose, directly tied to 27.9's `std::vector` reallocation discussion**: used internally by the standard library (and available for your own generic code) specifically to preserve the STRONG EXCEPTION SAFETY GUARANTEE in scenarios like vector reallocation — if a move partway through reallocating a vector's elements were to THROW, the vector could be left in a corrupted, partially-moved-from state with no way to safely roll back; using a COPY instead (when the move isn't provably safe/noexcept) means that if something throws mid-copy, the ORIGINAL elements are all still intact and the operation can be safely abandoned/rolled back, preserving the vector's original valid state.

**Likely interview Qs for Ch 27:** How does exception handling decouple error detection from error handling, compared to return codes? What is stack unwinding, and why does it make RAII essential for exception safety? Why must derived exception catch handlers be listed BEFORE base-class handlers? What's the difference between `throw;` and `throw e;` when rethrowing, and why does one risk object slicing? Why should destructors basically never throw? What does marking a function `noexcept` actually change if it throws anyway? Why does a type's move constructor being `noexcept` matter for `std::vector`'s performance?

---
# CHAPTER 28 — Input and Output (I/O)

## 28.1 Input and output (I/O) streams
- C++'s I/O system is built around the concept of a **stream** — an abstraction representing a SEQUENCE of bytes flowing either INTO a program (input stream) or OUT of it (output stream), decoupling the actual SOURCE/DESTINATION (console, file, in-memory string, network socket, etc.) from the CODE that reads/writes it — the same `<<`/`>>` operators and general interface work uniformly whether you're writing to the console, a file, or a string.
- The class hierarchy (worth recognizing the NAMES/relationships, even if the full inheritance diagram is rarely quizzed in detail): `std::ios_base` / `std::ios` sit at the root, providing shared FORMATTING/STATE functionality; `std::istream` (input) and `std::ostream` (output) derive from this shared base; `std::iostream` derives from BOTH (supporting bidirectional I/O). The familiar global objects — `std::cin` (an istream), `std::cout`/`std::cerr`/`std::clog` (ostreams) — are simply pre-constructed INSTANCES of these classes, already wired up to the console's standard input/output/error streams.
- **`std::cerr` vs `std::clog` vs `std::cout` (a commonly-asked distinction)**: `cout` is for normal program OUTPUT, and is BUFFERED (may not display immediately). `cerr` is specifically for ERROR messages, and is UNBUFFERED by default (flushes immediately after each insertion — important for ensuring error messages appear promptly, even if the program crashes right after). `clog` is for LOGGING messages, buffered like cout, but conventionally kept separate from normal program output.

## 28.2 Input with `istream`
- Covers `std::cin`'s member functions beyond the basic `>>` operator (already deeply covered in the earlier notes' I/O quiz section, Ch 9.5): `.get()` (extracts a SINGLE character, INCLUDING whitespace — unlike `>>`, which skips leading whitespace by default), `.getline()` (reads an entire line into a C-style char buffer, up to a specified maximum length or a delimiter — the member-function counterpart to the free function `std::getline` used with `std::string`, Ch 17.10-adjacent), `.ignore()` (discards characters, already covered extensively in the cin-failure-recovery pattern from Ch 9.5), and `.peek()` (looks at the NEXT character WITHOUT actually extracting/consuming it from the stream — useful for lookahead-style parsing logic).
- **`>>` (formatted extraction) SKIPS leading whitespace by default before extracting**; `.get()` (unformatted extraction) does NOT skip whitespace — this distinction directly explains WHY mixing `cin >> x` with `cin.get(ch)` in the same program can produce surprising results if you don't account for it (e.g. after `cin >> x`, the very next `.get()` call may immediately consume the leftover newline character rather than the next "real" input character — closely related to the classic `cin >>` / `getline` interaction bug already covered in the earlier I/O notes).

## 28.3 Output with `ostream` and `ios`
- Covers STREAM MANIPULATORS (from `<iomanip>` and `<ios>`) that control OUTPUT FORMATTING — several already introduced piecemeal in earlier chapters, formalized here: `std::setw(n)` (sets the MINIMUM field width for the NEXT single insertion only — a one-shot manipulator, unlike most others which persist until changed), `std::setfill(c)` (sets the PADDING character used when a field is widened via setw, default is space), `std::left`/`std::right`/`std::internal` (alignment within a padded field), `std::setprecision(n)` (already covered — controls floating-point precision, interacting with `std::fixed`/`std::scientific`), `std::boolalpha` (prints `bool`s as "true"/"false" instead of 1/0).
- **`std::setw` is a one-shot manipulator (frequently-tested gotcha)**: unlike `std::fixed`, `std::setprecision`, etc. (which PERSIST for all subsequent output until explicitly changed again), `std::setw` applies ONLY to the very NEXT single value inserted, then automatically reverts — a common mistake is expecting a single `setw` call to apply to MULTIPLE subsequent outputs in a loop, when it must actually be re-specified before EACH one.
- `std::ios::fmtflags` and direct flag manipulation (`.setf()`/`.unsetf()`) provide a lower-level, more verbose alternative to the manipulator syntax — manipulators are generally preferred for readability in modern code, but recognizing that manipulators are essentially convenient SYNTACTIC SUGAR over setting underlying stream FLAGS is a good conceptual point to have ready.

## 28.4 Stream classes for strings — `std::stringstream`
- Recap/formalization of material already introduced in the Ch 21.4/I/O-quiz context: `std::stringstream` (and its input-only/output-only counterparts `std::istringstream`/`std::ostringstream`, header `<sstream>`) provide an IN-MEMORY stream that behaves just like `cin`/`cout` syntactically, but reads from / writes to an internal `std::string` buffer instead of the console.
- **Primary use cases explicitly highlighted**: (1) PARSING — splitting/converting a line of mixed-type text input into individual typed values, using the exact same `>>` extraction syntax and whitespace-skipping behavior as `cin` (e.g. tokenizing `"5 3.14 hello"` into an int, double, and string); (2) BUILDING formatted strings incrementally via `<<` insertion, then extracting the final result with `.str()`, as a more readable alternative to repeated raw string concatenation when the message involves many interleaved pieces of different types.
- `.str()` retrieves the stringstream's current full buffer contents as a `std::string`; `.str(newValue)` RESETS the buffer's contents (note: this does NOT automatically reset the stream's internal read/write POSITION or any error flags — a subtle, occasionally-tested gotcha if you're reusing the same stringstream object repeatedly, needing an explicit `.clear()` and/or a fresh position alongside `.str("")`).

## 28.5 Stream states and input validation
- Formalizes the state-flag mechanics already covered extensively in the earlier I/O-focused MCQ material — worth having the EXACT flag NAMES memorized for a direct "what are the stream state flags" question:
  - **`goodbit`**: no errors — everything is fine (`.good()` returns true when this is the ONLY bit set, i.e., no other error bits are also set).
  - **`eofbit`**: end-of-input has been reached (`.eof()`).
  - **`failbit`**: a (typically recoverable) formatting/extraction error occurred — e.g. attempting to extract a non-numeric token into a numeric variable (`.fail()`).
  - **`badbit`**: a more SERIOUS, typically UNRECOVERABLE error at the underlying stream/system level (e.g. a genuine I/O hardware failure) — distinguished from failbit as being a more fundamental problem (`.bad()`).
- **`.clear()`** resets ALL error state flags back to `goodbit` (necessary before the stream can be used again normally after a failure) — this does NOT, by itself, discard any BAD data still sitting in the input buffer; that's what `.ignore()` is separately for (the full, standard recovery IDIOM combining both — `cin.clear(); cin.ignore(numeric_limits<streamsize>::max(), '\n');` — was already covered in depth in the Ch 9.5 notes and the earlier MCQ quiz).
- This lesson formalizes/reinforces the EXACT same input-validation patterns and gotchas already extensively covered (partial extraction leaving "Frankenstein" object state, needing to check BOTH extraction success AND semantic validity separately, etc.) — worth treating this lesson as the canonical, formal reference point for all of that material rather than genuinely new content.

## 28.6 Basic file I/O
- `std::ifstream` (input file stream), `std::ofstream` (output file stream), `std::fstream` (bidirectional) — header `<fstream>` — behave SYNTACTICALLY just like `cin`/`cout` (same `>>`/`<<`/`.getline()`/etc. interface), but read from / write to an actual FILE on disk instead of the console.
```cpp
std::ofstream outFile("data.txt");   // opens (creating/truncating) the file for writing
if (!outFile) { /* handle failure to open, e.g. permissions or invalid path */ }
outFile << "Hello, file!\n";
// outFile's destructor automatically closes the file when it goes out of scope (RAII!)
```
- **File streams are RAII-COMPLIANT** — this is EXPLICITLY highlighted (and directly connects back to the Ch 27.5 discussion of preferring RAII members over manual resource management): the file is automatically, safely CLOSED when the stream OBJECT's destructor runs (going out of scope), even if the function exits early via an exception or an early return — you do NOT need to manually call `.close()` in most cases (though it CAN be called explicitly if you want to close the file before the object's natural end-of-scope, e.g. to release it for another process to use sooner).
- **Always check whether a file stream successfully OPENED before using it** — the file might not exist (for reading), or the path/permissions might be invalid (for writing) — checking `if (!fileStream)` (or `.is_open()` explicitly) immediately after construction/opening is the idiomatic, expected pattern; failing to check and then proceeding to read/write anyway leads to silent failure (extraction/insertion operations on a failed stream simply fail, similar to any other stream failure state, rather than crashing outright) — a frequently-tested "what's missing from this file-handling code" gotcha.
- Opening MODES (passed as a second constructor argument, e.g. `std::ofstream("data.txt", std::ios::app)`) control behavior like APPENDING to an existing file (`std::ios::app`) vs. the DEFAULT of TRUNCATING/overwriting it, opening in BINARY mode (`std::ios::binary`, disabling text-mode newline translation) vs. the default TEXT mode, etc.

## 28.7 Random file I/O
- By default, file streams read/write SEQUENTIALLY from a current internal POSITION that automatically advances after each operation — **random file I/O** lets you explicitly JUMP to an arbitrary position within the file rather than only reading/writing strictly in order.
- **`.seekg(pos)`** (seek for GET/input) and **`.seekp(pos)`** (seek for PUT/output) reposition the stream's internal read/write pointer — note the SEPARATE get vs. put positions (`g` for input streams, `p` for output streams; an `fstream`, being bidirectional, tracks BOTH independently) — a detail occasionally tested ("why are there two different seek functions").
- **`.tellg()`/`.tellp()`** return the CURRENT position (useful for e.g. recording a position to seek back to later, or computing a file's total size by seeking to the end and checking the position there).
- Seek positions can be specified as an ABSOLUTE offset from the beginning of the file, or RELATIVE to a reference point (`std::ios::beg`, `std::ios::cur`, `std::ios::end`) via an overload taking an offset PLUS a direction: `file.seekg(-10, std::ios::end);` seeks to 10 bytes before the end of the file.
- Explicitly flagged as a MORE ADVANCED/NICHE topic — most everyday file-processing code just reads/writes sequentially from start to finish and never needs random access — but worth recognizing the function NAMES and general PURPOSE (e.g. for a "how would you read the last N bytes of a large file without reading the whole thing" style question).

---

## Quick Cross-Chapter Traps to Memorize (Ch 26–28)

| Topic | The trap |
|---|---|
| Template class member functions | Must stay visible in the header (or equivalent) — splitting into a separate .cpp causes a linker error |
| Static members of a class template | NOT shared across different instantiations — `Foo<int>::x` and `Foo<double>::x` are separate variables |
| Partial specialization | Only legal for CLASS templates, never function templates — use overloading instead for functions |
| Uncaught exceptions | Calls std::terminate() — stack unwinding isn't guaranteed to complete properly in this case |
| Catch handler ordering | Derived exception types must be caught BEFORE base types, or the base catch steals everything |
| Rethrowing | `throw;` preserves the original type; `throw e;` can slice a derived exception down to its caught base type |
| Destructors + exceptions | A destructor throwing during unwinding from another exception calls std::terminate() immediately |
| `noexcept` violated | Calls std::terminate() immediately — does NOT propagate to an outer catch like a normal throw would |
| `std::vector` + move | Only uses move during reallocation if the element's move constructor is noexcept; otherwise falls back to copy |
| `std::setw` | One-shot — applies only to the very next inserted value, unlike most other manipulators |
| File streams | RAII — destructor auto-closes the file; always check the stream opened successfully before using it |
| `.seekg` vs `.seekp` | Separate get/put positions — relevant specifically for bidirectional fstream objects |
