---
layout: post
title: Exploiting simultaneous pack-expansion inside macros
subtitle: A curious solution, for a rare problem.
gh-repo: DNKpp/mimicpp
gh-badge: [star, fork, follow]
tags: [c++,macro,template,pack-expansion]
comments: true
readtime: true
mathjax: false
---

Even if I try to avoid macros as much as possible when writing regular c++ code, sometimes they are still a valuable option for getting things done.
In this case it's all about a convenience macro, which simplifies the process of declaring mocks with my own c++-20 mocking-framework [mimic++](https://github.com/DNKpp/mimicpp).

If you don't know ``mimic++`` yet, that's ok. It merely provides the hook for the problem. Let me just provide a little bit of context:

In ``mimic++`` mocks are just template-types with a call-operator, which will also be used when mocking ``virtual`` functions.
The latter requires some sort of redirect, which unfortunately leads to some tedious boilerplate-code.
This is where the macro mentioned above comes into play:
```cpp
// function_name: The function to be mocked (e.g. "foo")
// return_type: The return type of that function (e.g. "void")
// param_list: The param-list (e.g. "()" or "(int, std::string&)")
#define MOCK_METHOD(function_name, return_type, param_list) ... // the actual definition is not relevant here
```

Consider the following interface:
```cpp
struct Interface
{
    virtual void foo(int) = 0;	// function to be mocked
};
```

To create a mock type, we could come up with this:
```cpp
struct MyMock : Interface
{
    MOCK_METHOD(foo, void, (int));
};
```

The macro internally calls several other macros, until it eventually expands to the following (simplified) code:
```cpp
struct MyMock : Interface
{
    mimicpp::Mock<void()> foo_{};         // the actual mock object
    void foo(int arg_i) override
    {
        foo_(std::forward<int&&>(arg_i)); // forwards the call to the mock object "foo_"
    }
};
```

{: .box-note}
**NOTE** For the sake of readability, I'll omit the required \ characters at the end of each line in the macro definitions provided. All contiguous lines following a ``#define`` statement are part of that definition.

As you can see, the macro internally has to generate multiple parts of which the ``std::forward<int&&>(arg_i)`` is the relevant part for this post.
This is generated by the macro ``MIMICPP_DETAIL_FORWARD_ARG`` (simplified version):
```cpp
#define MIMICPP_DETAIL_FORWARD_ARG(param_type, param_name) std::forward<param_type>(param_name)
```
For the majority of cases, this is sufficient and get's the job done.

### Mocking variadic-template interfaces

