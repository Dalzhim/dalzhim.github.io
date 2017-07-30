Memory corruption is the symptom of a program that writes into the wrong memory location. There are a few different classes of bugs that can lead to memory corruption such as uninitialized memory, dangling pointers and buffer overflows. Recently, I've discovered a variety I had never seen before, which I'd consider a subclass of buffer overflows. Here's the short story.

I've been asked to help to debug unstability with a piece of software for which there were crash reports coming in and for which we could reproduce the crash quite reliably, but never in the same way. Every crash would occur at some different location in the code in a seemingly random manner. When facing such a situation, memory corruption is an excellent hypothesis to start working with because corruption will lead to undefined behavior and this implies non-deterministic behavior.

There are excellent tools out there to detect many types of memory management mistakes. As far as I am concerned, clang's Address Sanitizer is my favorite tool. One of its drawbacks is that I have not found a way to use it to instrument a plugin that is dynamically loaded within software that is not instrumented. Without the Address Sanitizer, I decided to resort to a simple binary search to hunt for the bug.

With the knowledge of a nearly-deterministic procedure to reproduce the crash, I could already isolate a subset of the code where the bug had to be lurking. By commenting out half of the code, I was able to prove that the execution of one half was absolutely necessary to reproduce the bug, while the other half had absolutely no impact. By repeating this process, I eventually found my culprit within a <em>cpp</em> file :

```cpp
MyObject* obj = new MyObject;
```

<p>Obviously, upon discovering that, I had a look at the constructor, which was within an objective c implementation file (<em>*.mm</em>) to see what was happening :</p>
```cpp
MyObject::MyObject() : myVariable1(…), myVariable2(…) {}
```

<p>This is confusing. This constructor doesn't stand out in any way. Could the problem be somewhere else? Why then is the bug getting solved when I comment out the allocation? It is only when I finally had a look at the header file that I could spot the culprit :</p>
```cpp
class MyObject
{
    int myVariable1;
    #ifdef __objc__
    int myVariable2;
    #endif
public:
    MyObject();
}
```

<p>Here's the deal. This header file describes two different classes. The first one, when compiling without objective c support, has a single variable within MyObject. The second one, when compiling with objective c support, has two variables. This means that sizeof(MyObject) from within a cpp file is not equal to `sizeof(int)` from within an objective c file.</p>
<p>Now the reason the application crashes randomly is that calling new from within a cpp file allocates a smaller amount of memory than what is really required. When the objective c code tries to initialize the object, it accesses memory that is out of bounds, overwriting the data of some undefined data structure, resulting in weird and random crashes.</p>
<p>The two distinct definitions for <code>class MyObject</code> violate the <a href="https://en.wikipedia.org/wiki/One_Definition_Rule">One Definition Rule</a> which states that a single program should only have a single definition for every object. In order to understand what what the One Definition Rule means, one has to understand the differences between declarations and definitions.</p>

In C++, a declaration announces that something exists by naming it. A definition does the work of fully describing what that something is. As far as classes, methods and functions are concerned, the declaration precedes the curly brackets {}, and the definition consists in those curly brackets and everything between each of them. Of course, this is a simplification and [here is an article with more details](https://msdn.microsoft.com/en-us/library/0e5kx78b.aspx).

<p>Now that we have located and understood the source of the bug, how can we solve it? Well in this case, the variable within the #if preprocessor instruction was not really an int. It was an objective c type. MyObject was designed as a class that could be used by cross-platform code in order to do platform specific work. There are many ways to create cross platform classes with platform specific implementations. Two easy designs that can be considered are the <a href="http://www.drdobbs.com/cpp/cross-platform-design-strategies/184410963">abstract base class</a> and the <a href="http://stackoverflow.com/questions/1884316/cross-platform-oop-in-c">detail implementation</a>. Of course there are others, but this is a good start.</p>
<p>Finally, I was really bothered by the impossibility of using the Address Sanitizer so I spended a lot of time trying to figure out a way to use it. When I decided to admit failure, I had the idea of creating a simulator for the application over which I have no control. Eight hours later, I had a basic simulator instrumented with the Address Sanitizer and it became possible to try and detect other memory management problems much more efficiently.</p>
