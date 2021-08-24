# M.6 — std::unique_ptr

At the beginning of the chapter, we discussed how use of pointers can lead to bugs and memory leaks in some situations. For example, this can happen when a function early returns, or throws an exception, and the pointer is not properly deleted.

在本章开始的时候，我们讨论了指针可能造成的bug和内存泄露问题。当函数提前返回或者抛出了一个异常或者指针没有被合理的删除（实际的代码情况会比这复杂）等等。

```C++
#include <iostream>
 
void someFunction()
{
    auto *ptr{ new Resource() };
 
    int x{};
    std::cout << "Enter an integer: ";
    std::cin >> x;
 
    if (x == 0)
        throw 0; // the function returns early, and ptr won’t be deleted!
 
    // do stuff with ptr here
 
    delete ptr;
}
```

Now that we’ve covered the fundamentals of move semantics, we can return to the topic of smart pointer classes. As a reminder, a smart pointer is a class that manages a dynamically allocated object. Although smart pointers can offer other features, the defining characteristic of a smart pointer is that it manages a dynamically allocated resource, and ensures the dynamically allocated object is properly cleaned up at the appropriate time (usually when the smart pointer goes out of scope).

现在我们已经讲了移动语义，我们可以回到只能指针的话题了。提示一下，一个智能指针是用来管理一个动态分配的对象的类。尽管智能指针还可以提供额外功能，智能指针的定义特征是它管理一个动态分配的资源，确保动态分配的那个对象能够在合适的时间被合适的清理掉。

Because of this, smart pointers should never be dynamically allocated themselves (otherwise, there is the risk that the smart pointer may not be properly deallocated, which means the object it owns would not be deallocated, causing a memory leak). By always allocating smart pointers on the stack (as local variables or composition members of a class), we’re guaranteed that the smart pointer will properly go out of scope when the function or object it is contained within ends, ensuring the object the smart pointer owns is properly deallocated.

因此，智能指针不应该被动态分配（否则的话，就有智能指针不被正常析构的可能，也意味着它拥有的对象没有被合理的清除，导致内存泄露问题）。通过总是把智能指针分配在栈上（作为一个本地变量或者是作为一个组合类的成员）。我们确保智能指针在死亡的时候，能够合理的被析构。

C++11 standard library ships with 4 smart pointer classes: std::auto_ptr (which you shouldn’t use -- it’s being removed in C++17), std::unique_ptr, std::shared_ptr, and std::weak_ptr. std::unique_ptr is by far the most used smart pointer class, so we’ll cover that one first. In the next lessons, we’ll cover std::shared_ptr and std::weak_ptr.

C++11提供了4种只能指针，autoptr（你不应该用它），uniqueptr，sharedptr，weakptr。uniqueptr是使用最多的智能指针类。所以我们会先讲这个指针类。在后面的课程，我们也会探讨sharedptr和weakptr。

### std::unique_ptr 独占资源的智能指针

std::unique_ptr is the C++11 replacement for std::auto_ptr. It should be used to manage any dynamically allocated object that is not shared by multiple objects. That is, std::unique_ptr should completely own the object it manages, not share that ownership with other classes. std::unique_ptr lives in the <memory> header.

uniqueptr是C++对autoptr的替代。它应该被用来管理任何动态分配的内存对象，只要这个对象不被多个对象所共有。这意味着uniqueptr应该完全拥有它管理的那个内存对象，而不是和别人共享这种所有权。uniqueptr在memory头文件中。

Let’s take a look at a simple smart pointer example:

我们看一个简单的智能指针的例子：

```
#include <iostream>
#include <memory> // for std::unique_ptr
 
class Resource
{
public:
	Resource() { std::cout << "Resource acquired\n"; }
	~Resource() { std::cout << "Resource destroyed\n"; }
};
 
int main()
{
	// allocate a Resource object and have it owned by std::unique_ptr
	std::unique_ptr<Resource> res{ new Resource() };
 
	return 0;
} // res goes out of scope here, and the allocated Resource is destroyed
```

