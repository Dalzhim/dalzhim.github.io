---
tags: c++
title: Enum class bitfields
---
<p>C++11's addition of scoped enumeration has given us the ability to create stronger types that can't be accidentally converted between integer types and enumeration types.</p>
```cpp
enum e
{
    kEnumerator1,
    kEnumerator2
};
int i = kEnumerator2; // implicit conversion
```

<p>It also prevents namespaces from being polluted by the enumerators as the enumerators are only accessible through the enum type like this :</p>
```cpp
enum class eClass
{
    kEnumerator1,
    kEnumerator2
};
eClass e = eClass::kEnumerator1;
```

<p>The main irritant when using scoped enumerations is that it doesn't work well with bitfields. Because enumerators are strongly typed, bitwise operators cannot be used the same way it is done with unscoped enumerators. This strong typing prevents an implicit conversion to an integer type which can be used with bitwise operators. Following this logic, everything works fine with explicit conversions.</p>
```cpp
enum class eClass
{
    kEnumerator1 = 1 << 0,
    kEnumerator2 = 1 << 1
};
eClass flag = eClass::kEnumerator1 & eClass::kEnumerator2; // Compilation error
eClass flag = static_cast<eClass>(static_cast<int>(eClass::kEnumerator1) & static_cast<int>(eClass::kEnumerator2)); // Success with explicit conversions
```

All those static_casts are tedious so we need something better. After researching this topic, I started rolling out my own solution. I got to a point where all the relevant operators were being generated through templates and SFINAE. It was working pretty well, and as I was closing in on a great solution, I knew a bit better what I was searching for and that's when I found [Anthony William's blog post](https://www.justsoftwaresolutions.co.uk/cplusplus/using-enum-classes-as-bitfields.html) about the very same topic. The interesting takeaway from that reading was that he had also thrown in an opt-in mecanism that restricts the operator generation only to the subset of enum classes that have opted-in. An exerpt of the result can be found below :
```cpp
template<typename T>
struct enable_enum_class_bitfield
{
    static constexpr bool value = false;
};
template<typename T>
typename std::enable_if<std::is_enum<T>::value && enable_enum_class_bitfield<T>::value, T>::type
operator&(T lhs, T rhs)
{
    typedef typename std::underlying_type<T>::type integer_type;
    return static_cast<T>(static_cast<integer_type>(lhs) & static_cast<integer_type>(rhs));
}
enum class eClass
{
    kEnumerator1 = 1 << 0,
    kEnumerator2 = 1 << 1
};
template<>
struct enable_enum_class_bitfield<eClass>
{
    static constexpr bool value = true;
};
eClass flag = eClass::kEnumerator1 & eClass::kEnumerator2;
```

<p>This operator overload template is meant to generate functions that will allow us to combine instances of an enum class’s enumerators using the familiar bitwise operators. Using SFINAE, we restrict the generation of those methods to types for which T is an enum type and the enable_enum_class_bitfield::value bool is true. In order to fulfill this second condition, a template specialization of the struct enable_enum_class_bitfield is required for the enum type on which we wish to enable bitwise operators. A macro is provided in order to make it easy to enable the bitwise operators without any knowledge about template specializations. This setup has the advantage of having no impact except where the programmer explicitly opts-in to the desired behavior.</p>
<p><strong>Lines 1-5 :</strong> Base definition of the enable_enum_class_bitfield where the initial value is false. This means that all types for which there is no specialization will opt-out of the bitwise operators.
<strong>Line 8 :</strong> Application of <a href="http://en.cppreference.com/w/cpp/language/sfinae" target="_blank">SFINAE</a> to enable operators only for specific types. There are two conditions: first, the type needs to be an enum type; second, the enable_enum_class_bitfields must be specialized to opt-in. Only then will the std::enable_if<…>::type expression resolve to T which is the appropriate return type for the operator overload.
<strong> Line 9 :</strong> Standard operator overload.
<strong> Line 11 :</strong> std::underlying_type::type resolves to the integer type backing the enum. This could be variety of types such as int, long, long long, unsigned variants, etc. This typedef allows us to cast enumerators to the corresponding integer type without any truncation problems in order to apply the bitwise operations.
<strong> Line 12 :</strong> Here we simply cast the enumerators to the underlying type, we apply the bitwise operator and we cast back into an enumerator.
<strong> Lines 20-24 :</strong> Specialization of the enable_enum_class_bitfield to opt-in into bitwise operators for enum type eClass.
<strong> Line 26 :</strong> Usage of the bitwise enumerator.</p>
<p>Finally, all that is left to do is to apply this solution for every bitwise operator : &, |, ^, ~, &=, |=, ^=. You can find the <a href="https://github.com/Dalzhim/ArticleEnumClass" target="_blank">complete source</a> on github.</p>
