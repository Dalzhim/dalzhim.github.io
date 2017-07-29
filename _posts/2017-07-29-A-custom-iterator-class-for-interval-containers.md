I wrote an article two years ago about a peculiar usage of Boost.ICL with a `DomainType` that changes (dynamically) at runtime. That domain could be described as the elements of a container ordered according to a strict weak relationship. The implementation I offered worked, but it suffered a few weaknesses. The present article will go over the various weaknesses and offer a new implementation that addresses those issues.

If you want to skim over the previous implementation, you can find it [on Github over here][1].

## Deficiencies in the code from the previous article
There are two deficiencies that are due to my previous lack of knowledge of the Boost.ICL extension points :
1. Declaring an interval container with the custom `DomainType` requires explicit customization of multiple template parameters beyond the `DomainType` and `CodomainType`.
1. Implementation details of the library are used as extension points

There are also two deficiencies that are due to a lack of abstraction in the domain's design :
1. The `std::less` specialization has wider impact than what is desirable.
1. There is overhead in representing an empty interval

## Declaring an interval container requires explicit customization of multiple template parameters

The `boost::icl::interval_map<…>` class template has 8 template parameters: `DomainType`, `CodomainType`, `Traits`, `Compare`, `Combine`, `Section`, `Interval` and `Alloc`. Each one of those has a default value, except the first two. My previous article was based on an implementation where the seventh parameter (`Interval`) had to be customized to `boost::icl::closed_interval`. This makes declaring one of those structures very verbose. It makes it error prone as well, as omitting the seventh parameter will compile just fine, but it produces a result riddled with use-after-free bugs.

As it turns out, there is an extension point that can be used to customize the default interval type used by interval containers. The interval containers rely on that extension point to provide the default value for the seventh template parameter, solving both aspects of this problem : it's not verbose anymore and it's not error prone either. Here's how it looks :

```cpp
namespace boost { namespace icl {
  template<>
  struct interval_type_default
  {
   using type = boost::icl::closed_interval;
  };
}}
```

## Implementation details of the library are used as extension points
The detail namespace is a convention used by header only library to isolate implementation details from the public interface of the library. That means that specializing templates within that namespace is not future-proof and might break with some future update.

Also, the reason those implementation details were being specialized in the previous implementation was that `DomainType` didn't really respect the requirements documented by Boost.ICL for that type. Even though, syntactically, a pointer does offer `operator++` and `operator--` as required, they don't have the proper semantics: incrementing to the next element of the domain and decrementing to the previous element. Hence the specializations of the `successor` and `predecessor` templates (which are within the detail namespace) fixed the semantic issue.

Now, in order to clean this up, we'll need a new abstraction. But before presenting that solution, let's cover the remaining deficiencies.
The `std::less` specialization has wider impact than what is desirable.

The problem here is that specializing `std::less` has an impact that goes beyond the interval containers for which it has been designed. Multiple data structures such as `std::map` and `std::set`, and multiple algorithms such as `std::sort` and `std::partition` use `std::less` as their default ordering relationship.

A potential solution to this problem would be to provide a custom ordering relationship functor to the fourth template parameter (Compare) of our interval containers. It does solve the need for a `std::less` specialization that has a wider impact than what is desired, but it reintroduces the problems we just solved with the seventh template parameter (`Interval`) that made it both verbose and error prone to declare interval containers.

Once again, to clean this up, we'll need a new abstraction. But before presenting that solution, let's cover the remaining deficiencies.

## There is overhead in representing an empty interval
The last problem about the previous implementation is representing an empty interval. The very nature of the domain means that the set of domain elements can be empty at some point in time. In such a situation, there are only two iterators that can possibly be created. One corresponds to the sentinel iterator of the domain elements container (corresponds to `end()`). The other iterator that can be created is a `DefaultConstructed` iterator that isn't bound to any container.

The `boost::icl::closed_interval` template represents the empty interval with an upper bound that compares less than the lower bound. In some cases, it may even use a `DefaultConstructed` `DomainType` instance and an incremented `DefaultConstructed` `DomainType` instance (also called the unit element) rather than two existing domain elements.

This was a problem, because a `DefaultConstructed` `DomainType` instance cannot be incremented. The chosen solution was to specialize an extension point of the library : `is_empty`. While the default implementation of `is_empty` returns `true` if the upper bound compares less than the lower bound, the specialization would check that both bounds are not equal to a `DefaultConstructed` `DomainType` before making the usual comparison. That's where the overhead lies. There are two checks that have to be performed before we can safely make the comparison we meant to do.

## Better abstractions to the rescue

In general terms, the solution to the different problems outlined previously is to provide a custom `DomainType` that respects both the syntactic and semantic requirements imposed by the interval containers. Those requirements are the following :
* [DefaultConstructible][2]
* [CopyConstructible][3]
* [StrictWeakOrdering<DomainType, Compare>][4]
* IsIncrementable
* IsDecrementable
* HasUnitElement

In other words, here's the interface we expect from `DomainType` :
* DomainType()
* DomainType(const DomainType&)
* bool operator<(const DomainType&)
* DomainType& operator++()
* DomainType& operator--()

We've already understood that a pointer conforms to those requirements syntactically (everything compiles), but not semantically (leads to the wrong behavior). That means we need to build an abstraction layer on top of them that is conforming in both regards.

That abstraction layer is an iterator. It needs to be a ​`BidirectionalIterator` that also offers `operator<`. Also, the iterators must never be invalidated unless the underlying element is actually removed from the domain.

In the new implementation, I wrapped `std::map<K, V>::const_iterator` within my own iterator class where I've provided `operator<` in a way that requires dereferencing both iterators that are being compared. It solves quite a few problems :
* The iterator doesn't depend on implementation details of Boost.ICL (`predecessor` and `successor`)
* The iterator doesn't require a `std::less` specialization that has wider impact than desired
* The iterator doesn't require a custom `Compare` functor to be given to the interval containers

But there are also some problems that aren't yet solved :
* It requires a specialization of `is_empty` because the `DefaultConstructed` iterator cannot be incremented
* It requires a specialization of `unit_element` because the `DefaultConstructed` iterator cannot be incremented

## One last ugly hack
What if the `DefaultConstructed` iterator could be incremented? That would solve the last two pending issues. Well I only know of one way, it is that the `DefaultConstructed` iterator should be bound to an existing data structure that has enough elements to allow for incrementation. This is how it looks :

```cpp
class dynamic_domain_iterator
{
public:
  dynamic_domain_iterator() : it(g_default_container.begin()) {}
  \[…]

private:
  const static std::map<K, V> g_default_container;
};

const std::map<K, V> dynamic_domain_iterator::g_default_container\{\{…element1…}, \{…element2…}};
```

This solves the overhead problem introduced by the specialization of `is_empty`, and it also makes it completely unnecessary to specialize both `is_empty` and `unit_element`. I'm not sure this last piece of the puzzle could be considered very clean though. If anyone knows better, please let me know.

The complete code can be found [on GitHub][5].

[1]: https://github.com/Dalzhim/ArticleDynamicFiniteDomain/blob/fb45ca0b805aeb2a3eaac3619bdf318fbd80fbb4/main.cpp

[2]: http://en.cppreference.com/w/cpp/concept/DefaultConstructible

[3]: http://en.cppreference.com/w/cpp/concept/CopyConstructible

[4]: https://en.wikipedia.org/wiki/Weak_ordering#Strict_weak_orderings

[5]: https://github.com/Dalzhim/ArticleDynamicFiniteDomain-v2
