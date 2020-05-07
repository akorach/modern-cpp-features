# C++17

## Overview
Many of these descriptions and examples come from various resources (see [Acknowledgements](#acknowledgements) section), summarized in my own words.

C++17 includes the following new language features:
- [new rules for auto deduction from braced-init-list](#new-rules-for-auto-deduction-from-braced-init-list)
- [nested namespaces](#nested-namespaces)
- [utf-8 character literals](#utf-8-character-literals)
- [folding expressions](#folding-expressions)
- [template argument deduction for class templates](#template-argument-deduction-for-class-templates)
- [non-type template parameters with auto type](#non-type-template-parameters-with-auto-type)
- [constexpr lambda](#constexpr-lambda)
- [lambda capture this by value](#lambda-capture-this-by-value)
- [inline variables](#inline-variables)
- [structured bindings](#structured-bindings)
- [selection statements with initializer](#selection-statements-with-initializer)
- [constexpr if](#constexpr-if)
- [direct-list-initialization of enums](#direct-list-initialization-of-enums)
- [fallthrough, nodiscard, maybe_unused attributes](#fallthrough-nodiscard-maybe_unused-attributes)

C++17 includes the following new library features:
- [std::variant](#stdvariant)
- [std::optional](#stdoptional)
- [std::any](#stdany)
- [std::string_view](#stdstring_view)
- [std::invoke](#stdinvoke)
- [std::apply](#stdapply)
- [std::filesystem](#stdfilesystem)
- [std::byte](#stdbyte)
- [splicing for maps and sets](#splicing-for-maps-and-sets)
- [parallel algorithms](#parallel-algorithms)

## C++17 Language Features

### New rules for auto deduction from braced-init-list
Changes to `auto` deduction when used with the uniform initialization syntax. Previously, `auto x {3};` deduced a `std::initializer_list<int>`, now it's deduced as `int`.
```c++
auto x1 = {1, 2.0};  // error: cannot deduce element type
auto x2 = {1, 2, 3}; // x2 is std::initializer_list<int>
auto x3 {1, 2, 3};   // error: not a single element
auto x4 = { 3 };     // decltype(x4) is std::initializer_list<int>
auto x5 {3};         // x3 is int
```

### static_assert with no message
Self-explanatory - `static_assert` is no longer required to be called with a message. A condition is enough.

### typename in a template template parameter
Allows you to use `typename` instead of `class` when declaring a template template parameter. Normal type parameters can use them interchangeably, but template template parameters were restricted to `class`, so this change unifies these forms somewhat.
```c++
template<typename T> struct A {};
template<typename T> using B = int;

Before:
template<template<typename> class X> struct C;
C<A> ca; // ok
C<B> cb; // ok, not a class template

After:
template<template<typename> typename X> struct C;
C<A> ca; // ok
C<B> cb; // ok, not a class template
```
After: [BFilipek](https://www.bfilipek.com/2017/01/cpp17features.html#typename-in-a-template-template-parameter)

### Removing trigraphs
What trigraphs are: 
The source character set of C source programs is contained within the 7-bit ASCII character set but is a superset of the ISO 646-1983 Invariant Code Set. Trigraph sequences allow C programs to be written using only the ISO (International Standards Organization) Invariant Code Set. Trigraphs are sequences of three characters (introduced by two consecutive question marks) that the compiler replaces with their corresponding punctuation characters. You can use trigraphs in C source files with a character set that does not contain convenient graphic representations for some punctuation characters.

C++17 removes trigraphs from the language. Implementations may continue to support trigraphs as part of the implementation-defined mapping from the physical source file to the basic source character set, though the standard encourages implementations not to do so. Through C++14, trigraphs are supported as in C.

From [MSDN Trigraphs](https://docs.microsoft.com/en-us/cpp/c-language/trigraphs?redirectedfrom=MSDN&view=vs-2019)

Trigraphs will no longer have to be escaped, nor will they be recognised in code.


### Nested namespaces
Allows you to introduce namespaces with a shortened syntax.
```c++
namespace A { namespace B { namespace C {
 int i;
}}}

// vs.
namespace A::B::C {
  int i;
}
```

### Attributes for namespaces and enumerators
Attributes on enumerators and namespaces are now permitted.
```c++
enum E {
  foo = 0,
  bar [[deprecated]] = foo
};

E e = bar; // Emits warning

namespace [[deprecated]] oldStuff{
    void legacy();
}

oldStuff::legacy(); // Emits warning
```
From: [BFilipek](https://www.bfilipek.com/2017/01/cpp17features.html#attributes-for-namespaces-and-enumerators)


### UTF-8 character literals
A character literal that begins with `u8` is a character literal of type `char`. The value of a UTF-8 character literal is equal to its ISO 10646 code point value.
```c++
char x = u8'x';
```
Rationale: [StackOverflow](https://stackoverflow.com/questions/31970111/what-is-the-point-of-the-utf-8-character-literals-proposed-for-c17)


### Allow constant evaluation for all non-type template arguments
Remove syntactic restrictions for pointers, references, and pointers to members that appear as non-type template parameters. For instance:
```c++
template<int *p> struct A {};
int n;
A<&n> a; // ok

constexpr int *p() { return &n; }
A<p()> b; // error before C++17
```
From: [BFilipek](https://www.bfilipek.com/2017/01/cpp17features.html#allow-constant-evaluation-for-all-non-type-template-arguments)


### Fold expressions
A fold expression performs a fold of a template parameter pack over a binary operator.
* An expression of the form `(... op e)` or `(e op ...)`, where `op` is a fold-operator and `e` is an unexpanded parameter pack, are called _unary folds_.
* An expression of the form `(e1 op ... op e2)`, where `op` are fold-operators, is called a _binary fold_. Either `e1` or `e2` is an unexpanded parameter pack, but not both.
Allows to avoid explicit use of recursion.
```c++
template <typename... Args>
bool logicalAnd(Args... args) {
    // Binary folding.
    return (true && ... && args);
}
bool b = true;
bool& b2 = b;
logicalAnd(b, b2, true); // == true
```
```c++
template <typename... Args>
auto sum(Args... args) {
    // Unary folding.
    return (... + args);
}
sum(1.0, 2.0f, 3); // == 6.0
```

### Unary fold expressions and empty parameter packs
If the parameter pack is empty then the value of the fold is:
```c++
&& // true
|| // false
,  // void()
```
For any operator not listed above, a unary fold expression with an empty parameter pack is ill-formed.
From: [BFilipek](https://www.bfilipek.com/2017/01/cpp17features.html#unary-fold-expressions-and-empty-parameter-packs)


### Make exception specifications part of the type system
Previously exception specifications for a function weren't part of the function's type. This changed in C++17 and now we’ll get an error in this case:
```c++
void (*p)();
void (**pp)() noexcept = &p;   // error: cannot convert to pointer to noexcept function

struct S { typedef void (*p)(); operator p(); };
void (*q)() noexcept = S();   // error: cannot convert to pointer to noexcept function
```
From: [BFilipek](https://www.bfilipek.com/2017/01/cpp17features.html#make-exception-specifications-part-of-the-type-system)


### Aggregate initialization of classes with base classes
If a class was derived from some other type using aggregate initialisation wasn't possible. But now the restriction is removed.
```c++
struct Base { int a1, a2; };
struct Derived : Base { int b1; };

Derived d1{{1, 2}, 3};      // full explicit initialization
Derived d1{{}, 1};          // the base is value initialized
```
From: [BFilipek](https://www.bfilipek.com/2017/01/cpp17features.html#aggregate-initialization-of-classes-with-base-classes)


### \_\_has_include in preprocessor conditionals
This feature allows a C++ program to directly, reliably and portably determine whether or not a library header is available for inclusion.

Example: This demonstrates a way to use a library optional facility only if it is available.
```c++
#if __has_include(<optional>)
#  include <optional>
#  define have_optional 1
#elif __has_include(<experimental/optional>)
#  include <experimental/optional>
#  define have_optional 1
#  define experimental_optional 1
#else
#  define have_optional 0
#endif
```

### Lambda capture of `*this`
Capturing `this` in a lambda's environment was previously reference-only. An example of where this is problematic is asynchronous code using callbacks that require an object to be available, potentially past its lifetime. `*this` (C++17) will now make a copy of the current object, while `this` (C++11) continues to capture by reference.
```c++
struct MyObj {
  int value {123};
  auto getValueCopy() {
    return [*this] { return value; };
  }
  auto getValueRef() {
    return [this] { return value; };
  }
};
MyObj mo;
auto valueCopy = mo.getValueCopy();
auto valueRef = mo.getValueRef();
mo.value = 321;
valueCopy(); // 123
valueRef(); // 321
```

### Direct list initialization of enums
Allows to initialize enum class with a fixed underlying type without the necessity to resort to casts:
```c++
enum class Handle : uint32_t { Invalid = 0 };
Handle h { 42 }; // OK
```
Allows to introduce easy-to-use strong-typed type aliases.


### constexpr lambda
Compile-time lambdas using `constexpr`.
```c++
auto identity = [](int n) constexpr { return n; };
static_assert(identity(123) == 123);
```
```c++
constexpr auto add = [](int x, int y) {
  auto L = [=] { return x; };
  auto R = [=] { return y; };
  return [=] { return L() + R(); };
};

static_assert(add(1, 2)() == 3);
```
```c++
constexpr int addOne(int n) {
  return [n] { return n + 1; }();
}

static_assert(addOne(1) == 2);
```

### [[fallthrough]] attribute
`[[fallthrough]]` indicates to the compiler that falling through in a switch statement is intended behavior.
```c++
int a = 1;
switch (n) {
  case 1:
    ++a;
    [[fallthrough]];
  case 2:
    ++a;
    break;
}
```


### [[nodiscard]] attribute
`[[nodiscard]]` issues a warning when the return value of a function with this attribute is discarded.
```c++
[[nodiscard]] bool do_something() {
  return is_success; // true for success, false for failure
}

do_something(); // warning: ignoring return value of 'bool do_something()',
                // declared with attribute 'nodiscard'
```

`[[nodiscard]]` can also refer to a class. A warning is then issued whenever an object of this type is returned by value and the value is discarded.
```c++
// Only issues a warning when `error_info` is returned by value.
struct [[nodiscard]] error_info {
  // ...
};

error_info do_something() {
  error_info ei;
  // ...
  return ei;
}

do_something(); // warning: ignoring returned value of type 'error_info',
                // declared with attribute 'nodiscard'
```

### [[maybe_unused]] attribute
`[[maybe_unused]]` indicates to the compiler that a variable or parameter might be intentionally unused.
```c++
void my_callback(std::string msg, [[maybe_unused]] bool error) {
  // Don't care if `msg` is an error message, just log it.
  log(msg);
}
```
An example use case is when you want to provide two platform-dependent implementations of a function, one of which doesn't make use of one of the arguments.


### Hexadecimal floating-point literals
Allows to express some special floating point values, for example, the smallest normal IEEE-754 single precision value is readily written as `0x1.0p-126`.
```c++
double d = 0x1.2p3; // hex fraction 1.2 (decimal 1.125) scaled by 2^3, that is 9.0
```

### Using attribute namespaces without repetition
Simplifies the case where you want to use multiple attributes:
```c++
void f() {
    [[rpr::kernel, rpr::target(cpu,gpu)]] // repetition
    do-task();
}
```
After the change:
```c++
void f() {
    [[using rpr: kernel, target(cpu,gpu)]]
    do-task();
}
```
From: [BFilipek](https://www.bfilipek.com/2017/01/cpp17features.html#using-attribute-namespaces-without-repetition)

### Template argument deduction for class templates
Automatic template argument deduction much like how it's done for functions, but now including class constructors.
```c++
template <typename T = float>
struct MyContainer {
  T val;
  MyContainer() : val{} {}
  MyContainer(T val) : val{val} {}
  // ...
};
MyContainer c1 {1}; // OK MyContainer<int>
MyContainer c2; // OK MyContainer<float>
```
Second example:
```c++
void foo(std::pair<int, char>);

// Before C++17
foo(std::pair<int, char>(42, 'z'));

// C++17
foo(std::pair(42, 'z'));
```

### Non-type template parameters with auto type
Automatically deduce the type of non-type template parameters:
```c++
template <auto value> void f() { }
f<10>();               // deduces int
```

### Guaranteed copy elision
Copy elision for temporaries (prvalues), not for Named RVO, where glvalues are involved.
See: https://jonasdevlieghere.com/guaranteed-copy-elision/


### Inline variables
The inline specifier can be applied to variables as well as to functions. A variable declared inline has the same semantics as a function declared inline.
```c++
// Disassembly example using compiler explorer.
struct S { int x; };
inline S x1 = S{321}; // mov esi, dword ptr [x1]
                      // x1: .long 321

S x2 = S{123};        // mov eax, dword ptr [.L_ZZ4mainE2x2]
                      // mov dword ptr [rbp - 8], eax
                      // .L_ZZ4mainE2x2: .long 123
```

It can also be used to declare and define a static member variable, such that it does not need to be initialized in the source file.
```c++
struct S {
  S() : id{count++} {}
  ~S() { count--; }
  int id;
  static inline int count{0}; // declare and initialize count to 0 within the class
};
```

### Structured bindings
A proposal for de-structuring initialization, that would allow writing `auto [ x, y, z ] = expr;` where the type of `expr` was a tuple-like object, whose elements would be bound to the variables `x`, `y`, and `z` (which this construct declares). _Tuple-like objects_ include `std::tuple`, `std::pair`, `std::array`, and aggregate structures.
```c++
using Coordinate = std::pair<int, int>;
Coordinate origin() {
  return Coordinate{0, 0};
}

const auto [ x, y ] = origin();
x; // == 0
y; // == 0
```
```c++
std::unordered_map<std::string, int> mapping {
  {"a", 1},
  {"b", 2},
  {"c", 3}
};

// Destructure by reference.
for (const auto& [key, value] : mapping) {
  // Do something with key and value
}
```

### Selection statements with initializer
New versions of the `if` and `switch` statements which simplify common code patterns and help users keep scopes tight.
```c++
{
  std::lock_guard<std::mutex> lk(mx);
  if (v.empty()) v.push_back(val);
}
// vs.
if (std::lock_guard<std::mutex> lk(mx); v.empty()) {
  v.push_back(val);
}
```
```c++
Foo gadget(args);
switch (auto s = gadget.status()) {
  case OK: gadget.zip(); break;
  case Bad: throw BadFoo(s.message());
}
// vs.
switch (Foo gadget(args); auto s = gadget.status()) {
  case OK: gadget.zip(); break;
  case Bad: throw BadFoo(s.message());
}
```

### constexpr if
Write code that is instantiated depending on a compile-time condition.
```c++
template <typename T>
constexpr bool isIntegral() {
  if constexpr (std::is_integral<T>::value) {
    return true;
  } else {
    return false;
  }
}
static_assert(isIntegral<int>() == true);
static_assert(isIntegral<char>() == true);
static_assert(isIntegral<double>() == false);
struct S {};
static_assert(isIntegral<S>() == false);
```


## C++17 Library Features

### std::variant
The class template `std::variant` represents a type-safe `union`. An instance of `std::variant` at any given time holds a value of one of its alternative types (it's also possible for it to be valueless).
```c++
std::variant<int, double> v{ 12 };
std::get<int>(v); // == 12
std::get<0>(v); // == 12
v = 12.0;
std::get<double>(v); // == 12.0
std::get<1>(v); // == 12.0
```

### std::optional
The class template `std::optional` manages an optional contained value, i.e. a value that may or may not be present. A common use case for optional is the return value of a function that may fail.
```c++
std::optional<std::string> create(bool b) {
  if (b) {
    return "Godzilla";
  } else {
    return {};
  }
}

create(false).value_or("empty"); // == "empty"
create(true).value(); // == "Godzilla"
// optional-returning factory functions are usable as conditions of while and if
if (auto str = create(true)) {
  // ...
}
```

### std::any
A type-safe container for single values of any type.
```c++
std::any x {5};
x.has_value() // == true
std::any_cast<int>(x) // == 5
std::any_cast<int&>(x) = 10;
std::any_cast<int>(x) // == 10
```

### std::string_view
A non-owning reference to a string. Useful for providing an abstraction on top of strings (e.g. for parsing).
```c++
// Regular strings.
std::string_view cppstr {"foo"};
// Wide strings.
std::wstring_view wcstr_v {L"baz"};
// Character arrays.
char array[3] = {'b', 'a', 'r'};
std::string_view array_v(array, std::size(array));
```
```c++
std::string str {"   trim me"};
std::string_view v {str};
v.remove_prefix(std::min(v.find_first_not_of(" "), v.size()));
str; //  == "   trim me"
v; // == "trim me"
```

### std::invoke
Invoke a `Callable` object with parameters. Examples of `Callable` objects are `std::function` or `std::bind` where an object can be called similarly to a regular function.
```c++
template <typename Callable>
class Proxy {
  Callable c;
public:
  Proxy(Callable c): c(c) {}
  template <class... Args>
  decltype(auto) operator()(Args&&... args) {
    // ...
    return std::invoke(c, std::forward<Args>(args)...);
  }
};
auto add = [](int x, int y) {
  return x + y;
};
Proxy<decltype(add)> p {add};
p(1, 2); // == 3
```

### std::apply
Invoke a `Callable` object with a tuple of arguments.
```c++
auto add = [](int x, int y) {
  return x + y;
};
std::apply(add, std::make_tuple(1, 2)); // == 3
```

### std::filesystem
The new `std::filesystem` library provides a standard way to manipulate files, directories, and paths in a filesystem.

Here, a big file is copied to a temporary path if there is available space:
```c++
const auto bigFilePath {"bigFileToCopy"};
if (std::filesystem::exists(bigFilePath)) {
  const auto bigFileSize {std::filesystem::file_size(bigFilePath)};
  std::filesystem::path tmpPath {"/tmp"};
  if (std::filesystem::space(tmpPath).available > bigFileSize) {
    std::filesystem::create_directory(tmpPath.append("example"));
    std::filesystem::copy_file(bigFilePath, tmpPath.append("newFile"));
  }
}
```

### std::byte
The new `std::byte` type provides a standard way of representing data as a byte. Benefits of using `std::byte` over `char` or `unsigned char` is that it is not a character type, and is also not an arithmetic type; while the only operator overloads available are bitwise operations.
```c++
std::byte a {0};
std::byte b {0xFF};
int i = std::to_integer<int>(b); // 0xFF
std::byte c = a & b;
int j = std::to_integer<int>(c); // 0
```
Note that `std::byte` is simply an enum, and braced initialization of enums become possible thanks to [direct-list-initialization of enums](#direct-list-initialization-of-enums).

### Splicing for maps and sets
Moving nodes and merging containers without the overhead of expensive copies, moves, or heap allocations/deallocations.

Moving elements from one map to another:
```c++
std::map<int, string> src {{1, "one"}, {2, "two"}, {3, "buckle my shoe"}};
std::map<int, string> dst {{3, "three"}};
dst.insert(src.extract(src.find(1))); // Cheap remove and insert of { 1, "one" } from `src` to `dst`.
dst.insert(src.extract(2)); // Cheap remove and insert of { 2, "two" } from `src` to `dst`.
// dst == { { 1, "one" }, { 2, "two" }, { 3, "three" } };
```

Inserting an entire set:
```c++
std::set<int> src {1, 3, 5};
std::set<int> dst {2, 4, 5};
dst.merge(src);
// src == { 5 }
// dst == { 1, 2, 3, 4, 5 }
```

Inserting elements which outlive the container:
```c++
auto elementFactory() {
  std::set<...> s;
  s.emplace(...);
  return s.extract(s.begin());
}
s2.insert(elementFactory());
```

Changing the key of a map element:
```c++
std::map<int, string> m {{1, "one"}, {2, "two"}, {3, "three"}};
auto e = m.extract(2);
e.key() = 4;
m.insert(std::move(e));
// m == { { 1, "one" }, { 3, "three" }, { 4, "two" } }
```

### Parallel algorithms
Many of the STL algorithms, such as the `copy`, `find` and `sort` methods, started to support the *parallel execution policies*: `seq`, `par` and `par_unseq` which translate to "sequentially", "parallel" and "parallel unsequenced".

```c++
std::vector<int> longVector;
// Find element using parallel execution policy
auto result1 = std::find(std::execution::par, std::begin(longVector), std::end(longVector), 2);
// Sort elements using sequential execution policy
auto result2 = std::sort(std::execution::seq, std::begin(longVector), std::end(longVector));
```

## Acknowledgements
* [cppreference](http://en.cppreference.com/w/cpp) - especially useful for finding examples and documentation of new library features.
* [C++ Rvalue References Explained](http://thbecker.net/articles/rvalue_references/section_01.html) - a great introduction I used to understand rvalue references, perfect forwarding, and move semantics.
* [clang](http://clang.llvm.org/cxx_status.html) and [gcc](https://gcc.gnu.org/projects/cxx-status.html)'s standards support pages. Also included here are the proposals for language/library features that I used to help find a description of, what it's meant to fix, and some examples.
* [Compiler explorer](https://godbolt.org/)
* [Scott Meyers' Effective Modern C++](https://www.amazon.com/Effective-Modern-Specific-Ways-Improve/dp/1491903996) - highly recommended book!
* [Jason Turner's C++ Weekly](https://www.youtube.com/channel/UCxHAlbZQNFU2LgEtiqd2Maw) - nice collection of C++-related videos.
* [What can I do with a moved-from object?](http://stackoverflow.com/questions/7027523/what-can-i-do-with-a-moved-from-object)
* [What are some uses of decltype(auto)?](http://stackoverflow.com/questions/24109737/what-are-some-uses-of-decltypeauto)
* And many more SO posts I'm forgetting...

## Author
Anthony Calandra

## Content Contributors
See: https://github.com/AnthonyCalandra/modern-cpp-features/graphs/contributors

## License
MIT
