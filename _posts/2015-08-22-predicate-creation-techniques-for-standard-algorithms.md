---
tags: c++ stl
title: Predicate Creation Techniques for Standard Algorithms
---
Predicates are functions that return a boolean result upon being called. A predicate example is a function `bool isToday(Date& date)` which returns if the supplied date object corresponds to the current day or not. The Standard Template Library (STL) has many algorithms to which predicates can be supplied : `count_if`, `find_if`, `copy_if`, `remove_if`, etc. You might have noticed the examples I have given share the suffix `_if` which hints to the presence of an argument which is a predicate. In this article, I will go over many different ways to implement a predicate that can be supplied to the STL algorithms in order to extract some elements from a collection. First, here is the collection that will be filtered :

```cpp
class A
{
};
class B
{
public:
    B(int i, A* a) : i(i), a(a) {}
    A* getA() {return a;}
public:
    int i;
    A* a;
};
std::unique_ptr<A> a1 = std::make_unique<A>();
std::unique_ptr<A> a2 = std::make_unique<A>();
std::vector<std::unique_ptr<B>> owningVector;
owningVector.reserve(6);
owningVector.emplace_back(new B(1, a1.get()));
owningVector.emplace_back(new B(2, a2.get()));
owningVector.emplace_back(new B(3, a1.get()));
owningVector.emplace_back(new B(4, a2.get()));
owningVector.emplace_back(new B(5, a1.get()));
owningVector.emplace_back(new B(6, nullptr));
std::vector<B*> objects; // Source collection
objects.reserve(owningVector.size());
std::for_each(owningVector.begin(), owningVector.end(), [&objects](std::unique_ptr<B>& b) {
     objects.push_back(b.get());
});
std::vector<B*> result; // Destination collection
```

The first way we might implement a predicate is the implementation of a function. This technique allows the implementation of a simple condition without too much boilerplate. Here is a predicate to identify instances of `B` that do not have an associated instance of `A`.
```cpp
bool predicate0(B* b)
{
    return b->getA() == nullptr;
}
```

