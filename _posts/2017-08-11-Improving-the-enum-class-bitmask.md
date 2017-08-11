---
tags: c++
title: Improving the Enum Class Bitmask
---
Back in 2016, I wrote [a blog post](https://dalzhim.github.io/2016/02/16/enum-class-bitfields/) about enum class bitfields that was influenced by [Anthony William's blog post](https://www.justsoftwaresolutions.co.uk/cplusplus/using-enum-classes-as-bitfields.html) about the very same topic: how to painlessly enable bitmask operators with the `enum class` feature that was introduced with C++11 for increased type safety and convenience. After using this in production code for a while, the amount of programming errors related to implicit conversions has diminished as usage has increased and legacy code has slowly converted. Yet, even by using this tool, I managed to write buggy code such as the following.
```cpp
auto someMask = f();
if (someMask == myEnum::someEnumerator)
```

That code worked for more than 95% of the cases… which is not good enough. Fixing it was very easy once the bug was spotted. Simply correct this way :
```cpp
auto someMask = f();
if (someMask & myEnum::someEnumerator)
```
Still, I wished this mistake could be detected at compile time. So I started working on a replacement for the previous solution that would make it impossible to use the `operator==` and `operator!=` operators to compare a bitmask with an enumerator. As someone pointed out on the Cpplang Slack channel, there are occasions when you want to make sure a mask has a single bit set and none other. This is one case where `mask == enumerator` is actually correct. I'll get back to this concern very shortly.

## Defining concepts
In order to make `mask == enumerator` a compilation error, two concepts are required : masks and enumerators. An enumerator's purpose is to give a name to a specific bit when it is *set*. A mask, on the other hand, represents the state of every bit (and this way, of every enumerator), whether they are *set* or *cleared*. The language already provides the `enum class` type which represents enumerators. From this point, I'll refer to `enum class` as type `E`. In order to represent masks, I define the `bitmask<E>` type.

If we get back to the `mask == enumerator` case that is considered valid. Why is it considered valid? Because the programmer really meant to use `operator==` because no other bit should be *set*. Well that violates the definition from the previous paragraph for the `enumerator` concept. The real intent when comparing a mask with an enumerator constant is to use that enumerator as a mask for the comparison. This means we require a convenient and intuitive way to promote `E` to `bitmask<E>`, like so:
```cpp
template <typename E> bitmask<E> make_bitmask(E);

if (mask == make_bitmask(myEnum::someEnumerator))
```
This makes the intent of this comparison very explicit, it is not a mistake that a single enumerator was promoted as a bitmask.

## Bitwise operations
Now let's talk about bitmask operations. The bitmask operations are `&` (AND), `|` (OR), `^` (exclusive OR, XOR), `~` (complement), `&=` (AND assignment), `|=` (OR assignment) and `^=` (XOR assignment). All of those operators are binary, except for the bitwise complement operator (`~`) which is unary. There are two different types every parameter can have, which results in 4 different permutations for the two parameters of the binary operators and 2 different types for the single parameter of the unary operation. Here is a summary the result types for every operation and permutation of parameters.

#### Binary bitwise operators

||**`E, E`**|**`E`, `bitmask<E>`**|**`bitmask<E>`, `E`**|**`bitmask<E>`, `bitmask<E>`**|
|**`operator&`**|`E`|`E`|`E`|`bitmask<E>`|
|**`operator|`**|`bitmask<E>`|`bitmask<E>`|`bitmask<E>`|`bitmask<E>`|
|**`operator^`**|`bitmask<E>`|`bitmask<E>`|`bitmask<E>`|`bitmask<E>`|
|**`operator&=`**|`bitmask<E>`|`bitmask<E>`|`bitmask<E>`|`bitmask<E>`|
|**`operator|=`**|`bitmask<E>`|`bitmask<E>`|`bitmask<E>`|`bitmask<E>`|
|**`operator^=`**|`bitmask<E>`|`bitmask<E>`|`bitmask<E>`|`bitmask<E>`|

#### Unary bitwise operators

||**`E`**|**`bitmask<E>`**|
|**`operator~`**|`bitmask<E>`|`bitmask<E>`|

#### Comparison operators

||**`E, E`**|**`E`, `bitmask<E>`**|**`bitmask<E>`, `E`**|**`bitmask<E>`, `bitmask<E>`**|
|**`operator==`**|`bool`|`static_assert`|`static_assert`|`bool`|
|**`operator!=`**|`bool`|`static_assert`|`static_assert`|`bool`|

As you may have noticed, there are only three bitwise operations that result with type `E`. Everything else returns the type `bitmask<E>`. Moreover, those three cases are three of the four permutations for a single operator : `operator&`. So the only way you can get back an `enumerator` from a `bitmask` is to apply a bitwise AND using at least one `enumerator` argument.

This design successfully enforces comparison of enumerator types with other enumerators of the same type, and of bitmask types with other bitmasks of the same type. Accidentally comparing an enumerator with a bitmask now triggers a static_assert that makes it explicit why this should be avoided.

## Bool conversion

There is still one annoying issue. It should be possible to convert `bitmask<E>` to `bool` by testing whether it is non-zero. Likewise, an enumerator `E` should be convertible to `bool` so that the following statement is valid:
```cpp
if (mask & enumerator)
```
Providing this conversion is done with the following function signature:
```cpp
constexpr explicit operator bool() const;
```
The `explicit` on the bool operator allows it to be [contextually convertible in five contexts](http://en.cppreference.com/w/cpp/language/implicit_conversion#Contextual_conversions):
1. controlling expression of `if`, `while`, `for`;
1. the logical operators `!`, `&&` and `||`;
1. the conditional operator `?:`;
1. `static_assert`;
1. `noexcept`. 

This is how we'll get implicit bool conversion for our `if` statements, while preventing other unwanted implicit conversions.

## Bool conversion (continued)

The `explicit operator bool` can be implemented on the `bitmask<E>` template, but it cannot work for type `E`. In order to make this utility as convenient as possible, I have also defined type `enumerator<T>` so that the result of `operator&` with at least one `enumerator` argument results in a type which is contextually convertible to `bool`. This made the tables up there a bit more complex. Rather than the original 4 overloads for every operator, there are now 9 required overloads to cover every parameter permutation:
1. `T, T`
1. `enumerator<T>, enumerator<T>`
1. `bitmask<T>, bitmask<T>`
1. `T, enumerator<T>`
1. `enumerator<T>, T`
1. `T, bitmask<T>`
1. `bitmask<T>, T`
1. `enumerator<T>, bitmask<T>`
1. `bitmask<T>, enumerator<T>`

Now that bitwise operations that produce an `enumerator<T>` are also contextually convertible to `bool`, the only thing left that is not is the original type `T`. This is actually a good thing as converting a single enumerator is always `true`. The only way it could be `false` is if you had an enumerator to represent `0`, but that would violate the concepts defined at the beginning of this article : the value 0 is a `mask`, not an `enumerator`.

The complete source code can be found in [my github repository](https://github.com/Dalzhim/ArticleEnumClass-v2).
