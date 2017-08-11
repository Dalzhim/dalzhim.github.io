---
tags: c++
title: Improving the Enum Class Bitmask
---
Back in 2016, I wrote [a blog post](https://dalzhim.github.io/2016/02/16/enum-class-bitfields/) about enum class bitfields where I've explained how to painlessly enable bitmask operators with the `enum class` feature that was introduced with C++11 for increased type safety and convenience. After using this in production code for a while, the amount of programming errors related to implicit conversions has diminished as usage has increased and legacy code has slowly converted. Yet, even by using this tool, I managed to write buggy code such as the following.
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

In order to make `mask == enumerator` a compilation error, two concepts are required : masks and enumerators. An enumerator's purpose is to give a name to a specific bit. A mask, on the other hand, represents the state of every bit (and this way, of every enumerator), whether they are *set* or *cleared*. The language already provides the `enum class` type which represents enumerators. From this point, I'll refer to `enum class` as type `E`. In order to represent masks, I define the `bitmask<E>` type.

If we get back to the `mask == enumerator` case that is considered valid. Why is it considered valid? Because the programmer really meant to use `operator==` because no other bit should be *Set*. Well that violates the definition from the previous paragraph for the `enumerator` concept. The real intent when comparing a mask with an enumerator constant is to use that enumerator as a mask for the comparison. This means we require a convenient and intuitive way to promote `E` to `bitmask<E>`, like so:
```cpp
template <typename E> bitmask<E> make_bitmask(E);

if (mask == make_bitmask(myEnum::someEnumerator))
```
This makes the intent of this comparison very explicit, it is not a mistake that a single enumerator was promoted as a bitmask.

Now let's talk about bitmask operations. The bitmask operations are `&` (AND), `|` (OR), `^` (exclusive OR, XOR), `~` (complement), `&=` (AND assignment), `|=` (OR assignment) and `^=` (XOR assignment). All of those operators are binary, except for the bitwise complement operator (`~`) which is unary. There are two different types every parameter can have, which results in 4 different permutations for the two parameters. Here is a table that summarizes the result type for a given operation and a specific permutation of parameters.

|**Binary**|**`E, E`**|**`E`, `bitmask<E>`**|**`bitmask<E>`, `E`**|**`bitmask<E>`, `bitmask<E>`**|
|**`operator&`**|`E`|`E`|`E`|`bitmask<E>`|
|**`operator|`**|`bitmask<E>`|`bitmask<E>`|`bitmask<E>`|`bitmask<E>`|
|**`operator^`**|`bitmask<E>`|`bitmask<E>`|`bitmask<E>`|`bitmask<E>`|
|**`operator&=`**|`bitmask<E>`|`bitmask<E>`|`bitmask<E>`|`bitmask<E>`|
|**`operator|=`**|`bitmask<E>`|`bitmask<E>`|`bitmask<E>`|`bitmask<E>`|
|**`operator^=`**|`bitmask<E>`|`bitmask<E>`|`bitmask<E>`|`bitmask<E>`|

|**Unary**|**`E`**|**`bitmask<E>`**|
|**`operator~`**|`bitmask<E>`|`bitmask<E>`|

As you may have noticed, there are only three operations that result with type `E`. Everything else provides a `bitmask<E>`. Moreover, those three cases are three of the four permutations for a single operator : `operator&`.
