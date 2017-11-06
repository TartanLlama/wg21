<pre class='metadata'>
Title: Let's pretend std::initializer_list never happened
Status: D
ED: wg21.tartanllama.xyz/initializer_lst
Shortname: dXXXX
Level: 0
Editor: Christopher Di Bella, christopher@codeplay.com
Editor: Simon Brand, simon@codeplay.com
Group: wg21
Audience: LEWG
Markup Shorthands: markdown yes
Default Highlight: C++
Abstract: std::initializer_list does not support some key use-cases, and can often cause subtle semantic errors. We propose moving away from their use and replacing them with tagged variadic template constructors.
</pre>

# Motivation

`std::initializer_list` has three main problems:

- Unusable with move-only types
- Not deducible
- Small changes in initialization syntax can change semantics

## Move-only types

```
struct move_only {
    move_only() = default;
    move_only(const move_only&) = delete;
    move_only(move_only&&) = default;
};

void oh_no(std::initializer_list<move_only>a) {
    auto compiler_is_sad = std::move(*a.begin());
    //<source>:11:48: error: use of deleted function 'move_only::move_only(const move_only&)'
    //auto compiler_is_sad = std::move(*a.begin());
}

int main() {
    oh_no({move_only{}});
}
```

## Not deducible

```
template <class T>
void oh_no (T&& t) {

}

oh_no({0,1,2});
// <source>:10:18: error: no matching function for call to 'oh_no(<brace-enclosed initializer list>)'
// oh_no({0,1,2});
```

```
template <class T>
void okay_i_guess (std::initializer_list<T> t) {

}

okay_i_guess({0,1,2});
```

```
template <class T, class... Args>
void okay_i_guess (std::initializer_list<T> t, Args&&... args) {

}

okay_i_guess({0,1,2}, 3, 4, 5);
```

```
auto a {0,1,2}; //compiler is sad
auto b {0}; //either int or std::initializer_list<int> depending on compiler age
auto c = {0,1,2}; //okay
```

## Syntax and semantics

```
std::vector a (1,2);   // 2
std::vector b {1,2};   // 1, 2
std::vector c ({1,2}); // 1, 2
std::vector d = {1,2}; // 1, 2
```

# Proposal

Provide constructors for containers which take a tag and a parameter pack of forwarding references:

```
template <class T>
struct vector {
    template <class... Args>
    vector (std::in_place_t, Args&&...);
};

vector yay (std::in_place, move_only{}, move_only{});
```

This addresses all three of the points in the motivation.

Move-only types are supported because the arguments will be perfect-forwarded inside the constructor.

This construct is deducible because it acts the same as a normal variadic template:

```
template <class... Args>
auto make_thing (Args&&...);

make_container(std::in_place, 0, 1, 2);
```

The syntax problem is reduced, as there won't be a semantic difference between `(...)` and `{...}` in the majority of cases:

```
struct no_init_list {
    template <class... Args>
    no_init_list(std::in_place_t, Args&&...);
};

struct init_list {
    template <class... Args>
    init_list(std::in_place_t, Args&&...);

    template <class U>
    init_list(std::initializer_list<U>);
};

no_init_list a (std::in_place, 0, 1); //okay
no_init_list b {std::in_place, 0, 1}; //okay

init_list c (std::in_place, 0, 1); //okay
init_list d {std::in_place, 0, 1}; //doesn't compile
init_list e {std::in_place};       //std::initializer_list<std::in_place_t>
```

Only that last case is inconsistent, which is somewhat unfortunate. However, in the case of the standard containers, the `std::initializer_list` parameter doesn't deduce it's template parameter, instead using `std::initializer_list<T>`, so that case wouldn't compile.

# `std` vs. `std2`

Given the last case in the previous section, and that lots of code using `std::vector` depends on its current form, this paper
considers introducing this constructor to namespace `std2`.

# Bikeshedding

## Alternative names

Some alternative names for the tag are: `std::init_with` and `std::init`.

# Other options

# Acknowledgements
