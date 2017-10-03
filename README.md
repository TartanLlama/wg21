Monadic operations for `std::optional`
======================================

Abstract
--------

`std::optional` will be a very important vocabulary type in C++17 and up. Some uses of it can be very verbose and would benefit from operations which allow functional composition. I propose adding `map` and `bind` member functions to `std::optional` to support this monadic style of programming.

Motivation
----------

`std::optional` aims to be a "vocabulary type", i.e. the canonical type to represent some programming concept. As such, `std::optional` will become widely used to represent an object which may or may not contain a value. Unfortunately, chaining together many computations which may or may not produce a value can be verbose, as empty-checking code will be mixed in with the actual programming logic. Take the following example:

```
float get_foo(int i);
std::string get_bar(float f);
std::function<int(std::string)> get_func(float f); //std::function use for exposition

int no_optional(int i) {
    auto f = get_foo(i);
    auto func = get_func(f);
    auto s = get_bar(f);
    return func(s);
}
```
This code is pretty straightforward and concise. But what happens if all of these functions could reasonably not produce a result?

```
std::optional<float>
maybe_get_foo(int i);
std::optional<std::string>
maybe_get_bar(float f);
std::optional<std::function<int(std::string)>>
maybe_get_func(float f);

std::optional<int> with_optional_icky(int i) {
    auto f = maybe_get_foo(i);
    if (!f) return std::nullopt;

    auto func = maybe_get_func(*f);
    if (!func) return std::nullopt;

    auto s = maybe_get_bar(*f);
    if (!s) return std::nullopt;
    
    return func.value()(*s);
}
```

Our code now has a lot of boilerplate to deal with the case where a step fails. Not only does this increase the noise and cognitive load of the function, but if we forget to put in a check, then suddenly we're throwing an exception if we do a bad optional access.

Another possibility would be to call `.value()` on the optionals and let the exception be thrown and caught like so:

```
std::optional<int> with_optional_exceptions(int i) {
   try {
      auto f = maybe_get_foo(i);
      auto func = maybe_get_func(f.value());
      auto s = maybe_get_bar(f.value());
      return func.value()(s.value());
   }
   catch (std::bad_optional_access& e) {
      return std::nullopt;
   }
}
```

