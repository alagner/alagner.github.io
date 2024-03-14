# Passing arguments by move

## Or rather "passink": a few words on sink arguments

### Introduction

Move semantics is one of the most prominent features of modern C++.
Undoubtedly, it enables improvements in terms of performance as well as expressive ways of stating and transferring ownership of an object without resorting to `std::auto_ptr`-like trickery.

The important part of move semantics is the function template `std::move`. Its name is not necessarily the most suitable one: it does not move anything. In fact, all it does is cast its argument to a reference to an *rvalue*, effectively making the object act as if it were a temporary variable. In other words, `std::move` could (and probably should) have been given a name of `rvalue_cast` so it would describe its behaviour explicitly.
This allows the *move constructor* or *move assigment* to be called and effectively consume the object being moved. Therefore, the result of `std::move`is merely a reference, and it remains a reference until the actual move construction or move assignment happens.

This leads to various consequences; some less obvious than others.

### The usual boring example

There seems to be a wide consensus among C++ developers about passing arguments by a reference to rvalue; they are generally considered *sink arguments*.  An object passed to a function implementing such an interface should be considered *moved from* (whatever that moved from state may be, e.g. empty or *valid, but unspecified*) after the function call, e.g.:

```c++
struct DataType { /*whatever*/ };


void funcRvalueRef(DataType&&);
/* ....
* many lines of code later
*/

DataType d = computeMyData();
funcRvalueRef(std::move(d));
// here it is reasonable to assume that d was moved from;
```
Note that a sink argument does not necessarily have to be an rvalue reference; in some situations passing by value is also a viable option.
This applies (most notably) to non-polymorphic types, as well as a variety of types implementing value semantics. `std::move` is not required at the call site, unless a move-only type is handled. Therefore, the programmer can treat the function as an argument sink if they desire to, but they are not limited to such usage.
```c++
void funcValue(DataType);
/* ....
* many lines of code later
*/
DataType d = computeMyData();
funcValue(std::move(d)); //use as sink
// here it is also reasonable to assume that d was moved from;

DataType e = computeMyData();
funcValue(e); // copy for copyable types or compilation error move-only types
// either way: explicit move needed
```

### The controversial simplicity

When performance is not a critical factor, one could argue that apart from some specific cases the relevance of the above considerations is mostly negligible. They could be considered a matter of style, as move construction is by definition supposed to be relatively cheap. Of course,  moving a struct of several dozens of vectors multiple layers down the stack usually means reassigning multiple pointers per each stack frame. Rvalue reference would probably use a single pointer assignment in terms of which the reference is most likely implemented.
The decision to use one way over the other is left to programmer's own judgement (or profiler-based measurements).

Also, compilation time might be affected, as passing references is possible in case of incomplete types; hence, lower number of header files can be used. This is a thing to consider in some cases although other factors might outweigh it depending on the exact context.

One reason for passing arguments by value can be code brevity.
A single function can replace a *sink* and *non-sink* overload, making the caller responsible for moving the argument.

Consider:
```c++
class Person
{
public:
    explicit Person(std::string&& theName)
    : name(std::move(theName))
    {}
    explicit Person(const std::string& theName)
    : name(theName)
    {}
private:
    std::string name;
};

Person alex{"alex"s}; // ok, temporary, string&&

std::string mike{"thompson"s};
Person mikeThompson{mike}; // ok, copy, const string&

std::string john{"smith"s};
Person johnsmith{std::move(john}; // expclit move, still relatively cheap, string&&
```
versus:
```c++
class Person
{
public:
    explicit Person(std::string theName)
    : name(std::move(theName))
    {}
private:
    std::string name;
};

Person alex{"alex"s}; // ok, temporary

std::string mike{"thompson"s};
Person mikeThompson{mike}; // ok, copy

std::string john{"smith"s};
Person johnsmith{std::move(john)}; // expclit move, still relatively cheap
```
Note how the usage of value semantics allowed to reduce boilerplate. This prevents the usage of polymorphic arguments, but it is not always an issue.

An attentive reader would probably notice the lack of *forwarding referenes*. They introduce yet another possibility to reduce code duplication and make it polymorphism-friendly at the same time. Nonetheless, they enforce the usage of function templates which can be troublesome at times.