Because the std::unique_ptr is allocated on the stack here, it’s guaranteed to eventually go out of scope, and when it does, it will delete the Resource it is managing.

因为uniqueptr是在栈上分配的，所以它最后肯定会死亡，当他死亡的时候，它管理的资源也会被释放。

Unlike std::auto_ptr, std::unique_ptr properly implements move semantics.

不像autoptr，uniqueptr可以合理的实现移动语义。

```
#include <iostream>
#include <memory> // for std::unique_ptr
 
class Resource
{
public:
	Resource() { std::cout << "Resource acquired\n"; }
	~Resource() { std::cout << "Resource destroyed\n"; }
};
 
int main()
{
	std::unique_ptr<Resource> res1{ new Resource{} }; // Resource created here
	std::unique_ptr<Resource> res2{}; // Start as nullptr
 
	std::cout << "res1 is " << (static_cast<bool>(res1) ? "not null\n" : "null\n");
	std::cout << "res2 is " << (static_cast<bool>(res2) ? "not null\n" : "null\n");
 
	// res2 = res1; // Won't compile: copy assignment is disabled
	res2 = std::move(res1); // res2 assumes ownership, res1 is set to null
 
	std::cout << "Ownership transferred\n";
 
	std::cout << "res1 is " << (static_cast<bool>(res1) ? "not null\n" : "null\n");
	std::cout << "res2 is " << (static_cast<bool>(res2) ? "not null\n" : "null\n");
 
	return 0;
} // Resource destroyed here when res2 goes out of scope
```

This prints:

```
Resource acquired
res1 is not null
res2 is null
Ownership transferred
res1 is null
res2 is not null
Resource destroyed
```

Because std::unique_ptr is designed with move semantics in mind, copy initialization and copy assignment are disabled. If you want to transfer the contents managed by std::unique_ptr, you must use move semantics. In the program above, we accomplish this via std::move (which converts res1 into an r-value, which triggers a move assignment instead of a copy assignment).

因为uniqueptr是设计了移动语义的，拷贝构造和拷贝赋值是禁用的。如果你想要转移它管理的内容，你应该使用移动语义，在上面的程序中，我们通过stdmove（把左值变成了右值引用，然后触发了移动赋值运算）完成了这件事情。

### Accessing the managed object 访问智能指针管理的对象

std::unique_ptr has an overloaded operator* and operator-> that can be used to return the resource being managed. Operator* returns a reference to the managed resource, and operator-> returns a pointer.

uniqueptr有一个重载的运算符*和->，可以用来返回被管理的对象。`*`返回一个对管理对象的引用。operator`->`返回一个指针。

Remember that std::unique_ptr may not always be managing an object -- either because it was created empty (using the default constructor or passing in a nullptr as the parameter), or because the resource it was managing got moved to another std::unique_ptr. So before we use either of these operators, we should check whether the std::unique_ptr actually has a resource. Fortunately, this is easy: std::unique_ptr has a cast to bool that returns true if the std::unique_ptr is managing a resource.

记住，uniqueptr不一定管理一个对象，如果它创建了一个空的，或者因为它管理的资源被转移出去了，都有可能。所以我们在使用这些操作符之前，我们应该检查一下这个uniqueptr是否真的拥有一个资源。幸运的是，这很简单，uniqueptr有一个类型转换运算符重载能够转换成bool，如果它有资源的话，就返回true，否则false。

Here’s an example of this:

```
#include <iostream>
#include <memory> // for std::unique_ptr
 
class Resource
{
public:
	Resource() { std::cout << "Resource acquired\n"; }
	~Resource() { std::cout << "Resource destroyed\n"; }
	friend std::ostream& operator<<(std::ostream& out, const Resource &res)
	{
		out << "I am a resource\n";
		return out;
	}
};
 
int main()
{
	std::unique_ptr<Resource> res{ new Resource{} };
 
	if (res) // use implicit cast to bool to ensure res contains a Resource
		std::cout << *res << '\n'; // print the Resource that res is owning
 
	return 0;
}
```

