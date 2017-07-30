I have been exploring some particular use cases with boost ICL's interval_map in order to map values to intervals based on a dynamic finite domain. I would like to share what I have learned in the process so I will start with a quick overview of a few concepts of the ICL and then show some code snippets to get it working.

## Quick introduction to boost ICL
<p>The Interval Container Library (ICL) of boost offers interesting data structures to work with ranges. The tutorial of the library takes you through the common uses of the library by illustrating the examples with date/time related scenarios that make it easy to understand how those data structures solve real problems. As an example, you could fill an <code>interval_set</code> with business days so that you can easily look up whether or not the business will be opened on a specific date. You could also use an <code>interval_map</code> to create a schedule where employees are assigned to specific days, allowing you to easily look up who is working on a specific day.</p>
<p>An <code>interval_map</code> is very similar to a <code>std::map</code> as its <code>value_type</code> is a <code>std::pair</code> that associates a <code>key_type</code> with a <code>mapped_type</code>. In this particular case, the <code>key_type</code> is an interval based on the domain of the <code>interval_map</code>, and the mapped type is the codomain of the <code>interval_map</code>. These are the first template parameters that an <code>interval_map</code> accepts (<code>domain_type</code> and <code>codomain_type</code>). Using the chosen <code>domain_type</code>, an <code>interval_map</code> can represent <code>interval_type</code> with open intervals, semi-open intervals, or closed intervals. In any case, an interval is always made of two <code>domain_type</code> values. Hence, <code>interval_type</code> is templated on <code>domain_type</code>.</p>
<p>There are three different <code>interval_map</code>s which handle the underlying pairs in different ways. The first is the <code>boost::icl::interval_map</code>. Whenever there are overlapping or adjacent intervals that share the same codomain value, they will be merge together into a single <code>value_type</code> instance. The <code>boost::icl::separate_interval_map</code> has a slightly different behavior as it will only merge overlapping ranges. Adjacent ranges will be kept separate. Finally, the <code>boost::icl::split_interval_map</code> doesn't merge anything at all, so by adding the ranges [1, 4] and [2, 3], we end up with the ranges [1, 1], [2, 3] and [4, 4]. When overlapping occurs, the various <code>interval_map</code> structures handle the codomain value according to the type <code>codomain_combine</code> which makes the combining behavior customizable.</p>

## What is a dynamic finite domain?
<p>Now, what exactly do I mean by a dynamic finite abstract domain? Dynamic means the elements part of the domain may change over time. As an example, if we would consider the first names of everyone in the room you are currently in as a domain, it would be dynamic as it will most certainly change at some point in time. The dynamic domain I will implement for the interval_map is based on an ordered set of objects. As for finite, there is a finite amount of elements within the ordered set. It can be zero, it can be 5000, etc. The important point is that it even if it may differ over time, it can always be counted.</p>
<p>An important thing to note is that once an object becomes part of the domain, its position relative to neighboring elements of the domain cannot be changed without breaking things down. If the ordering of an element would change because of a modification, then it means that the object has become a different element of the domain. In order to do so, it must first be removed from the ordered set, then it can be modified, and finally it can be inserted again.</p>
<p>Let's start with this short piece of code where I assume that DynamicDomain is a simple wrapper for <code>int</code> and is ordered the same way:
```cpp
class DynamicDomain;
class Value;
std::set<std::unique_ptr<DynamicDomain>> domainSet;
domainSet.insert(std::unique_ptr(new DynamicDomain(1)));
domainSet.insert(std::unique_ptr(new DynamicDomain(2)));
domainSet.insert(std::unique_ptr(new DynamicDomain(3)));
domainSet.insert(std::unique_ptr(new DynamicDomain(4)));
domainSet.insert(std::unique_ptr(new DynamicDomain(5)));
typedef boost::icl::interval_map<DynamicDomain*, Value> MapType;
MapType intervalMap;
```

Please note that for simplicity, <code>domainSet</code> holds <code>unique_ptr</code> instances and <code>MapType</code> holds raw pointers. In order to use <code>shared_ptr</code> in both cases, the ICL's <code>interval_map</code> would need a different solution than what I'm about to present in this article. As such, the elements in <code>domainSet</code> must outlive the raw pointers used in <code>intervalMap</code>.</p>
<p>There are 5 requirements to use `DynamicDomain*` as the <code>domain_type</code>:</p>
<ol>
<li><code>domainSet</code> must be ordered according to the <a href="http://en.cppreference.com/w/cpp/concept/LessThanComparable" target="_blank">LessThanComparable</a> concept</li>
<li><code>DynamicDomain</code> instances can be modified as long as their ordering is preserved</li>
<li><code>MapType</code> must use <code>closed_interval</code> exclusively</li>
<li><code>DynamicDomain</code> instances shall be able to return pointers to their immediate neighbors within <code>domainSet</code></li>
<li>A <code>nullptr</code> bound within an interval implies an empty interval</li>
</ol>

