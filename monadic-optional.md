<pre class='metadata'>
Title: Monadic operations for std::optional
Status: P
ED: wg21.tartanllama.xyz/monadic-optional
Shortname: p0798
Level: 6
Editor: Sy Brand, sybrand@microsoft.com
Abstract: std::optional is a very important vocabulary type in C++17 and up. Some uses of it can be very verbose and would benefit from operations which allow functional composition. I propose adding map, and_then, and or_else member functions to std::optional to support this monadic style of programming.
Group: wg21
Audience: LWG
Markup Shorthands: markdown yes
Default Highlight: C++
Line Numbers: yes
</pre>

# Changes from r5

- Respecify wording in terms of "Equivalent to"
- Add constraints for all invocables
- Use `decay_t` for `transform`
- Remove function bodies from synopsis
- Make rvalue `or_else` overload require *Cpp17CopyConstructible*


# Changes from r4

- Remove non-`const` lvalue and `const` rvalue overloads for `or_else`
- Remove special-casing of `void` from `or_else`:
    Mandates: `invoke_result_t<F>` is <span style="background: lightcoral">`void` or</span> convertible to `optional`.

    Effects: If `*this` has a value, returns `*this`. <span style="background: lightcoral">Otherwise, if `invoke_result_t<F>` is `void`, calls `std::forward<F>(f)` and returns `nullopt`.</span> Otherwise, returns `std::forward<F>(f)()`.

    Effects: If `*this` has a value, returns `std::move(*this)`. <span style="background: lightcoral">Otherwise, if `invoke_result_t<F>` is `void`, calls `std::forward<F>(f)` and returns `nullopt`.</span> Otherwise, returns `std::forward<F>(f)()`.
- Use `remove_cv_ref_t` instead of `decay_t`
- Add feature test macro
- Add function bodies to synopsis
- Formatting changes

# Changes from r3
- Wording improvements

# Changes from r2
- Rename `map` to `transform`
- Remove "Alternative Names" section
- Address feedback on disallowing function pointers
- Remove capability for mapping void-returning functions
- Tighten wording
- Discuss SFINAE-friendliness

# Changes from r1
- Add list of programming languages to proposed solution section
- Change map example code to not take address of stdlib function
- Update wording to use `std::monostate` on `void`-returning callables

# Changes from r0

- More notes on P0650
- Discussion about mapping of functions returning `void`

# Motivation

`std::optional` aims to be a "vocabulary type", i.e. the canonical type to represent some programming concept. As such, `std::optional` will become widely used to represent an object which may or may not contain a value. Unfortunately, chaining together many computations which may or may not produce a value can be verbose, as empty-checking code will be mixed in with the actual programming logic. As an example, the following code automatically extracts cats from images and makes them more cute:

```
image get_cute_cat (const image& img) {
    return add_rainbow(
             make_smaller(
               make_eyes_sparkle(
                 add_bow_tie(
                   crop_to_cat(img))));
}
```

But there's a problem. What if there's not a cat in the picture? What if there's no good place to add a bow tie? What if it has its back turned and we can't make its eyes sparkle? Some of these operations could fail.

One option would be to throw exceptions on failure. However, there are many code bases which do not use exceptions for a variety of reasons. There's also the possibility that we're going to get *lots* of pictures without cats in them, in which case we'd be using exceptions for control flow. This is commonly seen as bad practice, and has an item warning against it in the [C++ Core Guidelines](https://github.com/isocpp/CppCoreGuidelines/blob/master/CppCoreGuidelines.md#e3-use-exceptions-for-error-handling-only).

Another option would be to make those operations which could fail return a `std::optional`:

```
std::optional<image> get_cute_cat (const image& img) {
    auto cropped = crop_to_cat(img);
    if (!cropped) {
      return std::nullopt;
    }

    auto with_tie = add_bow_tie(*cropped);
    if (!with_tie) {
      return std::nullopt;
    }

    auto with_sparkles = make_eyes_sparkle(*with_tie);
    if (!with_sparkles) {
      return std::nullopt;
    }

    return add_rainbow(make_smaller(*with_sparkles));
}
```

Our code now has a lot of boilerplate to deal with the case where a step fails. Not only does this increase the noise and cognitive load of the function, but if we forget to put in a check, then suddenly we're down the hole of undefined behaviour if we `*empty_optional`.

Another possibility would be to call `.value()` on the optionals and let the exception be thrown and caught like so:

