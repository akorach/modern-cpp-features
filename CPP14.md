# C++14

## Overview
Many of these descriptions and examples come from various resources (see [Acknowledgements](#acknowledgements) section), summarized in my own words.

C++14 includes the following new language features:
- [binary literals](#binary-literals)
- [generic lambda expressions](#generic-lambda-expressions)
- [lambda capture initializers](#lambda-capture-initializers)
- [return type deduction](#return-type-deduction)
- [decltype(auto)](#decltypeauto)
- [relaxing constraints on constexpr functions](#relaxing-constraints-on-constexpr-functions)
- [variable templates](#variable-templates)
- [\[\[deprecated\]\] attribute](#deprecated-attribute)

C++14 includes the following new library features:
- [user-defined literals for standard library types](#user-defined-literals-for-standard-library-types)
- [compile-time integer sequences](#compile-time-integer-sequences)
- [std::make_unique](#stdmake_unique)

## C++14 Language Features

### Binary literals
Binary literals provide a convenient way to represent a base-2 number.
It is possible to separate digits with `'`.
```c++
0b110 // == 6
0b1111'1111 // == 255
```

### Digit separators
From Wikipedia: In C++14, the single-quote character may be used arbitrarily as a digit separator in numeric literals, both integer literals and floating point literals.
```c++
auto integer_literal = 1'000'000;
auto floating_point_literal = 0.000'015'3;
auto binary_literal = 0b0100'1100'0110;
auto silly_example = 1'0'0'000'00;
```

### Generic lambda expressions
C++14 now allows the `auto` type-specifier in the parameter list, enabling polymorphic lambdas.
```c++
auto identity = [](auto x) { return x; };
int three = identity(3); // == 3
std::string foo = identity("foo"); // == "foo"
```

### Lambda capture initializers
This allows creating lambda captures initialized with arbitrary expressions. The name given to the captured value does not need to be related to any variables in the enclosing scopes and introduces a new name inside the lambda body. The initializing expression is evaluated when the lambda is _created_ (not when it is _invoked_).
```c++
int factory(int i) { return i * 10; }
auto f = [x = factory(2)] { return x; }; // returns 20

auto generator = [x = 0] () mutable {
  // this would not compile without 'mutable' as we are modifying x on each call
  return x++;
};
auto a = generator(); // == 0
auto b = generator(); // == 1
auto c = generator(); // == 2
```
Because it is now possible to _move_ (or _forward_) values into a lambda that could previously be only captured by copy or reference we can now capture move-only types in a lambda by value. Note that in the below example the `p` in the capture-list of `task2` on the left-hand-side of `=` is a new variable private to the lambda body and does not refer to the original `p`.
```c++
auto p = std::make_unique<int>(1);

auto task1 = [=] { *p = 5; }; // ERROR: std::unique_ptr cannot be copied
// vs.
auto task2 = [p = std::move(p)] { *p = 5; }; // OK: p is move-constructed into the closure object
// the original p is empty after task2 is created
```
Using this reference-captures can have different names than the referenced variable.
```c++
auto x = 1;
auto f = [&r = x, x = x * 10] {
  ++r;
  return r + x;
};
f(); // sets x to 2 and returns 12
```

### Return type deduction for normal functions
Using an `auto` return type in C++14, the compiler will attempt to deduce the type for you. With lambdas, you can now deduce its return type using `auto`, which makes returning a deduced reference or rvalue reference possible.
```c++
// Deduce return type as `int`.
auto f(int i) {
 return i;
}
```
```c++
template <typename T>
auto& f(T& t) {
  return t;
}

// Returns a reference to a deduced type.
auto g = [](auto& x) -> auto& { return f(x); };
int y = 123;
int& z = g(y); // reference to `y`
```
More on Wikipedia about recursion: https://en.wikipedia.org/wiki/C%2B%2B14#Function_return_type_deduction

### decltype(auto)
The `decltype(auto)` type-specifier also deduces a type like `auto` does. However, it deduces return types while keeping their references and cv-qualifiers, while `auto` will not. From Scott Meyers' _Effective Modern C++_:
```c++
template<typename Container, typename Index>
auto authAndAccess(Container& c, Index i) -> decltype(c[i]) // the return type depends on the container type
{
  authenticateUser();
  return c[i];
}
```
Is the same as:
```c++
template<typename Container, typename Index>
decltype(auto) authAndAccess(Container& c, Index i) // the return type depends on the container type
{
  authenticateUser();
  return c[i];
}
```
More generic:
```c++
template<typename Container, typename Index>
decltype(auto) authAndAccess(Container&& c, Index i) // also accepts universal references
{
  authenticateUser();
  return std::forward<Container>(c)[i];
}
```
See also: `decltype` (C++11).

### Relaxing constraints on constexpr functions
In C++11, `constexpr` function bodies could only contain a very limited set of syntaxes, including (but not limited to): `typedef`s, `using`s, and a single `return` statement. In C++14, the set of allowable syntaxes expands greatly to include the most common syntax such as `if` statements, multiple `return`s, loops, other `constexpr` functions or fundamental data types that have to be initialised with a `constexpr` expression. 
```c++
constexpr int factorial(int n) {
  if (n <= 1) {
    return 1;
  } else {
    return n * factorial(n - 1);
  }
}
factorial(5); // == 120
```

### Variable templates
C++14 allows variables to be templated:

```c++
template<class T>
constexpr T pi = T(3.1415926535897932385);
template<class T>
constexpr T e  = T(2.7182818284590452353);
```
This makes it possible to arbitrarily choose the precision you need for your constants and constant expressions that use them. In C++20 there's a library defining such constants based on this feature.

### [[deprecated]] attribute
C++14 introduces the `[[deprecated]]` attribute to indicate that a unit (function, class, etc) is discouraged and will yield a compilation warning. If a reason is provided, it will be included in the warning.
```c++
[[deprecated]]
void old_method();
[[deprecated("Use new_method instead")]]
void legacy_method();

//-----------------------

// Deprecate a function
[[deprecated]]
void foo();

// Deprecate a variable
[[deprecated]]
int x;

// Deprecate one declarator in a multi-declarator declaration
int y [[deprecated]], z;

// Deprecate a function parameter
int triple([[deprecated]] int x);

// Deprecate a class (or struct)
class [[deprecated]] my_class {
	public:
		// Deprecate a member
		[[deprecated]] int member;
};

// Deprecate a variable of a class defined in the same line
[[deprecated]] class C { } c;

// Deprecate an enum
enum [[deprecated]] animals {
	CAT, DOG, MOUSE
};

// Deprecate a typedef
[[deprecated]]
typedef int type;

// Deprecate a template specialization
template <typename T> class templ;

template <>
class [[deprecated]] templ<int> {};
```
From: [Joseph Mansfield](https://josephmansfield.uk/articles/marking-deprecated-c++14.html)

### Aggregate initialisation improvements
Adding _in-class member initializers_ does not make a class a non-aggregate. The following example is still an aggregate:
```c++
struct SimplePoint
{
  int x = 7;
  int y = 3;
};
```

## C++14 Library Features

### User-defined literals for standard library types
C++11 added user-defined literals, but didn’t use them in the standard library. Now some very useful ones work, including built-in literals for `chrono` and `basic_string`. These can be `constexpr` meaning they can be used at compile-time. Some uses for these literals include compile-time integer parsing, binary literals, and imaginary number literals.
```c++
using namespace std::chrono_literals;
using namespace std::string_literals;

auto string = "hello there"s;   // type std::string
auto minute = 60s;              // type std::chrono::duration = 60 seconds
auto day    = 24h;              // type std::chrono::duration = 24 hours
```
Note 's' means “string” when used on a string literal, and “seconds” when used on an integer literal, without ambiguity.

### Compile-time integer sequences
The class template `std::integer_sequence` represents a compile-time sequence of integers. There are a few helpers built on top:
* `std::make_integer_sequence<T, N>` - creates a sequence of `0, ..., N - 1` with type `T`.
* `std::index_sequence_for<T...>` - converts a template parameter pack into an integer sequence.

Convert an array into a tuple:
```c++
template<typename Array, std::size_t... I>
decltype(auto) a2t_impl(const Array& a, std::integer_sequence<std::size_t, I...>) {
  return std::make_tuple(a[I]...);
}

template<typename T, std::size_t N, typename Indices = std::make_index_sequence<N>>
decltype(auto) a2t(const std::array<T, N>& a) {
  return a2t_impl(a, Indices{});
}
```
Full example: https://en.cppreference.com/w/cpp/utility/integer_sequence

### std::make_unique
`std::make_unique` is the recommended way to create instances of `std::unique_ptr`s due to the following reasons:
* Avoid having to use the `new` operator.
* Prevents code repetition when specifying the underlying type the pointer shall hold.
* Most importantly, it provides exception-safety. Suppose we were calling a function `foo` like so:
```c++
foo(std::unique_ptr<T>(new T()), function_that_throws());
```
The compiler is given a lot of flexibility in terms of how it handles this call. It could create a new T, then call function_that_throws(), then create the std::unique_ptr that manages the dynamically allocated T. If function_that_throws() throws an exception, then the T that was allocated will not be deallocated, because the smart pointer to do the deallocation hasn’t been created yet. This leads to T being leaked.
```c++
foo(std::make_unique<T>(), function_that_throws());
```
std::make_unique() doesn’t suffer from this problem because the creation of the object T and the creation of the std::unique_ptr happen inside the std::make_unique() function, where there’s no ambiguity about order of execution.
After: https://www.learncpp.com/cpp-tutorial/15-5-stdunique_ptr/

### std::shared_timed_mutex
`std::mutex` - normal mutex; can be `lock`ed (blocks) or `try_lock`ed (returns if the lock isn't available).
`std::timed_mutex` - can be additionally `try_lock_for`ed (returns after a duration) and `try_lock_until`ed (returns when a specific time point is reached.
`std::shared_mutex` (C++17) - provides both exclusive and shared locking with the addition of `lock_shared` and `try_lock_shared`.
If one thread has acquired the exclusive lock (through `lock`, `try_lock`), no other threads can acquire the lock (including the shared).
If one thread has acquired the shared lock (through `lock_shared`, `try_lock_shared`), no other thread can acquire the *exclusive* lock, but can acquire the *shared* lock.
`std::shared_timed_mutex` - allows for timed lock acquisition trials for locks of both types.

Interesting trivia on why `std:shared_timed_mutex` was added in C++14 but `std::shared_mutex` only in C++17 https://stackoverflow.com/questions/40207171/why-shared-timed-mutex-is-defined-in-c14-but-shared-mutex-in-c17

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