This prints:

```
Resource acquired
I am a resource
Resource destroyed
```

In the above program, we use the overloaded operator* to get the Resource object owned by std::unique_ptr res, which we then send to std::cout for printing.

在上面的程序中，我们使用重载的`*`来获得被管理的资源对象，然后把它打印给输出。

#### std::unique_ptr and arrays 智能指针和数组的比较

Unlike std::auto_ptr, std::unique_ptr is smart enough to know whether to use scalar delete or array delete, so std::unique_ptr is okay to use with both scalar objects and arrays.

不像autoptr，uniqueptr是足够聪明的知道，到底要使用数组的delete还是使用普通的delete。所以uniqueptr管理数组和管理单个对象是没问题。

However, std::array or std::vector (or std::string) are almost always better choices than using std::unique_ptr with a fixed array, dynamic array, or C-style string.

然而，array和vector比管理了一个固定数组或者动态数组或者C风格字符串的uniqueptr应该更优先被使用。

***Rule: Favor std::array, std::vector, or std::string over a smart pointer managing a fixed array, dynamic array, or C-style string***

### **std::make_unique** 函数

C++14 comes with an additional function named std::make_unique(). This templated function constructs an object of the template type and initializes it with the arguments passed into the function.

C++14有一个新的函数叫做make_unique，这个模板函数构造一个模板类型的对象，而且用传入的参数初始化它。

```
#include <memory> // for std::unique_ptr and std::make_unique
#include <iostream>
 
class Fraction
{
private:
	int m_numerator{ 0 };
	int m_denominator{ 1 };
 
public:
	Fraction(int numerator = 0, int denominator = 1) :
		m_numerator{ numerator }, m_denominator{ denominator }
	{
	}
 
	friend std::ostream& operator<<(std::ostream& out, const Fraction &f1)
	{
		out << f1.m_numerator << '/' << f1.m_denominator;
		return out;
	}
};
 
 
int main()
{
	// Create a single dynamically allocated Fraction with numerator 3 and denominator 5
	// We can also use automatic type deduction to good effect here
	auto f1{ std::make_unique<Fraction>(3, 5) };
	std::cout << *f1 << '\n';
 
	// Create a dynamically allocated array of Fractions of length 4
	auto f2{ std::make_unique<Fraction[]>(4) };
	std::cout << f2[0] << '\n';
 
	return 0;
}
```

The code above prints:

```
3/5
0/1
```

Use of std::make_unique() is optional, but is recommended over creating std::unique_ptr yourself. This is because code using std::make_unique is simpler, and it also requires less typing (when used with automatic type deduction). Furthermore it resolves an exception safety issue that can result from C++ leaving the order of evaluation for function arguments unspecified.

使用make_unique是可选的，但是它是强烈建议的，比你自己创建uniqueptr对象要好。这是因为使用makeunique更简单，而且打的字也更少。而且，他解决了一个异常安全问题，这个问题是因为C++标准没有指定传参的执行顺序。

```
Rule
use std::make_unique() instead of creating std::unique_ptr and using new yourself
```

### **The exception safety issue in more detail**  异常安全问题

For those wondering what the “exception safety issue” mentioned above is, here’s a description of the issue.

对想要知道上面说的异常安全问题，下面给出了一个描述：

Consider an expression like this one:

思考一下下面的问题：

```
some_function(std::unique_ptr<T>(new T), function_that_can_throw_exception());

```

The compiler is given a lot of flexibility in terms of how it handles this call. It could create a new T, then call function_that_can_throw_exception(), then create the std::unique_ptr that manages the dynamically allocated T. If function_that_can_throw_exception() throws an exception, then the T that was allocated will not be deallocated, because the smart pointer to do the deallocation hasn’t been created yet. This leads to T being leaked.