```
std::optional<image> get_cute_cat (const image& img) {
    try {
        auto cropped = crop_to_cat(img);
        auto with_tie = add_bow_tie(cropped.value());
        auto with_sparkles = make_eyes_sparkle(with_tie.value());
        return add_rainbow(make_smaller(with_sparkles.value()));
    catch (std::bad_optional_access& e) {
        return std::nullopt;
    }
}
```

Again, this is using exceptions for control flow. There must be a better way.

# Proposed solution

This paper proposes adding additional member functions to `std::optional` in order to push the handling of empty states off to the side. The proposed additions are `map`, `and_then` and `or_else`. Using these new functions, the code above becomes this:

```
std::optional<image> get_cute_cat (const image& img) {
    return crop_to_cat(img)
           .and_then(add_bow_tie)
           .and_then(make_eyes_sparkle)
           .map(make_smaller)
           .map(add_rainbow);
}
```

We’ve successfully got rid of all the manual checking. We’ve even improved on the clarity of the non-`optional` example, which needed to either be read inside-out or split into multiple declarations.

This is common in other programming languages. Here is a list of programming languages which have a `optional`-like type with a monadic interface or some similar syntactic sugar:

- Java: `Optional`
- Swift: `Optional`
- Haskell: `Maybe`
- Rust: `Option`
- OCaml: `option`
- Scala: `Option`
- Agda: `Maybe`
- Idris: `Maybe`
- Kotlin: `T?`
- StandardML: `option`
- C#: `Nullable`

Here is a list of programming languages which have a `optional`-like type without a monadic interface or syntactic sugar:

- C++
- I couldn’t find any others

All that we need now is an understanding of what `transform` and `and_then` do and how to use them.

## `transform`

`transform` is used to apply a function to change the value (and possibly the type) stored in an optional. It applies a function to the value stored in the optional and returns the result wrapped in an optional. If there is no stored value, then it returns an empty optional.

For example, if you have a `std::optional<std::string>` and you want to get a `std::optional<std::size_t>` giving the size of the string if one is available, you could write this:

```
auto s = opt_string.transform([](auto&& s) { return s.size(); });
```

which is somewhat equivalent to:

```
if (opt_string) {
    std::size_t s = opt_string->size();
}
```

`transform` has one overload (expositional):

```
template <class T>
class optional {
    template <class Return>
    std::optional<Return> transform (function<Return(T)> func);
};
```

It takes any invocable. If the optional does not have a value stored, then an empty optional is returned. Otherwise, the given function is called with the stored value as an argument, and the return value is returned inside an optional.

If you come from a functional programming or category theory background, you may recognise this as a functor map.

## `and_then`

`and_then` is used to compose functions which return `std::optional`.

For example, say you have `std::optional<std::string>` and a function like `std::stoi` which returns a `std::optional<int>` instead of throwing on failure. Rather than manually checking the optional string before calling, you could do this:

```
std::optional<int> i = opt_string.and_then(stoi);
```

Which is roughly equivalent to:

```
if (opt_string) {
   std::optional<int> i = stoi(*opt_string);
}
```

`and_then` has one overload which looks like this (again, expositional):

```
template <class T>
class optional {
    template <class Return>
    std::optional<Return> and_then (function<std::optional<Return>(T)> func);
};
```

It takes any callable object which returns a `std::optional`. If the `optional` does not have a value stored, then an empty `optional` is returned. Otherwise, the given function is called with the stored value as an argument, and the return value is returned.

Again, those from an FP background will recognise this as a monadic bind.

## `or_else`

`or_else` returns the optional if it has a value, otherwise it calls a given function. This allows you do things like logging or throwing exceptions in monadic contexts:

```
get_opt().or_else([]{std::cout << "get_opt failed";});
get_opt().or_else([]{throw std::runtime_error("get_opt_failed")});
```

Users can easily abstract these patterns if they are common:

```
void opt_log(std::string_view msg) {
     return [=] { std::cout << msg; };
}

void opt_throw(std::string_view msg) {
     return [=] { throw std::runtime_error(msg); };
}

get_opt().or_else(opt_log("get_opt failed"));
get_opt().or_else(opt_throw("get_opt failed"));

```

It has one overload (expositional):

```
template <class T>
class optional {
    template <class Return>
    std::optional<T> or_else (function<Return()> func);
};
```

`func` will be called if `*this` is empty. `Return` will either be convertible to `std::optional<T>`, or `void`. In the former case, the return of `func` will be returned from `or_else`; in the second case `std::nullopt` will be returned.

## Chaining

With these two functions, doing a lot of chaining of functions which could fail becomes very clean:

```
std::optional<int> foo() {
    return
      a().and_then(b)
         .and_then(c)
         .and_then(d)
         .and_then(e);
}
```

