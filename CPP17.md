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


### Ignore unknown attributes
Clarifies that implementations should ignore any attribute namespaces which they don't support, as this used to be unspecified.
```c++
//compilers which don't support MyCompilerSpecificNamespace will ignore this attribute
[[MyCompilerSpecificNamespace::do_special_thing]]
void foo();
```
From: [BFilipek](https://www.bfilipek.com/2017/01/cpp17features.html#ignore-unknown-attributes)


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


### Stricter expression evaluation order
As a rule, given an expression such as f(a, b, c), the order in which the sub-expressions f, a, b, c (which are of arbitrary shapes) are evaluated is left unspecified by the standard. Here are a few examples:
```c++
// unspecified behaviour below!
f(i++, i);

v[i] = i++;

std::map<int, int> m;
m[0] = m.size(); // {{0, 0}} or {{0, 1}} ?
```
Here's a summary of changes:
* Postfix expressions are evaluated from left to right. This includes functions calls and member selection expressions.
* Assignment expressions are evaluated from right to left. This includes compound assignments.
* Operands to shift operators are evaluated from left to right.

### Structured bindings
A proposal for de-structuring initialization, that would allow writing `auto [ x, y, z ] = expr;` where the type of `expr` was a tuple-like object, whose elements would be bound to the variables `x`, `y`, and `z` (which this construct declares). _Tuple-like objects_ include `std::tuple`, `std::pair`, `std::array`, and aggregate structures.
Before:
```c++
int a = 0;
double b = 0.0;
long c = 0;
std::tie(a, b, c) = tuple; // a, b, c need to be declared first
```
After:
```c++
auto [ a, b, c ] = tuple;
```
More examples
```c++
using Coordinate = std::pair<int, int>;
Coordinate origin() {
  return Coordinate{0, 0};
}

const auto [ x, y ] = origin();

std::unordered_map<std::string, int> mapping {
  {"a", 1},
  {"b", 2},
  {"c", 3}
};

for (const auto& [key, value] : mapping) {
  // Do something with key and value
}
```


### constexpr if
The static-if for C++. This allows you to discard branches of an if statement at compile-time based on a constant expression condition.
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
This removes a lot of the necessity for tag dispatching and SFINAE.

SFINAE:
```c++
template <typename T, std::enable_if_t<std::is_arithmetic<T>{}>* = nullptr>
auto get_value(T t) {/*...*/}

template <typename T, std::enable_if_t<!std::is_arithmetic<T>{}>* = nullptr>
auto get_value(T t) {/*...*/}
```
Tag dispatching:
```c++
template <typename T>
auto get_value(T t, std::true_type) {/*...*/}

template <typename T>
auto get_value(T t, std::false_type) {/*...*/}

template <typename T>
auto get_value(T t) {
    return get_value(t, std::is_arithmetic<T>{});
}
```
if constexpr:
```c++
template <typename T>
auto get_value(T t) {
     if constexpr (std::is_arithmetic_v<T>) {
         //...
     }
     else {
         //...
     }
}
```
From: [BFilipek](https://www.bfilipek.com/2017/01/cpp17features.html#constexpr-if-statements)


### Init-statements for if and switch
New versions of the if and switch statements for C++: if (init; condition) and switch (init; condition).

Before:
```c++
{
    auto val = GetValue();
    if (condition(val))
        // on success
    else
        // on false...
}
```
After:
```c++
if (auto val = GetValue(); condition(val))
    // on success
else
    // on false...
```
From: [BFilipek](https://www.bfilipek.com/2017/01/cpp17features.html#init-statements-for-if-and-switch)


### Inline variables
Previously only methods/functions could be specified as inline, now you can do the same with variables, inside a header file.

A variable declared inline has the same semantics as a function declared inline: it can be defined, identically, in multiple translation units, must be defined in every translation unit in which it is used, and the behavior of the program is as if there is exactly one variable.

So, in practical terms the feature allows you to use the `inline` keyword to define an external linkage `const` namespace scope variable, or any `static` class data member, in a header file, so that the multiple definitions that result when that header is included in multiple translation units are OK with the linker – it just chooses one of them.
```c++
struct MyClass
{
    static const int sValue;
};

inline int const MyClass::sValue = 777;
```
Or even:
```c++
struct MyClass
{
    inline static const int sValue = 777;
};
```
Before C++17 we had to resort to:
```c++
template<typename Dummy>
struct MyClass_
{
    static int const sNumber;
};

template<typename Dummy>
int const MyClass_<Dummy>::sNumber = 42;

using MyClass = MyClass_<void>;    // Allows you to write `MyClass::sNumber`.
```
From: [BFilipek](https://www.bfilipek.com/2017/01/cpp17features.html#inline-variables)


### Removing deprecated exception specifications from C++17
Dynamic exception specifications were deprecated in C++11. This paper formally proposes removing the feature from C++17, while retaining the (still) deprecated throw() specification strictly as an alias for noexcept(true).

From: [BFilipek](https://www.bfilipek.com/2017/01/cpp17features.html#removing-deprecated-exception-specifications-from-c17)


### Pack expansions in using-declarations
Allows you to inject names with using-declarations from all types in a parameter pack.

In order to expose operator() from all base classes in a variadic template, we used to have to resort to recursion:
```c++
template <typename T, typename... Ts>
struct Overloader : T, Overloader<Ts...> {
    using T::operator();
    using Overloader<Ts...>::operator();
    // […]
};

template <typename T> struct Overloader<T> : T {
    using T::operator();
};
```
Now we can simply expand the parameter pack in the using-declaration:
```c++
template <typename... Ts>
struct Overloader : Ts... {
    using Ts::operator()...;
    // […]
};
```

## C++17 Library Features

### std::uncaught_exceptions()
The function returns the number of uncaught exception objects in the current thread.

This might be useful when implementing proper Scope Guards that works also during stack unwinding:

A type that wants to know whether its destructor is being run to unwind this object can query uncaught_exceptions
in its constructor and store the result, then query uncaught_exceptions again in its destructor; if the result is different,
then this destructor is being invoked as part of stack unwinding due to a new exception that was thrown later than the object’s construction.

From: [BFilipek](https://www.bfilipek.com/2017/01/cpp17features.html#stduncaughtexceptions)


### std::size(), std::empty() and std::data()
Generic functions to return these qualities for containers.


### std::bool_constant
`std::integral_constant` wraps a static constant of specified type. It is the base class for the C++ type traits.
The behavior of a program that adds specializations for `integral_constant` is undefined.

A helper alias template std::bool_constant is defined for the common case where T is bool:
```c++
template< class T, T v >
struct integral_constant;

template <bool B>
using bool_constant = integral_constant<bool, B>;
```


### std::shared_mutex
`std::mutex` - normal mutex; can be `lock`ed (blocks) or `try_lock`ed (returns if the lock isn't available).

`std::timed_mutex` - can be additionally `try_lock_for`ed (returns after a duration) and `try_lock_until`ed (returns when a specific time point is reached.

`std::shared_mutex` (C++17) - provides both exclusive and shared locking with the addition of `lock_shared` and `try_lock_shared`.

If one thread has acquired the exclusive lock (through `lock`, `try_lock`), no other threads can acquire the lock (including the shared).

If one thread has acquired the shared lock (through `lock_shared`, `try_lock_shared`), no other thread can acquire the *exclusive* lock, but can acquire the *shared* lock.
`std::shared_timed_mutex` - allows for timed lock acquisition trials for locks of both types.

Interesting trivia on why `std:shared_timed_mutex` was added in C++14 but `std::shared_mutex` only in C++17 https://stackoverflow.com/questions/40207171/why-shared-timed-mutex-is-defined-in-c14-but-shared-mutex-in-c17


### Type traits variable templates
Following the success of the \_t alias templates for type traits, a matching set of \_v variable templates have been proposed. The language support for variable templates arrived too late to confidently make this change for C++14, but experience since has shown that such variable templates are more succinct, can clean up the text in a similar way that the \_t aliases have been widely adopted through the standard, and the author's experience using them in his own implementation of the standard type traits library is that code is much simpler when written using such variable templates directly, rather than turning a value into a type, then performing template manipulations on the type, before turning the type back into a value.

The impact on the standard is that many places that reference `some_trait<T>::value` would instead use `some_trait_v<T>`. The saving is not quite as great as in the case of alias templates, as there is no irksome typename to remove. However, the consistecy of using \_t and \_v to refer to traits, and not using `::something` to extract meaning is compelling.


### Logical operator type traits
From the [proposal](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0013r1.html):

I propose three new type traits for performing logical operations with other traits, conjunction, disjunction, and negation, corresponding to the operators &&, ||, and ! respectively. These utilities are used extensively in libstdc++ and are valuable additions to any metaprogramming toolkit.
```c++
// logical conjunction:
template<class... B> struct conjunction;

// logical disjunction:
template<class... B> struct disjunction;

// logical negation:
template<class B> struct negation;
```
Motivation:

The proposed traits apply a logical operator to the result of one or more type traits, for example the specialization `conjunction<is_copy_constructible<T>, is_copy_assignable<T>>` has a `BaseCharacteristic` of `true_type` only if `T` is copy constructible and copy assignable, or in other words it has a `BaseCharacteristic` equivalent to `bool_constant<is_copy_constructible_v<T> && is_copy_assignable_v<T>>`.

The traits are especially useful when dealing with variadic templates where you want to apply the same predicate or transformation to every element in a parameter pack, for example `disjunction<is_nothrow_default_constructible<T>...>` can be used to determine whether at least one of the types in the pack `T` has a non-throwing default constructor.

To demonstrate how to use conjunction with is_same to make an 'all_same' trait, constraining a variadic function template so that all arguments have the same type can be done using conjunction<is_same<T, Ts>...>, for example:
```c++
// Accepts one or more arguments of the same type.
template<typename T, typename... Ts>
  enable_if_t< conjunction_v<is_same<T, Ts>...> >
  func(T, Ts...)
  { }
```
For the sake of clarity this function doesn't do perfect forwarding, but if the parameters were forwarding references the constraint would only be slightly more complicated: `conjunction_v<is_same<decay_t<T>, decay_t<Ts>>...>`.

Constraining all elements of a parameter pack to a specific type can be done similarly:

```c++
// Accepts zero or more arguments of type int.
template<typename... Ts>
  enable_if_t< conjunction_v<is_same<int, Ts>...> >
  func(Ts...)
  { }
  ```
Of course the three traits can be combined to form arbitrary predicates, the specialization `conjunction<disjunction<foo, bar>, negation<baz>>` corresponds to `(foo::value || bar::value) && !baz::value`.

### Parallel algorithms
Parallel versions/overloads of most of std algorithms, plus a few new algorithms such as `reduce`, `transform_reduce`, `for_each`.
The support comes in the form of *parallel execution policies*: `seq`, `par` and `par_unseq` which translate to "sequential", "parallel" and "parallel unsequenced".

```c++
std::vector<int> v = genLargeVector();

// standard sequential sort
std::sort(v.begin(), v.end());

// explicitly sequential sort
std::sort(std::seq, v.begin(), v.end());

// permitting parallel execution
std::sort(std::par, v.begin(), v.end());

// permitting vectorization as well
std::sort(std::par_unseq, v.begin(), v.end());
```
From: [BFilipek](https://www.bfilipek.com/2017/01/cpp17features.html#merged-the-parallelism-ts-aka-parallel-stl)


### std::clamp()
Clamps the value either to min or to max if necessary.


### Hardware interference size	
`std::hardware_destructive_interference_size`: Defines a minimum offset between two objects to avoid false sharing, and is guaranteed to be at least `alignof(std::max_align_t)`.

`std::hardware_constructive_interference_size`: Defines a maximum size of contiguous memory to promote true sharing, and is guaranteed to be at least `alignof(std::max_align_t)`.


### std::filesystem
The new `std::filesystem` library provides a standard way to manipulate files, directories, and paths in a filesystem. It largely mirrors the functionality of `boost::filesystem`, with some subtle differences.

```c++
namespace fs = std::filesystem;

fs::path pathToShow(/* ... */);
cout << "exists() = " << fs::exists(pathToShow) << "\n"
     << "root_name() = " << pathToShow.root_name() << "\n"
     << "root_path() = " << pathToShow.root_path() << "\n"
     << "relative_path() = " << pathToShow.relative_path() << "\n"
     << "parent_path() = " << pathToShow.parent_path() << "\n"
     << "filename() = " << pathToShow.filename() << "\n"
     << "stem() = " << pathToShow.stem() << "\n"
     << "extension() = " << pathToShow.extension() << "\n";
```
Example from: From: [BFilipek](https://www.bfilipek.com/2017/01/cpp17features.html#merged-file-system-ts)


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
More on string_view: 

https://www.bfilipek.com/2018/07/string-view-perf.html

https://www.bfilipek.com/2018/07/string-view-perf-followup.html


### std::any
A type-safe container for single values of any type.
```c++
std::any x {5};
x.has_value() // == true
std::any_cast<int>(x) // == 5
std::any_cast<int&>(x) = 10;
std::any_cast<int>(x) // == 10
```
Possible use cases:
* In Libraries - when a library type has to hold or pass anything without knowing the set of available types.
* Parsing files - if you really cannot specify what are the supported types
* Message passing
* Bindings with a scripting language
* Implementing an interpreter for a scripting language
* User Interface - controls might hold anything
* Entities in an editor

From: [BFilipek](https://www.bfilipek.com/2018/06/any.html)


### std::void_t
Utility metafunction that maps a sequence of any types to the type `void`. Form:
```c++
#include <type_traits>

template< class... >
using void_t = void;
```
This metafunction is used in template metaprogramming to detect ill-formed types in SFINAE context:
```c++
// primary template handles types that have no nested ::type member:
template< class, class = std::void_t<> >
struct has_type_member : std::false_type { };
 
// specialization recognizes types that do have a nested ::type member:
template< class T >
struct has_type_member<T, std::void_t<typename T::type>> : std::true_type { };
```
It can also be used to detect validity of an expression:
```c++
// primary template handles types that do not support pre-increment:
template< class, class = std::void_t<> >
struct has_pre_increment_member : std::false_type { };
// specialization recognizes types that do support pre-increment:
template< class T >
struct has_pre_increment_member<T,
           std::void_t<decltype( ++std::declval<T&>() )>
       > : std::true_type { };
```
From: [cppreference](https://en.cppreference.com/w/cpp/types/void_t)


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