编译器能够决定参数计算的顺序。它可以先创建一个T，然后调用可以抛出异常的函数，再然后才创建uniqueptr来管理前面创建的T。如果那个函数真的抛出一个异常，然后T就不会被析构，因为智能指针还没有被构造呀。这导致T被泄露了。（特别是如果异常被忽略了）

std::make_unique() doesn’t suffer from this problem because the creation of the object T and the creation of the std::unique_ptr happen inside the std::make_unique() function, where there’s no ambiguity about order of execution.

make_unique函数不会有这样的问题，因为对对象T的创建和uniqueptr的创建实在make_unique函数内部发生的，所以这里执行顺序就没有什么问题了。如果先创建智能指针，然后后面的函数抛出异常，那么程序退出，创建的智能指针也因为go out of scope而被销毁，对应的内存对象也被释放。

#### **Returning std::unique_ptr from a function** 从函数中返回unique_ptr

std::unique_ptr can be safely returned from a function by value:

unique_ptr可以安全的从函数中通过值来返回。

```
std::unique_ptr<Resource> createResource()
{
     return std::make_unique<Resource>();
}
 
int main()
{
    auto ptr{ createResource() };
 
    // do whatever
 
    return 0;
}
```

In the above code, createResource() returns a std::unique_ptr by value. If this value is not assigned to anything, the temporary return value will go out of scope and the Resource will be cleaned up. If it is assigned (as shown in main()), in C++14 or earlier, move semantics will be employed to transfer the Resource from the return value to the object assigned to (in the above example, ptr), and in C++17 or newer, the return will be elided. This makes returning a resource by std::unique_ptr much safer than returning raw pointers!

在上面的代码中，createResource以值的方式返回了一个uniqueptr，如果这个value没有被赋值给任何东西，那么这个临时对象就会被清理掉。如果他复制给了什么东西，就像上面那样。在C++14或者之前，移动语义会被部署，从这个返回值中转移资源给这个被赋值的对象。在C++17或者更新，这个返回会被跳过。这使得返回uniqueptr比原始指针还快。

In general, you should not return std::unique_ptr by pointer (ever) or reference (unless you have a specific compelling reason to).

通常来说，你不应以引用或者值的方式来返回一个unique_ptr，你应该用value。

### Passing std::unique_ptr to a function 传递一个unique_ptr给一个函数

If you want the function to take ownership of the contents of the pointer, pass the std::unique_ptr by value. Note that because copy semantics have been disabled, you’ll need to use std::move to actually pass the variable in.

如果你想要给函数转移一个智能指针拥有的对象，你应该以值的方式来传入unique_ptr参数。注意，因为拷贝语义已经被删掉了，所以你应该使用std::move函数来传递一个右值引用进去（这样参数的构造过程用的就是移动构造函数）。

```
#include <memory> // for std::unique_ptr
 
class Resource
{
public:
	Resource() { std::cout << "Resource acquired\n"; }
	~Resource() { std::cout << "Resource destroyed\n"; }
	friend std::ostream& operator<<(std::ostream& out, const Resource &res)
	{
		out << "I am a resource\n";
		return out;
	}
};
 
void takeOwnership(std::unique_ptr<Resource> res)
{
     if (res)
          std::cout << *res << '\n';
} // the Resource is destroyed here
 
int main()
{
    auto ptr{ std::make_unique<Resource>() };
 
//    takeOwnership(ptr); // This doesn't work, need to use move semantics
    takeOwnership(std::move(ptr)); // ok: use move semantics
 
    std::cout << "Ending program\n";
 
    return 0;
}
```

The above program prints:

```
Resource acquired
I am a resource
Resource destroyed
Ending program
```

Note that in this case, ownership of the Resource was transferred to takeOwnership(), so the Resource was destroyed at the end of takeOwnership() rather than the end of main().

注意在这个例子中，所有权已经被移交给takeOwnership函数，所以Resource资源在takeOwnership函数结束的时候被释放掉了。