With `predicate0`, it is possible to use `std::copy_if` to copy the matching elements into a result collection. `std::copy_if` has 4 parameters : `first`, `last`, `output` and `predicate`. `first` and `last` are both iterators that define the range over which the algorithm will iterate. `output` is an <a href="http://en.cppreference.com/w/cpp/concept/OutputIterator">OutputIterator</a> used to copy elements into a destination collection. In order to supply an OutputIterator, one is constructed using [`std::back_inserter`](http://en.cppreference.com/w/cpp/iterator/back_inserter). This will append matching elements at the end of the destination collection. Finally, `predicate` is the function that will return true or false.
```cpp
std::copy_if(
    objects.begin(),
    objects.end(),
    std::back_inserter(result),
    predicate0 // This is the predicate
);
// Result : {6}
```

After executing `std::copy_if`, a single instance of `B` is copied into the destination collection and it has the value 6. It was pretty simple, but this technique is also pretty limited. If we wanted to filter out all the instances of `B` which are associated with a specific instance of `A`, it would require a static variable and it wouldn't be <a href="https://en.wikipedia.org/wiki/Thread_safety">thread-safe</a>. The problem is that our function cannot hold any state.

In order to hold an associated state with a predicate function, a <a href="https://en.wikipedia.org/wiki/Function_object">functor</a> can be implemented. A functor is a  function object that solves the problem of having a state. It can serve as an ordinary function because it implements `operator()`. In the context of this article's example, the functor will consist in a new `struct` that has an instance variable of type `A*`. Then, within the `struct`'s `operator()` definition, this instance variable will be compared to the elements of the collection.

```cpp
struct Functor
{
    Functor(A* a) : a(a) {}

    bool operator()(B* b) const
    {
        return b->getA() == a;
    }

public:
    A* a;
};
```

By implementing `operator()`, instances of `struct Functor` are now <a href="http://en.cppreference.com/w/cpp/concept/Callable">Callable</a>. The Callable concept encompasses everything that can be called as an ordinary function. As an example, here is how `struct Functor` can be called:
```cpp
std::unique_ptr<B> someBInstance = std::make_unique<B>();
Functor myFunctor(a1.get());
bool result = myFunctor(someBInstance.get());
```

And this is precisely what the `std::copy_if` algorithm will do: it will call the predicate by passing every element of the range `[first, last)` and evaluate the result of the function to decide whether to copy the element or not. Thus, all the instances of `B` associated with a specific instance of `A` can now be copied into the destination collection using the following expressions:
```cpp
std::copy_if(
    objects.begin(),
    objects.end(),
    std::back_inserter(result),
    Functor(a1.get()) // This is the predicate
);

// Result : {1, 3, 5}

std::copy_if(
    objects.begin(),
    objects.end(),
    std::back_inserter(result),
    Functor(a2.get()) // This is the predicate
);

// Result : {2, 4}
```

This is much more flexible than using functions, but there's also a lot more boilerplate to write (i. e.: the class, the instance variables and the constructor). But there is a solution to this problem: <a href="http://en.cppreference.com/w/cpp/language/lambda">lambdas</a>. Lambdas can be used to create anonymous functors with a very succinct syntax: `[Capture](Parameters) -> ReturnType {/*Code*/}`. `Capture` is a comma separated list of variables for which the anonymous class will have corresponding instance variables. `Parameters` is the list of parameters that the anonymous class's `operator()` function will accept. `ReturnType` is going to be the return type of the `operator()` function. Finally, `Code` will be the body of the `operator()` function. In other words, here is how a lambda looks like:
```cpp
struct COMPILER_GENERATED_NAME {
    COMPILER_GENERATED_NAME(T1 Capture1, T2 Capture2, /* … */, TN CaptureN) : _instance1(Capture1), _instance2(Capture2), /* … */, instanceN(CaptureN) {}

    ReturnType operator()(Parameter1, Parameter2, /* … */, ParameterN) const
    {
        Code
    }

    T1 instance1;
    T2 instance2;
    /* … */
    TN instanceN;
};
```

The `struct Functor` created earlier can be defined much more concisely with a lambda in this way:

```cpp
auto predicate1 = [a2](B* b) -> bool {return b->getA() == a2.get();};
std::copy_if(
    objects.begin(),
    objects.end(),
    std::back_inserter(result),
    predicate1 // This is the predicate
);

// Result : {2, 4}
```

This solution with a lambda expression is perfectly fine and some might prefer sticking to it because it is relatively easy to maintain. For the purpose of this article, there is an equivalent solution using the <a href="https://en.wikipedia.org/wiki/Generic_programming">generic programming</a> paradigm. This is somewhat more complex, so let's first decompose the lambda expression in `predicate1` in its smallest parts.

1. `B`'s `getA()` method is called
1. A comparison using `operator==` is performed on the variable `a` captured by the lambda expression, and with the result of the `getA()` method call

It is simpler to make the second operation generic because the STL already provides a functor that does that job for us : `std::equal_to`. As an example, we could rewrite the lambda expression using this functor:
```cpp
auto predicate2 = [a1](B* b) -> bool {
    return std::equal_to<A*>()(b->getA(), a1.get());
};
std::copy_if(
    objects.begin(),
    objects.end(),
    std::back_inserter(result),
    predicate2 // This is the predicate
);

// Result : {1, 3, 5}
```

When used as a function, the `std::equal_to<A*>`'s signature is `bool(A*, A*)`. It is a function that receives two pointers of `A` and returns a `bool`. The predicate expected by an algorithm is a UnaryFunction, so the expected signature is `bool(B*)`. In order to supply the `std::equal_to` functor as a predicate to `std::copy_if`, both the functor and an instance of A must be wrapped in an enclosing functor. So how can a functor be passed as an argument, and how can it be stored in an instance variable? The answer is `std::function`, which is a generic function wrapper that can wrap any callable object (functions, functors, lambdas, bind expressions, etc.). The `std::function` functor must be templated on the method signature for which it can wrap `Callable` instances like this: `std::function<bool(A*, A*)>`. Here is a functor example that can wrap another functor using `std::function`:
```cpp
struct FunctorWrapper
{
    FunctorWrapper(std::function<bool(A*, A*)> functor, A* a) : functor(functor), a(a) {}

    bool operator()(B* b) const
    {
        return functor(a, b->getA());
    }

public:
    std::function<bool(A*, A*)> functor;
    A* a;
};
```

`FunctorWrapper` can now be used as a predicate for the `std::copy_if` algorithm:
```cpp
std::copy_if(
    objects.begin(),
    objects.end(),
    std::back_inserter(result),
    FunctorWrapper(std::equal_to<A*>(), a2.get()) // This is the predicate
);

// Result : {2, 4}
```

Once again, writing functors is a lot of boilerplate to write, and the STL offers a solution: `std::bind`. This functor factory makes it possible to wrap a Callable instance with specific arguments or with "placeholders".
```cpp
auto equalToA1 = std::bind(
    std::equal_to<A*>(),
    std::placeholders::_1,
    a1.get()
);
```

Here, `std::bind` is called with three arguments because the Callable supplied as the first argument expects two parameters. Thus there are two parameters that can be bound. The first argument, `std::placeholders::_1`, can be understood as the first parameter of the `operator()` method of the functor being created. The second argument, `a1`, is more straightforward: the contained pointer is simply passed along as the second parameter of the wrapped Callable. A new version of the predicate can now be written using this tool:
```cpp
auto predicate3 = [equalToA1](B* b) -> bool {
    return equalToA1(b->getA());
};
std::copy_if(
    objects.begin(),
    objects.end(),
    std::back_inserter(result),
    predicate3 // This is the predicate
);

// Result : {1, 3, 5}
```

Now there is one missing piece in our puzzle. A functor with the signature `bool(A*)` was successfully created, but `std::count_if` expects a predicate with the signature `bool(B*)`. Up to now, a lambda expression was used to call `B::getA()` before passing the result to `predicate3`. The next step is to get rid of the lambda expression altogether.

In order to do so, a functor with the signature `A*(B*)` that does its job by calling `B::getA()` has to be created:

```cpp
struct MethodWrapper
{
    MethodWrapper(A*(B::*method)()) : method(method) {}

    A* operator()(B* instance) const
    {
        return (instance->*method)();
    }

    A*(B::*method)();
};
```

This last piece of code has a pretty weird syntax. `A*(B::*method)()` is a pointer to member of `B` named `method` that returns `A*`. In order to use this pointer to achieve a method call, the ->\* operator is used to bind `instance` as `this` within `B::getA()`. Then, the result of this operator must absolutely be enclosed within parentheses before being called. As a last step, another functor is required to compose the calls to two functors.
```cpp
struct ComposingWrapper
{
    ComposingWrapper(std::function<bool(A*)> f, std::function<A*(B*)> g) : f(f), g(g) {}

    bool operator()(B* b)
    {
        return f(g(b));
    }
public:
    std::function<bool(A*)> f;
    std::function<A*(B*)> g;
};
```

```cpp
std::copy_if(
    objects.begin(),
    objects.end(),
    std::back_inserter(result),
    ComposingWrapper(equalToA1, MethodWrapper(&B::getA)) // This is the predicate
);

// Result : {1, 3, 5}
```

When instantiating `ComposingWrapper`, a pointer to member of class `B` is obtained with the syntax `&B::getA`. The `operator&` here has the same meaning as it usually does: it returns the address of its operand.

As you may have guessed at this point, it is not necessary to write functors manually. There are two more things `std::bind` has to offer for this particular problem.

First, `std::bind`'s first parameter can be a pointer to member of class. Then, in addition to that method's parameters, an additional parameter representing `this` will need to be bound. Accordingly, a method without any parameter has a single parameter to bind : `this`.

```cpp
std::bind(
    &B::getA,
    std::placeholders::_1
);
```

Second, the result of calling `std::bind` can be used as an argument for another call to `std::bind` which makes it possible to make function composition.
```cpp
auto predicate4 = std::bind(
    std::equal_to<A*>(),
    std::bind(
        &B::getA,
        std::placeholders::_1
    ),
    a2.get()
);
```

The outermost functor will be built by taking account of the innermost functor which requires a single argument of type `B*`. Thus the outermost functor will also require a single argument of type `B*`. A functor with the signature `bool(B*)` has been successfully created.
```cpp
std::copy_if(
    objects.begin(),
    objects.end(),
    std::back_inserter(result),
    predicate4 // This is the predicate
);

// Result : {2, 4}
```

There is one gotcha that will inevitably happen in a real codebase. Here is a new definition for `class B` where a subtle modification will make it harder to compose our functions:
```cpp
class B
{
public:
    B(int i, A* a) : i(i), a(a) {}

    A* getA() {return a;}
    A* getA() const {return a;}

public:
    int i;
    A* a;
};
```

Now the definition of `predicate4` results in the error: "No matching function for call to 'bind'". Secondary messages read: "Candidate template ignored: couldn't infer template argument '\_Fp'" and "Candidate template ignored: couldn't infer template argument '\_Rp'". It turns out that adding a `const` overload of our function has created an ambiguity in the call to `std::bind` and the compiler is giving us cryptic error messages. In order to disambiguate the bind expression, we need a `static_cast`:
```cpp
auto predicate4 = std::bind(
    std::equal_to<A*>(),
    std::bind(
        static_cast<A*(B::*)()>(&B::getA),
        std::placeholders::_1
    ),
    a
);

std::copy_if(
    objects.begin(),
    objects.end(),
    std::back_inserter(result),
    predicate4 // This is the predicate
);
```

An important note about using `std::bind` is that function composition cannot be accomplished with any Callable (i. e.: `std::function`). Function composition happens if `std::is_bind_expression<T>::value == true` for one of the arguments supplied to `std::bind`. This presumably means it would be possible to specialize a few templates so that a custom function wrapper can be used in function composition, but this possibility won't be explored in this article.

## Conclusion
Four different predicate creation techniques for standard algorithms were presented in this article:

* Function
* Hand-written Functor
* Lambda
* std::bind generated Functor

For any new C++11 code, functions should be avoided as they lack flexibility because they do not hold any state. As for functors, unless there are other requirements beside the bool operator(T) const method, then lambdas are always preferable as they reduce the boilerplate and make the process less error-prone. Finally, this article doesn't have any advice on whether to prefer lambdas over std::bind or the opposite, both seem equally capable.

The complete code for this article can be found [over here](https://github.com/Dalzhim/ArticlePredicateTechniques).