Taking the example of `stoi` from earlier, we could carry out mathematical operations on the result by just adding `map` calls:

```
std::optional<int> i = opt_string
                       .and_then(stoi)
                       .transform([](auto i) { return i * 2; });
```

We can also intersperse our chain with error checking code:

```
std::optional<int> i = opt_string
                       .and_then(stoi)
                       .or_else(opt_throw("stoi failed"))
                       .transform([](auto i) { return i * 2; });
```

# How other languages handle this

`std::optional` is known as `Maybe` in Haskell and it provides much the same functionality. `transform` is in `Functor` and named `fmap`, and `and_then` is in `Monad` and named `>>=` (bind).

Rust has an `Option` class, and uses the same names as are proposed in this paper. It also provides many additional member functions like `or`, `and`, `map_or_else`.

# Considerations

## Disallowing function pointers

In San Diego, a straw poll on disallowing passing function pointers to the functions proposed was made with the following results:

```
|SF|F|N|A|SA|
|3 |6|1|0|1 | 
```

This poll was carried out with the understanding that some parts of Ranges (in particular `ranges::transform_view`) disallow function pointers. However, this is not the case. From p0896r2:

```
template<InputRange R, CopyConstructible F>
requires View<R> && is_object_v<F&&> Invocable<F&,iter_reference_t<iterator_t<R>>>
class transform_view;
```

I imagine the confusion comes from `is_object_v<F&&>`, but function pointers are object types. From n4460 (C++17):

> An object type is a (possibly cv-qualiﬁed) type that is not a function type, not a reference type, and not cv void.

As far as I am aware, if we disallowed passing function pointers here then this would be the only place in the standard library which does so. Alongside the issue of consistency, ease-of-use is damaged. For example:

```
int this_will_never_be_overloaded(double);
my_optional_double.transform(this_will_never_be_overloaded); //not allowed
//have to write this instead
my_optional_double.transform([](double d){ return this_will_never_be_overloaded(d); });
//or use some sort of macro
my_optional_double.transform(LIFT(this_will_never_be_overloaded));
```

For library extensions which are supposed to aid ergonomics and simplify code, this goes in the opposite direction.

The benefit of disallowing function pointers is avoidance of build breakages when a previously non-overloaded function is overloaded. This is not a new issue: it applies to practically every single higher order function in the C++ standard library, including functions in `<algorithm>`, `<numeric>`, and ranges. There are papers in flight to address this issue in other ways (such as p1170), but in the meantime it remains a problem. While I see the desire to avoid this issue, I think the cost of consistency and ease-of-use far outweighs the benefits. As such, this paper still allows function pointers to be passed.

## Mapping functions returning `void`

There are three main options for handling `void`-returning functions for `transform`. The simplest is to disallow it (this is what `boost::optional` does). One is to return `std::optional<std::monostate>` (or some other unit type) instead of `std::optional<void>` (this is what `tl::optional` does). Another option is to add a `std::optional<void>` specialization. This functionality can be desirable since it allows chaining functions which are used purely for side effects.

```
get_optional()                // returns std::optional<T>
  .transform(print_value)     // returns std::optional<std::monostate>
  .transform(notify_success); // Is only called when get_optional returned a value
```

This proposal disallows mapping void-returning functions.

## More functions

Rust's [Option](https://doc.rust-lang.org/std/option/enum.Option.html) class provides a lot more than `map`, `and_then` and `or_else`. If the idea to add these is received favourably, then we can think about what other additions we may want to make.

## `transform` only

It would be possible to merge all of these into a single function which handles all use cases. However, I think this would make the function more difficult to teach and understand.

## Operator overloading

We could provide operator overloads with the same semantics as the functions. For example, `|` could mean `transform`, `>=` `and_then`, and `&` `or_else`. Rewriting the original example gives this:

```
// original
crop_to_cat(img)
  .and_then(add_bow_tie)
  .and_then(make_eyes_sparkle)
  .transform(make_smaller)
  .transform(add_rainbow);

// rewritten
crop_to_cat(img)
   >= add_bow_tie
   >= make_eyes_sparkle
    | make_smaller
    | add_rainbow;

```

Another option would be `>>` for `and_then`.

## Applicative Functors

`transform` could be overloaded to accept callables wrapped in `std::optionals`. This fits the *applicative functor* concept. It would look like this:

```
template <class Return>
std::optional<Return> map (std::optional<function<Return(T)>> func);
```

This would give functional programmers the set of operations which they may expect from a monadic-style interface. However, I couldn't think of many use-cases of this in C++. If some are found then we could add the extra overload.