However, most of the time, you won’t want the function to take ownership of the resource. Although you can pass a std::unique_ptr by reference (which will allow the function to use the object without assuming ownership), you should only do so when the called function might alter or change the object being managed.

然而，**大部分时候，你不想让这个函数拿走这个资源的所有权**，尽管你可以以引用的方式传递这个只能指针（允许这个函数使用这个指针而不需要交换所有权），你应该只在这个被调用的函数可能替换掉这个被管理的对象的情况下才这么做。

Instead, it’s better to just pass the resource itself (by pointer or reference, depending on whether null is a valid argument). This allows the function to remain agnostic of how the caller is managing its resources. To get a raw resource pointer from a std::unique_ptr, you can use the get() member function:

不过，把资源传过去更好（通过指针或者通过引用，取决于null是不是一个合理的参数）。这使得被调用的函数不能够知道调用者是怎么管理这个资源的（屏蔽）。要得到资源的原始指针，你应该使用get成员函数来得到原始指针。

```
#include <memory> // for std::unique_ptr
#include <iostream>
 
class Resource
{
public:
	Resource() { std::cout << "Resource acquired\n"; }
	~Resource() { std::cout << "Resource destroyed\n"; }
 
	friend std::ostream& operator<<(std::ostream& out, const Resource &res)
	{
		out << "I am a resource\n";
		return out;
	}
};
 
// The function only uses the resource, so we'll accept a pointer to the resource, not a reference to the whole std::unique_ptr<Resource>
void useResource(Resource *res)
{
	if (res)
		std::cout << *res << '\n';
}
 
int main()
{
	auto ptr{ std::make_unique<Resource>() };
 
	useResource(ptr.get()); // note: get() used here to get a pointer to the Resource
 
	std::cout << "Ending program\n";
 
	return 0;
} // The Resource is destroyed here
```

The above program prints:

```
Resource acquired
I am a resource
Ending program
Resource destroyed
```

### std::unique_ptr and classes 智能指针作为类的成员

You can, of course, use std::unique_ptr as a composition member of your class. This way, you don’t have to worry about ensuring your class destructor deletes the dynamic memory, as the std::unique_ptr will be automatically destroyed when the class object is destroyed. However, do note that if your class object is dynamically allocated, the object itself is at risk for not being properly deallocated, in which case even a smart pointer won’t help.

你当然也可以把uniqueptr作为你的组合类的成员。这样的话，你就不需要在析构函数里面操心动态内存释放的事情了，这个unique_ptr成员会在你的类的实例死亡的时候，同时死亡。然而，需要注意的是，如果你的类对象是动态分配的，那么这个对象就有可能不被合理的析构，在这种情况下，就算是智能指针，也帮不了你的忙 ：）

### Misusing std::unique_ptr 对智能指针的误用

There are two easy ways to misuse std::unique_ptrs, both of which are easily avoided. First, don’t let multiple classes manage the same resource. For example:

有两种很容易就把智能指针用错的情况。这两种都可以被轻松的避免，首先不要让两个对象管理同一个资源。

```
Resource *res{ new Resource() };
std::unique_ptr<Resource> res1{ res };
std::unique_ptr<Resource> res2{ res };
```

While this is legal syntactically, the end result will be that both res1 and res2 will try to delete the Resource, which will lead to undefined behavior.

虽然这在语法上是合法的，但是结果就是res1和res2都会试图删除这个资源，这样的结果是未定义的。

Second, don’t manually delete the resource out from underneath the std::unique_ptr.

第二，不要手动删除unique_ptr管理的那个资源。

```
Resource *res{ new Resource() };
std::unique_ptr<Resource> res1{ res };
delete res;
```

If you do, the std::unique_ptr will try to delete an already deleted resource, again leading to undefined behavior.

如果你这样做的话，unique_ptr会尝试删除一个已经删除掉的资源，这样也会导致一个未定义行为。

Note that std::make_unique() prevents both of the above cases from happening inadvertently.

**注意，std::make_unique可以阻止上面两种情况无意的发生。**