Consider:
```c++
class Person
{
public:
    template <typename T>
    explicit Person(T&& theName)
    : name(std::forward<T>(theName))
    { }
private:
    std::string name;
};

int main()
{
    Person me{"alex"};
    //Person myClone{me}; //oops, this fails to compile!
}
```
What has happened here? The non-const lvalue Person object matches the signature of the constructor template. The function template has been deduced to `Person::Person<Person&>(Person& &&)`, effectively becoming `Person::Person<Person&>(Person&)` thanks to *reference collapsing*. The compiler is then presented a better match for the argument than the copy constructor `Person::Person(const Person&)`.
It the typical workaround is constraining the template e.g. using SFINAE.

This is just one of many caveats related to object initialization using *forwarding references*.
Describing them exceeds the scope of this article. An excellent talk from CPPCON 2018 [The Nightmare of Initialization in C++](https://www.youtube.com/watch?v=7DTlWPgX6zs) by Nikolai Josuttis contains further information on this topic.

### The wicked ways of the code

For the sake of an example, let us make a certain assumption about the implementation of functions:  `funcRvalueRef` and `funcValue` from one of the previous chapters: **both of them ignore their arguments completely**. To keep it extremely simple, have those two functions do completely nothing.

```c++
struct DataType { std::vector<int> v{1,2,3}; };
void funcRvalueRef(DataType&&) { /*NO-OP*/ }
void funcValue(DataType) { /*NO-OP*/ }
```
Provided the caller knows the function `void funcRvalueRef(DataType&&) ` does not consume the reference, they may consider the argument intact:

```c++
DataType d = computeMyData();
assert(!d.v.empty());
funcRvalueRef(std::move(d)); // sink argument... or is it?
assert(!d.v.empty()); // it is fine, as we know the implementation
```

For `void funcValue(DataType);` this assumption cannot be made:
in case of using `std::move` the moved argument is consumed at call site, i.e. the the function argument is move-constructed the moment the function is called.

```c++
DataType d = computeMyData();
assert(!d.v.empty());
funcValue(std::move(d)); // this still consumes the argument
assert(d.v.empty());
```

### The exceptional situation

One could raise a question about how often a non-consuming function taking rvalue reference can be encountered.
There are of course some very specific situations, e.g.:
```c++
template <typename...Args>
void suppressUnusedVariableWarning(Args&&...){}
```
but this is not a piece of code written on a regular basis and can probably be replaced by some other means, e.g. cast to `(void)`, `[[maybe_unused]]` attribute or an assignment to `std::ignore` .

Of course, one could specify an interface of this kind:
```c++
bool sendUartNonBlocking(Buffer&& buf); // true on success
// if failed, input is not consumed
```
implemented in the following fashion:
```c++
bool sendUartNonBlocking(Buffer&& buf)
{
    if (sender.wouldBlock()) {
        return false;
    }
    sender.send(std::move(buf));
    return true;
}
```
but it would likely violate the principle of least astonishment, or it is at best highly uncommon.

However, what if, in some specific cases, a function could do something not clearly specified by its interface? Or rather: something so infrequent and so broad in specification, that tends to be overlooked on a regular basis, despite it being visible in the fuction declaration.
I believe the play on words in the title of the chapter has already been noticed by now:
**exceptions**.

Consider the following code:

```c++
void refThrow(DataType&& d, bool t)
{
    if (t) {
        throw std::runtime_error("oops");
    }
    auto x = std::move(d);
    //do sth with x
}

//later in the code:

DataType d = computeMyData();
assert(!d.v.empty());
try {
    refThrow(std::move(d), true);
} catch (const std::runtime_error&) {
    assert(!d.v.empty()); // the argument remains intact when exception was thrown
    // the function maintains the strong exception guarantee towards its argument
}
```
whereas when passing by value:
```c++
void valThrow(DataType d, bool t)
{
    if (t) {
        throw std::runtime_error("oops");
    }
    auto x = std::move(d);
    //do sth with x
}

//later in the code:

DataType d = computeMyData();
assert(!d.v.empty());
try {
    valThrow(std::move(d), true);
} catch (const std::runtime_error&) {
    assert(d.v.empty());// the argument has already been consumed
}
```
This clearly indicates that rvalue references should be considered when designing
functions with *strong exception guarantee*.

### The inconclusive conclusion

The question of passing sink arguments by value or by reference remains open.
Surely, for polymorphic types, references are the only viable option.

For non-polymorphic types, one has to consider exception safety. Should it be unimpotant (oftentimes it is), one can consider passing by value for the sake of simplicity and brevity. That of course on condition those extra moves be not overly costly.
However, should exception safety have any relevance, the rvalue references seem the mechanism to pursue. Probably followed by silent sighs and complaints about `noexcept` being opt-in.