## SFINAE-friendliness

`transform` and `and_then` are specified to return `auto`. This makes them not SFINAE-friendly. This is required because of the issue described in [p0826](https://wg21.link/p0826): if the callable which is passed in has an SFINAE-unfriendly call operator template, it could produce a hard error when instantiated in the `const` and non-`const` overloads of `transform` and `and_then` during overload resolution. For example, if `transform` was SFINAE-friendly, the following code would result in a hard error:

```
struct foo {
  void non_const() {}
};

std::optional<foo> f = foo{};
auto l = [](auto &&x) { x.non_const(); };
//error: passing 'const foo' as 'this' argument discards qualifiers
f.transform(l); 
```

If p0847 is accepted and this proposal is rebased on top of it, then this would no longer be an issue, and `transform` and `and_then` could be made SFINAE-friendly.

# Pitfalls

Users may want to write code like this:

```
std::optional<int> foo(int i) {
    return
      a().and_then(b)
         .and_then(get_func(i));
}
```

The problem with this is `get_func` will be called regardless of whether `b` returns an empty `std::optional` or not. If it has side effects, then this may not be what the user wants.

One possible solution to this would be to add an additional function, `bind_with` which will take a callable which provides what you want to bind to:

```
std::optional<int> foo(int i) {
    return
      a().and_then(b)
         .bind_with([i](){return get_func(i)});
}
```


Other solutions
----------------------

There is a proposal for adding a [general monadic interface](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0650r0.pdf) to C++. Unfortunately doing the kind of composition described above would be very verbose with the current proposal without some kind of Haskell-style `do` notation. The code for my first solution above would look like this:

```
std::optional<int> get_cute_cat(const image& img) {
    return
      functor::map(
        functor::map(
          monad::bind(
            monad::bind(crop_to_cat(img),
              add_bow_tie),
            make_eyes_sparkle),
         make_smaller),
      add_rainbow);
}
```

My proposal is not necessarily an alternative to this proposal; compatibility between the two could be ensured and the generic proposal could use my proposal as part of its implementation. This would allow users to use both the generic syntax for flexibility and extensibility, and the member-function syntax for brevity and clarity.

If `do` notation or [unified call syntax](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0301r1.html) is accepted, then my proposal may not be necessary, as use of the generic monadic functionality would allow the same or similarly concise syntax.

Another option would be to use a ranges-style interface for the general monadic interface:

```
std::optional<int> get_cute_cat(const image& img) {
    return crop_to_cat(img)
         | monad::bind(add_bow_tie)
         | monad::bind(make_eyes_sparkle)
         | functor::map(make_smaller)
         | functor::map(add_rainbow);
}
```

Interaction with other proposals
--------------------------------

[p0847](https://wg21.link/p0847) would ease implementation and solve the issue of SFINAE-friendliness for `transform` and `and_then`.

There is a proposal for [std::expected](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0323r2.pdf) which would benefit from many of these same ideas. If the idea to add monadic interfaces to standard library classes on a case-by-case basis is chosen rather than a unified non-member function interface, then compatibility between this proposal and the `std::expected` one should be maintained.

Mapping functions which return `void` can be supported, but is a pain to implement since `void` is not a regular type. If the [Regular Void](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0146r1.html) proposal was accepted, implementation would be simpler and the results of the operation would conform to users' expectations better.

Any proposals which make lambdas or overload sets easier to write and pass around will greatly improve this proposal. In particular, proposals for [lift operators](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3617.htm) and [abbreviated lambdas](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0573r0.html) would ensure that the clean style is preserved in the face of many anonymous functions and overloads.


# Implementation experience

This proposal has been implemented [here](https://github.com/TartanLlama/monadic-optional).

---------------------------------------

# Proposed Wording

## Add feature test macro to Table 36 in [support.limits.general]
<table>
	<tbody>
		<tr>
			<td><b>Macro name</b></td>
			<td><b>Value</b></td>
			<td><b>Header(s)</b></td>
  	</tr>
    <tr>
			<td> ... </td><td> ... 	</td><td> ...	</td>
    </tr>
    <tr>
					<td>_­_­cpp_­lib_­memory_­resource
					</td><td>201603L
					</td><td>&lt;memory_resource&gt;
				</td></tr><tr>
					<td style="background: palegreen;">__cpp_lib_monadic_optional
					</td><td style="background: palegreen;">201907L
					</td><td style="background: palegreen;">&lt;optional&gt;
				</td></tr><tr>
					<td>__cpp_lib_node_extract
					</td><td>201606L
					</td><td>&lt;map&gt; &lt;set&gt; &lt;unordered_map&gt; &lt;unordered_set&gt;
				</td></tr><tr>
					<td> ...
					</td><td> ...
					</td><td> ...
		</td></tr></tbody></table>

##  Add declarations for the monadic operations to the synopsis of class template optional in [optional.optional]

```
//...
template<class U> constexpr T value_or(U&&) const&;
template<class U> constexpr T value_or(U&&) &&;
// [optional.monadic], monadic operations
template <class F> constexpr auto and_then(F&& f) &;
template <class F> constexpr auto and_then(F&& f) &&;
template <class F> constexpr auto and_then(F&& f) const&;
template <class F> constexpr auto and_then(F&& f) const&&;
template <class F> constexpr auto transform(F&& f) &;
template <class F> constexpr auto transform(F&& f) &&;
template <class F> constexpr auto transform(F&& f) const&;
template <class F> constexpr auto transform(F&& f) const&&;
template <class F> constexpr optional or_else(F&& f) &&;
template <class F> constexpr optional or_else(F&& f) const&;
// [optional.mod], modifiers
void reset() noexcept;
//...
```

## Add new subclause "Monadic operations [optional.monadic]" between [optional.observe] and [optional.mod]

```
template <class F> constexpr auto and_then(F&& f) &;
template <class F> constexpr auto and_then(F&& f) const&;
```

>Let `U` be `invoke_result_t<F, decltype(value())>`.
>
>*Constraints*: `F` models `invocable<decltype(value())>`.
>
>*Mandates*: `remove_cvref_t<U>` is a specialization of `optional`.
>
>*Effects*: Equivalent to:
>
>```
>if (*this) {
>  return invoke(std::forward<F>(f), value()); 
>} 
>else {
>  return remove_cvref_t<U>();
>}
>```


```
template <class F> constexpr auto and_then(F&& f) &&;
template <class F> constexpr auto and_then(F&& f) const&&;
```

>Let `U` be `invoke_result_t<F, decltype(std::move(value()))>`.
>
>*Constraints*: `F` models `invocable<decltype(std::move(value()))>`.
>
>*Mandates*: `remove_cvref_t<U>` is a specialization of `optional`.
>
>*Effects*: Equivalent to:
>
>```
>if (*this) {
>  return invoke(std::forward<F>(f), std::move(value())); 
>} 
>else {
>  return remove_cvref_t<U>();
>}
>```

```
template <class F> constexpr auto transform(F&& f) &;
template <class F> constexpr auto transform(F&& f) const&;
```

>Let `U` be `invoke_result_t<F, decltype(value())>`.
>
>*Constraints*: `F` models `invocable<decltype(value())>`.
>
>*Effects*: Equivalent to:
>
>```
>if (*this) {
>  return optional<U>(in_place, invoke(std::forward<F>(f), value())); 
>} 
>else {
>  return optional<U>();
>}
>```

```
template <class F> constexpr auto transform(F&& f) &&;
template <class F> constexpr auto transform(F&& f) const&&;
```

>Let `U` be `invoke_result_t<F, decltype(std::move(value()))>`.
>
>*Constraints*: `F` models `invocable<decltype(std::move(value()))>`.
>
>*Effects*: Equivalent to:
>
>```
>if (*this) {
>  return optional<U>(in_place, invoke(std::forward<F>(f), std::move(value()))); 
>} 
>else {
>  return optional<U>();
>}
>```

```
template <class F> constexpr optional or_else(F&& f) const&;
```

>*Constraints*: `F` models `invocable<>`.
>
>*Expects*: `T` meets the *Cpp17CopyConstructible* requirements (Table 27).
>
>*Effects*: Equivalent to:
>
>```
>if (*this) {
>  return *this;
>}
>else {
>  return std::forward<F>(f)();
>}
>```

```
template <class F> constexpr optional or_else(F&& f) &&;
```

>*Constraints*: `F` models `invocable<>`.
>
>*Expects*: `T` meets the *Cpp17CopyConstructible* requirements (Table 27).
>
>*Effects*: Equivalent to:
>
>```
>if (*this) {
>  return std::move(*this);
>}
>else {
>  return std::forward<F>(f)();
>}
>```

---------------------------------------

Acknowledgements
---------------

Thank you to Michael Wong and Chris Di Bella for representing this paper to the committee. Thanks to Kenneth Benzie, Vittorio Romeo, Jonathan Müller, Adi Shavit, Nicol Bolas, Vicente Escribá, Barry Revzin, Tim Song, and especially Casey Carter for review and suggestions.