{: .box-note}
**NOTE** Quick refresher: Variadic-templates are templates, which accept an arbitrary amount of template-arguments (see: [cppreference](https://en.cppreference.com/w/cpp/language/pack)).

Consider another interface:
```cpp
template <typename... Args>
struct VariadicInterface
{
    virtual void foo(Args...) = 0;
};
```
At a first glance, this doesn't look like an issue for the macro ``MOCK_METHOD``, but due to how the internal macro ``MIMICPP_DETAIL_FORWARD_ARG`` is defined, this won't compile. Let's see why.

The internal macro is invoked like ``MIMICPP_DETAIL_FORWARD_ARG(Args..., some_name)``, which expands to ``std::forward<Args...>(some_name)``.
This has two issues.
  - ``std::forward`` is not a variadic-template and thus does not accept an arbitrary amount of template-arguments and
  - ``some_name`` refers to a [pack](https://en.cppreference.com/w/cpp/language/pack), which must be expanded via ``...``.
  
What we actually need is this: ``std::forward<Args>(some_name)...``

As this is a feature that I really want to support, I've accepted the challenge.

{: .box-note}
**NOTE** Interestingly, neither ``trompeloeil`` nor ``gmock`` currently support this (see: [trompeloeil-example](https://godbolt.org/z/qfo77enTT) and [gmock-example](https://godbolt.org/z/nbzx96K49)).

### Solving the problem

{: .box-note}
**NOTE**  Admittedly, my preprocessor-skills are very limited, so I do not know whether a simple macro-solution exists for that kind of purpose. In the following I'll focus on solutions within the actual c++-language.

To be able to actually solve that problem, I first had to understand it.

There are two possible forms, which ``param_type`` can have:
- Either in form of ``T``, which then denotes a (possibly cv-ref qualified) type, or
- in form of ``T...``, which then requires a pack-expansion.

``param_name`` on the other hand is always just a plain identifier, which in the first case can be treated as regular param or — in the second case — must also be treated as a pack.

#### An attempt has been made...

The first (rather naïve) idea I came up with, was to detect whether the ``param_type`` argument denotes an actual type-identifier or a pack and — depending on the outcome — doing one thing or the other.
But this was more challenging than anticipated, as there is no official type-trait or any other support from the ``stl`` or language.
Another issue is, that when I do pass ``param_type`` to an actual ``template``/``concept`` I would lose the information whether it was a pack or not.
So I experimented with some kind of in-place ``requires``-statement:
```cpp
#define FORWARDING_MACRO(param_type, param_name)
    if constexpr (requires{ sizeof...(param_type); }) { // intention: does the operator ``sizeof...`` form a valid expression?
        std::forward<param_type>(some_name)...;
    }
    else {
        std::forward<param_type>(some_name)
    }
```
Well, as this is just plain wrong syntax, and not some kind of *substitution-failure*, this didn't work.
But at least this lead me to another idea.

#### A solution

By definition, pack-expansion (via ``...``) can encompass multiple packs; the only requirement is, that they have the same length.
So, instead of *conditionally detecting* whether an expansion is necessary...
Can I simply do a pack-expansion in any case and let the compiler decide, whether it expands a single pack or multiple packs simultaneously?
That would require some kind of promotion: From a type to a pack.

The trick is to somehow pass the ``param_type`` into a variadic-template.
A function for example could then take over the forwarding job:
```cpp
#define FORWARDING_MACRO(param_type, param_name) forwarding_function<param_type>(param_name)
```
But, wait. That won't work either, as we still have to conditionally expand the ``param_name``.
Thankfully, c++ has a feature, which acts like a function, but can also provide access to the outer scope: Lambdas to the rescue!

Let's see how this can look like:
```cpp
template <typename... Args>
struct type_list {};

#define FORWARDING_MACRO(param_type, param_name)
    [&]<template... types>(type_list<types...>) { // note the & capture
        std::forward<types>(param_name)...;       // unfortunately that's not very useful...
    }(type_list<param_type>{})                    // invoke the lambda immediately
```
That looks quite promising, but we are not done yet, because the lambda actually does nothing useful.
The question is, what shall I return from that lambda?
In fact, I decided to always create a ``std::tuple`` with appropriate references as elements and simply return that from the lambda,
as this resulted in just a few simple changes in the surrounding macros.

Eventually the solution looks like this:
```cpp
#define FORWARDING_MACRO(param_type, param_name)
    [&]<template... types>(type_list<types...>) {
        return std::forward_as_tuple(std::forward<types>(some_name)...);
    }(type_list<param_type>{})

// FORWARDING_MACRO(int, someName) expands to this:
[&]<template... types>(type_list<types...>) {
    return std::forward_as_tuple(std::forward<types>(someName)...); // ``types`` is a pack, but ``someName`` is just a regular param, thus only ``types`` is expanded.
}(type_list<int>{})

// FORWARDING_MACRO(Args..., someName) expands to this:
[&]<template... types>(type_list<types...>) {
    return std::forward_as_tuple(std::forward<types>(someName)...); // Both, ``types`` and ``someName``, are packs (guaranteed to be of same length),
}(type_list<Args...>{})                                             // thus the compiler expands them both simultaniously.
```

### Conclusion

The whole story is definitely an edge-case one does not normally have to solve very often, but I actually found it quite entertaining.
It also makes me a little bit proud, that ``mimic++`` supports a feature which other well-known alternatives have their struggle with.

Hopefully, that little story was also interesting for you, too.
Feel free to leave a comment and share your thoughts!
I also encourage you to check out ``mimic++`` as your next mocking framework, and I look forward to hearing about your experiences!

See you the next time,
Dominic