## 1. <code>domainSet</code> must be ordered according to the <code>LessThanComparable</code> concept
<p>Both <code>std::set</code> and <code>interval_map</code> use <code>std::less</code> as the default comparator. The implementation of this standard functor calls <code>operator<</code>. But this won't order pointers properly because in that case, it will result in an arithmetic comparison of the addresses. That's not the behavior that we are looking for. In order to provide a more useful behavior, we need to specialize <code>std::less</code> for `T = DynamicDomain*`.
```cpp
namespace std
{
  template <>
  struct less<DynamicDomain*>
  {
    bool operator()(const DynamicDomain* lhs, const DynamicDomain* rhs) const
    {
      return lhs->intValue < rhs->intValue;
    }
  };
}
```

## 2. <code>DynamicDomain</code> instances can be modified as long as their ordering is preserved
<p>If we change the intValue of the first element of domainSet to 6, then the order has been broken and no operation (insertion, lookup, removal) shall be performed on the container until the ordering is restored. Otherwise, operations might randomly fail or succeed depending on which elements are tested by binary searches. We can then proceed to modify the rest of the DynamicDomain instances and once we every instance has been incremented by 5, all operations on the container become valid once again. In a similar way, any corresponding `DynamicDomain*` values in <code>intervalMap</code> should be erased before it can be removed from <code>domainSet</code>, otherwise all operations might fail.</p>

## 3. <code>MapType</code> must use <code>closed_interval</code> exclusively
<p>The ICL offers two categories of interval types : static and dynamic. Static interval types are <code>boost::icl::open_interval</code>, <code>boost::icl::left_open_interval</code>, <code>boost::icl::right_open_interval</code> and <code>boost::icl::closed_interval</code>. Those types correspond to different interpretations of a pair of bounds. An open bound is an exclusive bound, while a closed bound is an inclusive one. In other words, open bounds perform comparisons with <code>operator<</code> and <code>operator></code> while closed bounds perform comparisons with <code>operator>=</code> and <code>operator<=</code>. Mathematical representations of these types of interval are (lower, upper), (lower, upper], [lower, upper) and [lower, upper] respectively.</p>
<p>On the other hand, dynamic interval types are <code>boost::icl::continuous_interval</code> and <code>boost::icl::discrete_interval</code>. The difference between dynamic and static interval types, is that dynamic intervals may mix any of the four static interval types in the same collection, while static interval types won't.</p>
<p>The reason I am telling you about the various types of interval is that only one among them can be safely used within <code>MapType</code> and all the others might lead to crashes. The answer is <code>boost::icl::closed_interval</code>. Let's suppose <code>intervalMap</code> holds one single range: [0, 4]. Now we mean to get rid of the fifth <code>DynamicDomain</code> instance, so we ask the map to remove it from its range and the map changes its internal representation this way: [0, 4). We effectively removed the fifth element from the range, but <code>intervalMap</code> still holds a pointer to it. Which means it might start crashing when the real object is actually removed from <code>domainSet</code> and freed.</p>
<p>The reason for which the map has changed its internal representation that way is that the default interval type used for pointers is <code>boost::icl::discrete_interval</code>, which is a dynamic interval type, which allows a closed range to change into a half-open range. Thus we need to make sure it only ever uses <code>boost::icl::closed_interval</code> to avoid this problem. Here's a new typedef for <code>MapType</code> which does the trick:
```cpp
typedef boost::icl::interval_map<DynamicDomain*, Value, boost::icl::partial_absorber, std::less, boost::icl::inplace_plus, boost::icl::inter_section, boost::icl::closed_interval<DynamicDomain*>> MapType;
```

We had to find out the default values of many template parameters in order to change the <code>interval_type</code>. This might get tedious if we have many different maps to build this way. So here's a generic typedef that will default to <code>closed_interval</code> instances and make it easier to create typedefs for pointer-based interval maps:
```cpp
template <typename Domain, typename Codomain, typename Traits = boost::icl::partial_absorber, template<class>class Compare = std::less, template<class>class Combine = boost::icl::inplace_plus, template<class>class Section = boost::icl::inter_section>
using closed_interval_map = boost::icl::interval_map<Domain*, Codomain, Traits, Compare, Combine, Section, boost::icl::closed_interval<Domain*>>;

typedef closed_interval_map<DynamicDomain, Value> MapType;
```

Now we can use the <code>closed_interval_map</code> type and provide it with the <code>Domain</code> and the <code>Codomain</code> which helps us keep things sane when defining multiple types.</p>