If some of those function calls can reasonably not produce a value, then this code is using exceptions for control flow. This is commonly seen as bad practice, and has an item warning against it in the [C++ Core Guidelines](https://github.com/isocpp/CppCoreGuidelines/blob/master/CppCoreGuidelines.md#e3-use-exceptions-for-error-handling-only).

Proposed solution
-----------------

This paper proposes adding additional member functions to `std::optional` in order to push the handling of empty states off to the side. The proposed additions are `map` and `bind`. Using these new functions, the code above becomes this:

```
std::optional<int> with_optional_good(int i) {
    auto f = maybe_get_foo(i);
    auto func = f.bind(maybe_get_func);
    return f.bind(maybe_get_bar)
            .map(func);
}
```

We've successfully got rid of all the manual checking; all that we need now is an understanding of what `map` and `bind` do and how to use them.

#### `map`

`map` has two overloads (in pseudocode for exposition):

```
template <class T>
class optional {
    template <class Return>
    std::optional<Return> map (function<Return(T)> func);

    template <class Return>
    std::optional<Return> map (std::optional<function<Return(T)>> func);
};
```

The first takes any callable object (like a function). If the `optional` does not have a value stored, then an empty optional is returned. Otherwise, the given function is called with the stored value as an argument, and the return value is returned inside an `optional`.

The second overload takes an optional callable object. If that optional does not have a value stored, then an empty optional is returned. Otherwise, the value is extracted, and used to carry out the same steps as the first overload.

The function object *may* return `void`, in which case the returned type will be `std::optional<std::monostate>`.

If you come from a functional programming or category theory background, you may recognise the first of these as a functor map, and the second as an applicative functor map.

#### `bind`

`bind` has two overloads which look roughly like this:

```
template <class T>
class optional {
    template <class Return>
    std::optional<Return> bind (function<std::optional<Return>(T)> func);

    template <class Return>
    std::optional<Return> bind (function<std::optional<Return>(T)> func, function<void()> error);
};
```

The first takes any callable object which returns a `std::optional`. If the `optional` does not have a value stored, then an empty optional is returned. Otherwise, the given function is called with the stored value as an argument, and the return value is returned.

The second overload does the same, but if the execution of `func` returns an empty optional, the `error` function is called before returning. This allows the user to do some logging or throw an exception if they like.

Again, those from an FP background will recognise this as a monadic bind.

#### Chaining

With these two functions, doing a lot of chaining of functions which could fail becomes very clean:

```
std::optional<int> foo() {
    return
      a().bind(b)
         .bind(c)
         .bind(d)
         .bind(e);
}
```

We can also do chaining where failures can result in exceptions being thrown:

```
auto err = [](auto msg){ throw std::runtime_error{msg}; };
return a().bind(b, err("b failed"))
          .bind(c, err("c failed"))
          .bind(d, err("d failed"))
          .bind(e, err("e failed"));
```

This code is equivalent to:

```
auto ra = a();
if (!a) return std::nullopt;

auto rb = b(*a);


How other languages handle this
-------------------------------

`std::optional` is known as `Maybe` in Haskell and it provides much the same functionality. `map` is split between `Functor` (`fmap`) and `Applicative` (`<*>`), and `bind` is in `Monad` and named `>>=`.

Rust has an `Option` class where the functor part of `map` is called `map`, and `bind` is called `and_then`. It also provides many additional member functions like `or_else` to return the option if it has a value and call a function otherwise.

Considerations
--------------

### More functions

Rust's [`Option`](https://doc.rust-lang.org/std/option/enum.Option.html) class provides a lot more than `map` and `bind` (`and_then`). If the idea to add `map` and `bind` is received favourably, then we can think about what other additions we may wan to make.

### `map` only

It would be possible to merge the `map` and `bind` functions into one single function which detects if the callable argument will return a `std::optional<T>`, and, if so, return a `std:optional<T>` rather than a `std::optional<std::optional<T>>`. However, I think that this conflates the uses of two functions: `map` is for functions which always produce a result, `bind` is for functions which may or may not. Furthermore, the second argument to `bind` does not make sense for `map`.

### Alternative names

`map` may confuse users who are more familiar with its use as a data structure, or consider the common array map from other languages to be different from this application. Some other possible names are `then`, `when_value`, `fmap`, `transform`.

`bind` may confuse those more familiar with `std::bind`. An alternative names are `compose`, `chain`, and `and_then`.


Pitfalls
--------

Users may want to write code like this:

```
std::optional<int> foo(int i) {
    return
      a().bind(b)
         .bind(get_func(i));
}
```

The problem with this is `get_func` will be called regardless of whether `b` returns an empty `std::optional` or not. If it has side effects, then this may not be what the user wants.

One possible solution to this would be to add an additional function, `bind_with` which will take a callable which provides what you want to `bind` to:

```
std::optional<int> foo(int i) {
    return
      a().bind(b)
         .bind_with([i](){return get_func(i)});
}
```

This kind of usage might benefit from the [lift operator](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3617.htm) or [abbreviated lambdas](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0573r0.html).


Other solutions
----------------------

There is a proposal for adding a [general monadic interface](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0650r0.pdf) to C++. Unfortunately doing the kind of composition described above would be very verbose with the current proposal without some kind of Haskell-style `do` notation. If my understanding is correct, then the code for my first solution above would look like this:

```
std::optional<int> with_optional_good(int i) {
    auto f = maybe_get_foo(i);
    auto func = monad::bind(f, maybe_get_func);
    return applicative::ap(
             monad::bind(maybe_get_bar), func);
}
```

This is not so bad, but when you start to do some longer chains, the non-member functions start to hurt:

```
std::optional<int> foo() {
    return
      monad::bind(e,    
        monad::bind(d,
          monad::bind(c,
            monad::bind(a(), b));
}
```

My proposal is not necessarily an alternative to this proposal; compatibility between the two could be ensured and the generic proposal could use my proposal as part of its implementation. This would allow users to use both the generic syntax for flexibility and extensibility, and the member-function syntax for brevity and clarity.

If `do` notation or [unified call syntax](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0301r1.html) is accepted, then this proposal may not be necessary.

Interaction with other proposals
--------------------------------

There is a proposal for [`std::expected`](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0323r2.pdf) which would benefit from many of these same ideas. If the idea to add monadic interfaces to standard library classes on a case-by-case basis is chosen rather than a unified non-member function interface, then compatibility between this proposal and the `std::expected` one should be maintained.

Mapping functions which return `void` is supported, but is a pain to implement since `void` is not a regular type. If the [Regular Void](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0146r1.html) proposal was accepted, implementation would be simpler and the results of the operation would conform to users expectations better (i.e. return `std::optional<void>` instead of `std::optional<std::monostate>`.

Implementation experience
-------------------------

This proposal has been implemented [here](https://github.com/TartanLlama/monadic-optional).

---------------------------------------
---------------------------------------

Proposed Wording
----------------

#### New synopsis entry: `[optional.monadic]`, monadic operations

```
template <class F> constexpr *see below* bind(F&& f) &;
template <class F> constexpr *see below* bind(F&& f) &&;
template <class F> constexpr *see below* bind(F&& f) const&;
template <class F> constexpr *see below* bind(F&& f) const&&;
template <class F, class E> constexpr *see below* bind(F&& f, E&& e) &;
template <class F, class E> constexpr *see below* bind(F&& f, E&& e) &&;
template <class F, class E> constexpr *see below* bind(F&& f, E&& e) const&;
template <class F, class E> constexpr *see below* bind(F&& f, E&& e) const&&;
template <class F> constexpr *see below* map(F&& f) &;
template <class F> constexpr *see below* map(F&& f) &&;
template <class F> constexpr *see below* map(F&& f) const&;
template <class F> constexpr *see below* map(F&& f) const&&;
```

#### New section: Monadic operations `[optional.monadic`]

```
template <class F> constexpr *see below* bind(F&& f) &;
template <class F> constexpr *see below* bind(F&& f) const&;
```

*Requires*: `std::invoke(std::forward<F>(f), value())` returns a `std::optional<U>` for some `U`.

*Returns*: Let `U` be the result of `std::invoke(std::forward<F>(f), value())`. Returns a `std::optional<U>`. The return value is empty if `*this` is empty, otherwise the return value of `std::invoke(std::forward<F>(f), value())` is returned.

---------------------------------------

```
template <class F> constexpr *see below* bind(F&& f) &&;
template <class F> constexpr *see below* bind(F&& f) const&&;
```

*Requires*: `std::invoke(std::forward<F>(f), std::move(value()))` returns a `std::optional<U>` for some `U`.

*Returns*: Let `U` be the result of `std::invoke(std::forward<F>(f), std::move(value()))`. Returns a `std::optional<U>`. The return value is empty if `*this` is empty, otherwise the return value of `std::invoke(std::forward<F>(f), std::move(value()))` is returned.

---------------------------------------

```
template <class F, class E> *see below* bind(F&& f, E&& e) &;
template <class F, class E> *see below* bind(F&& f, E&& e) const&;
```

*Requires*: `std::invoke(std::forward<F>(f), value())` returns a `std::optional<U>` for some `U`.

*Returns*: Let `U` be the result of `std::invoke(std::forward<F>(f), value())`. Returns a `std::optional<U>`. The return value is empty if `*this` is empty, otherwise the return value of `std::invoke(std::forward<F>(f), value())` is returned.

*Effects*: If `*this` is not empty and `std::invoke(std::forward<F>(f), value())` returns an empty `std::optional`, then `e` is called with no arguments.

---------------------------------------

```
template <class F, class E> *see below* bind(F&& f, E&& e) &&;
template <class F, class E> *see below* bind(F&& f, E&& e) const&&;
```
*Requires*: `std::invoke(std::forward<F>(f), std::move(value()))` returns a `std::optional<U>` for some `U`.

*Returns*: Let `U` be the result of `std::invoke(std::forward<F>(f), std::move(value()))`. Returns a `std::optional<U>`. The return value is empty if `*this` is empty, otherwise the return value of `std::invoke(std::forward<F>(f), std::move(value()))` is returned.

*Effects*: If `*this` is not empty and `std::invoke(std::forward<F>(f), std::move(value()))` returns an empty `std::optional`, then `e` is called with no arguments.

---------------------------------------

```
template <class F> constexpr *see below* map(F&& f) &;
template <class F> constexpr *see below* map(F&& f) const&;
```

*Returns*: Let `U` be the result of `std::invoke(std::forward<F>(f), value())`. Returns a `std::optional<U>`. The return value is empty if `*this` is empty, otherwise an `optional<U>` is constructed from the return value of `std::invoke(std::forward<F>(f), value())` and is returned.

```
template <class F> constexpr *see below* map(F&& f) &&;
template <class F> constexpr *see below* map(F&& f) const&&;
```

*Returns*: Let `U` be the result of `std::invoke(std::forward<F>(f), std::move(value()))`. Returns a `std::optional<U>`. The return value is empty if `*this` is empty, otherwise an `optional<U>` is constructed from the return value of `std::invoke(std::forward<F>(f), std::move(value()))` and is returned.

---------------------------------------
---------------------------------------

Acknowledgements
---------------

Thank you to [Kenneth Benzie](https://twitter.com/KmBenzie) for the idea to add an error-handling argument to `bind`. Thanks to [Vittorio Romeo](https://twitter.com/supahvee1234) and [Jonathan Müller](https://twitter.com/foonathan) for initial review and suggestions.

License
-------
Distributed under the [Boost Software License, Version 1.0](http://www.boost.org/LICENSE_1_0.txt).