## 4. <code>DynamicDomain</code> instances shall be able to return pointers to their immediate neighbors within <code>domainSet</code>
<p>First, let's start by the problem that motivates this requirement: crashes. Even with all that we've done up to now, overlapping ranges will crash. Let's suppose <code>domainSet</code> contains five elements and <code>intervalMap</code> contains a single range [0, 4]. Now let's insert a new range: [2, 2]. Suddenly, the internal representation of the map goes from holding one range, to holding three of them : [0, 1], [2, 2] and [3, 4]. Well, how is the map supposed to guess the addresses of elements 1 and 3 if we have never provided it directly? It does so through the <code>boost::icl::detail::successor</code> and <code>boost::icl::detail::predecessor</code> templates. The responsibility of these templates is to provide the neighbouring elements of the domain when provided any element of the domain.</p>
<p>It turns out that there are two variations of both templates, each of which receives a distinct bool parameter. It is an optimization meant to avoid if conditions on whether or not the sorting is <code>std::less</code> (the default) or <code>std::greater</code>.</p>
<p>Now we need to do quite a lot of work to get this working. First, the <code>DynamicDomain</code> elements must be aware of their siblings. Second, we need to specialize the ICL templates.</p>
<p>Here is a quick way to prototype our little experiment. The class has a pointer to the collection that owns it. When looking for neighbours, it simply looks itself up and then finds the neighbour. A better implementation could be based on a double linked list and the domain type could be list nodes holding the `DynamicDomain*`. Anyway, here's the simple prototype:
```cpp
class DynamicDomain
{
public:
  DynamicDomain(int intValue, std::set<DynamicDomain*>* collection);

  DynamicDomain* getPredecessor();
  DynamicDomain* getSuccessor();

  int intValue;

private:
  std::set<DynamicDomain*>* collection;
};

std::set<std::unique_ptr<DynamicDomain>> domainSet;
domainSet.insert(std::unique_ptr(new DynamicDomain(1, &amp;domainSet)));
domainSet.insert(std::unique_ptr(new DynamicDomain(2, &amp;domainSet)));
domainSet.insert(std::unique_ptr(new DynamicDomain(3, &amp;domainSet)));
domainSet.insert(std::unique_ptr(new DynamicDomain(4, &amp;domainSet)));
domainSet.insert(std::unique_ptr(new DynamicDomain(5, &amp;domainSet)));
```

And here are the template specializations:
```cpp
namespace boost { namespace icl { namespace detail {
  template<>
  struct successor<DynamicDomain*, true>
  {
    inline static DynamicDomain* apply(DynamicDomain* value)
    {
      return value->getSuccessor();
    }
  };

  template<>
  struct successor<DynamicDomain*, false>
  {
    inline static DynamicDomain* apply(DynamicDomain* value)
    {
      return value->getPredecessor();
    }
  };

  template<>
  struct predecessor<DynamicDomain*, true>
  {
    inline static DynamicDomain* apply(DynamicDomain* value)
    {
      return value->getPredecessor();
    }
  };

  template<>
  struct predecessor<DynamicDomain*, false>
  {
    inline static DynamicDomain* apply(DynamicDomain* value)
    {
      return value->getSuccessor();
    }
  };
}}}
```

## 5. A <code>nullptr</code> bound within an interval implies an empty interval
<p>In order to fix some more crashes when using <code>mapInstance</code>, we need to help the <code>interval_map</code> understand that some of the elements it discovers through the successor and predecessor templates are invalid as bounds. This invalid value is <code>nullptr</code>. We must once again specialize one of the ICL's templates: <code>boost::icl::is_empty</code>. The default implementation of <code>is_empty</code> for <code>closed_interval</code> checks if the upper bound is smaller than the lower bound. Because this implies dereferencing our pointers, we need to check for <code>nullptr</code> before the default behavior can be applied.
```cpp
namespace boost { namespace icl {
  template
  bool is_empty<boost::icl::closed_interval>(const boost::icl::closed_interval&amp; object)
  {
    return object.lower() == nullptr || object.upper() == nullptr || boost::icl::domain_less<boost::icl::closed_interval>(object.upper(), object.lower());
  }
}}
```

## Conclusion
<p>Now that we have met the 5 requirements stated earlier, we can use the members of <code>domainSet</code> as a dynamic finite domain for the ICL's data structures. This has been most useful to me in order to solve problems where an interval_map would be appropriate except for one problem: some operations invalidate a large part of the data structure because the discrete values used in ordering the DynamicDomain instances would shift. As an example, consider a spreadsheet software. When you select rows 5-10 and you insert a new row at index 3, then your selection has shifted to rows 6-10. With the dynamic finite domain, I can make changes to the discrete values and prevent the <code>interval_map</code> from being invalidated.</p>
<p>The complete code for this article can be found over here : https://github.com/Dalzhim/ArticleDynamicFiniteDomain</p>
