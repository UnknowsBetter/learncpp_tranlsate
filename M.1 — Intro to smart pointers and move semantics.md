# M.1 — Intro to smart pointers and move semantics

Consider a function in which we dynamically allocate a value:

思考下面的例子

```
void someFunction()
{
    Resource *ptr = new Resource(); // Resource is a struct or class
 
    // do stuff with ptr here
 
    delete ptr;
}
```

Although the above code seems fairly straightforward, it’s fairly easy to forget to deallocate ptr. Even if you do remember to delete ptr at the end of the function, there are a myriad of ways that ptr may not be deleted if the function exits early. This can happen via an early return:

上面的例子简单。很容易就忘记去释放内存。即便你能记住要释放，但是也可能因为别的情况而失败。函数提前返回，或者抛出异常。

```
#include <iostream>
 
void someFunction()
{
    Resource *ptr = new Resource();
 
    int x;
    std::cout << "Enter an integer: ";
    std::cin >> x;
 
    if (x == 0)
        return; // the function returns early, and ptr won’t be deleted!
 
    // do stuff with ptr here
 
    delete ptr;
}
```

or via a thrown exception:

```
#include <iostream>
 
void someFunction()
{
    Resource *ptr = new Resource();
 
    int x;
    std::cout << "Enter an integer: ";
    std::cin >> x;
 
    if (x == 0)
        throw 0; // the function returns early, and ptr won’t be deleted!
 
    // do stuff with ptr here
 
    delete ptr;
}
```

In the above two programs, the early return or throw statement execute, causing the function to terminate without variable ptr being deleted. Consequently, the memory allocated for variable ptr is now leaked (and will be leaked again every time this function is called and returns early).

上面的程序中，函数提前返回了，导致内存释放失败。

At heart, these kinds of issues occur because pointer variables have no inherent mechanism to clean up after themselves.

问题的核心在于普通的指针没有在自己死亡时，把自己管理的内存释放掉的机制。

## **Smart pointer classes to the rescue?**

One of the best things about classes is that they contain destructors that automatically get executed when an object of the class goes out of scope. So if you allocate (or acquire) memory in your constructor, you can deallocate it in your destructor, and be guaranteed that the memory will be deallocated when the class object is destroyed (regardless of whether it goes out of scope, gets explicitly deleted, etc…). This is at the heart of the RAII programming paradigm that we talked about in lesson [11.9 -- Destructors](https://www.learncpp.com/cpp-tutorial/destructors/).

类的好处是有一个析构函数可以在对象死亡的时候做一些清理工作。所以如果你在你的构造函数里面分配内存，你可以在析构函数中清理。在类对象死亡的时候，析构函数都会被调用（不管它是因为走出了作用域还是被明确的删除了）。这个是RAII的基本思想。

So can we use a class to help us manage and clean up our pointers? We can!

我们可以写一个类帮助我们清理指针吗？

Consider a class whose sole job was to hold and “own” a pointer passed to it, and then deallocate that pointer when the class object went out of scope. As long as objects of that class were only created as local variables, we could guarantee that the class would properly go out of scope (regardless of when or how our functions terminate) and the owned pointer would get destroyed.

思考一个类，它唯一的工作就是管理一个传给他的指针。只要这个类的对象是局部变量，在它死亡的时候，它的析构函数被调用，管理的内存也能在析构函数中释放。

Here’s a first draft of the idea:

下面是个基本的想法：

```
#include <iostream>
 
template<class T>
class Auto_ptr1
{
	T* m_ptr;
public:
	// Pass in a pointer to "own" via the constructor
	Auto_ptr1(T* ptr=nullptr)
		:m_ptr(ptr)
	{
	}
	
	// The destructor will make sure it gets deallocated
	~Auto_ptr1()
	{
		delete m_ptr;
	}
 
	// Overload dereference and operator-> so we can use Auto_ptr1 like m_ptr.
	T& operator*() const { return *m_ptr; }
	T* operator->() const { return m_ptr; }
};
 
// A sample class to prove the above works
class Resource
{
public:
    Resource() { std::cout << "Resource acquired\n"; }
    ~Resource() { std::cout << "Resource destroyed\n"; }
};
 
int main()
{
	Auto_ptr1<Resource> res(new Resource()); // Note the allocation of memory here
 
        // ... but no explicit delete needed
 
	// Also note that the Resource in angled braces doesn't need a * symbol, since that's supplied by the template
 
	return 0;
} // res goes out of scope here, and destroys the allocated Resource for us
```

This program prints:

```
Resource acquired
Resource destroyed
```

Consider how this program and class work. First, we dynamically create a Resource, and pass it as a parameter to our templated Auto_ptr1 class. From that point forward, our Auto_ptr1 variable res owns that Resource object (Auto_ptr1 has a composition relationship with m_ptr). Because res is declared as a local variable and has block scope, it will go out of scope when the block ends, and be destroyed (no worries about forgetting to deallocate it). And because it is a class, when it is destroyed, the Auto_ptr1 destructor will be called. That destructor will ensure that the Resource pointer it is holding gets deleted!

想一下上面的程序是怎么回事。

As long as Auto_ptr1 is defined as a local variable (with automatic duration, hence the “Auto” part of the class name), the Resource will be guaranteed to be destroyed at the end of the block it is declared in, regardless of how the function terminates (even if it terminates early).

Such a class is called a smart pointer. A **Smart pointer** is a composition class that is designed to manage dynamically allocated memory and ensure that memory gets deleted when the smart pointer object goes out of scope. (Relatedly, built-in pointers are sometimes called “dumb pointers” because they can’t clean up after themselves).

这样的类叫做智能指针，一个智能指针是一个组合类，用来管理动态分配的内存，确保内存能够及时释放不会发生内存泄露。相对的，原生的指针就叫做dumb pointer。

Now let’s go back to our someFunction() example above, and show how a smart pointer class can solve our challenge:

我们再看一下上面的例子：

```
#include <iostream>
 
template<class T>
class Auto_ptr1
{
	T* m_ptr;
public:
	// Pass in a pointer to "own" via the constructor
	Auto_ptr1(T* ptr=nullptr)
		:m_ptr(ptr)
	{
	}
	
	// The destructor will make sure it gets deallocated
	~Auto_ptr1()
	{
		delete m_ptr;
	}
 
	// Overload dereference and operator-> so we can use Auto_ptr1 like m_ptr.
	T& operator*() const { return *m_ptr; }
	T* operator->() const { return m_ptr; }
};
 
// A sample class to prove the above works
class Resource
{
public:
    Resource() { std::cout << "Resource acquired\n"; }
    ~Resource() { std::cout << "Resource destroyed\n"; }
    void sayHi() { std::cout << "Hi!\n"; }
};
 
void someFunction()
{
    Auto_ptr1<Resource> ptr(new Resource()); // ptr now owns the Resource
 
    int x;
    std::cout << "Enter an integer: ";
    std::cin >> x;
 
    if (x == 0)
        return; // the function returns early
 
    // do stuff with ptr here
    ptr->sayHi();
}
 
int main()
{
    someFunction();
 
    return 0;
}
```

If the user enters a non-zero integer, the above program will print:

```
Resource acquired
Hi!
Resource destroyed
```

If the user enters zero, the above program will terminate early, printing:

```
Resource acquired
Resource destroyed
```

Note that even in the case where the user enters zero and the function terminates early, the Resource is still properly deallocated.

即便函数提前终止，分配的内存也能及时释放

Because the ptr variable is a local variable, ptr will be destroyed when the function terminates (regardless of how it terminates). And because the Auto_ptr1 destructor will clean up the Resource, we are assured that the Resource will be properly cleaned up.

**因为ptr是一个局部变量，它会在函数终止的时候被销毁（不管是怎么终止的）。**也因为它在被销毁的时候，会调用析构函数。所以我们就确保内存被合理释放了。

## **A critical flaw**

The Auto_ptr1 class has a critical flaw lurking behind some auto-generated code. Before reading further, see if you can identify what it is. We’ll wait…

但是这个类有一个严重的缺陷。

(Hint: consider what parts of a class get auto-generated if you don’t supply them)

(Jeopardy music)

Okay, time’s up.

Rather than tell you, we’ll show you. Consider the following program:

```
#include <iostream>
 
// Same as above
template<class T>
class Auto_ptr1
{
	T* m_ptr;
public:
	Auto_ptr1(T* ptr=nullptr)
		:m_ptr(ptr)
	{
	}
	
	~Auto_ptr1()
	{
		delete m_ptr;
	}
 
	T& operator*() const { return *m_ptr; }
	T* operator->() const { return m_ptr; }
};
 
class Resource
{
public:
	Resource() { std::cout << "Resource acquired\n"; }
	~Resource() { std::cout << "Resource destroyed\n"; }
};
 
int main()
{
	Auto_ptr1<Resource> res1(new Resource());
	Auto_ptr1<Resource> res2(res1); // Alternatively, don't initialize res2 and then assign res2 = res1;
 
	return 0;
}
```

This program prints:

```
Resource acquired
Resource destroyed
Resource destroyed
```

Very likely (but not necessarily) your program will crash at this point. See the problem now? Because we haven’t supplied a copy constructor or an assignment operator, C++ provides one for us. And the functions it provides do shallow copies. So when we initialize res2 with res1, both Auto_ptr1 variables are pointed at the same Resource. When res2 goes out of the scope, it deletes the resource, leaving res1 with a dangling pointer. When res1 goes to delete its (already deleted) Resource, crash!

非常有可能（但不是一定的）你的程序会崩溃。看见上面的问题了吧？因为我们没有提供拷贝构造函数或者是赋值运算符重载，C++给我们提供了一份浅拷贝的版本。所以当我们用res1初始化res2的时候。两个指针指向了同一个内存空间，而在两个变量死亡的时候第一个死亡的先释放了内存。而第二个指针此时保留了一个悬挂指针，释放一块不属于自己的内存。

You’d run into a similar problem with a function like this:

在函数传参的时候也是这样的：

```
void passByValue(Auto_ptr1<Resource> res)
{
}
 
int main()
{
	Auto_ptr1<Resource> res1(new Resource());
	passByValue(res1)
 
	return 0;
}
```

In this program, res1 will be copied by value into passByValue’s parameter res, leading to duplication of the Resource pointer. Crash!

在这个程序中，res也通过只拷贝来进入参数，然后导致管理的内存在函数中被释放，在main函数里面进行二次释放，这样就崩溃了！

So clearly this isn’t good. How can we address this?

所以这样明显是不好的，怎么解决这个问题呢？

Well, one thing we could do would be to explicitly define and delete the copy constructor and assignment operator, thereby preventing any copies from being made in the first place. That would prevent the pass by value case (which is good, we probably shouldn’t be passing these by value anyway).

我们可以做的一个方法是**清楚的删掉拷贝构造函数和赋值运算符重载**。这样就从一开始阻止了这个事情的发生。能阻止传值的情形（这很好，我们不应该允许这个事情）。

But then how would we return an Auto_ptr1 from a function back to the caller?

但是我们怎么解决从函数内部返回的事情呢？

```
??? generateResource()
{
     Resource *r = new Resource();
     return Auto_ptr1(r);
}
```

We can’t return our Auto_ptr1 by reference, because the local Auto_ptr1 will be destroyed at the end of the function, and the caller will be left with a dangling reference. Return by address has the same problem. We could return pointer r by address, but then we might forget to delete r later, which is the whole point of using smart pointers in the first place. So that’s out. Returning the Auto_ptr1 by value is the only option that makes sense -- but then we end up with shallow copies, duplicated pointers, and crashes.

我们不能通过引用传回，因为本地变量会被销毁，传回的引用也会变成悬挂引用。用地址返回也是一样的。我们可以通过指针传回，但是我们可能忘记会删掉指针（说好的不用指针呢）。我们回到了唯一的选择是通过值来返回。但是我们只是使用了浅拷贝，然后重复的指针，然后崩溃。

Another option would be to override the copy constructor and assignment operator to make deep copies. In this way, we’d at least guarantee to avoid duplicate pointers to the same object. But copying can be expensive (and may not be desirable or even possible), and we don’t want to make needless copies of objects just to return an Auto_ptr1 from a function. Plus assigning or initializing a dumb pointer doesn’t copy the object being pointed to, so why would we expect smart pointers to behave differently?

另一个选项是从写拷贝构造函数和赋值运算符来做深拷贝。我们这样做能避免多个指针指向同意对象的问题，但是拷贝的代价是很大的（可能你不想这样做，也可能这样做不可行），而且我们也不想要做没必要的拷贝，就只是为了从函数里面返回。另外赋值初始化一个普通指针也没有拷贝指向的对象。所以为什么我们想让智能指针表现的和普通指针不一样呢？

What do we do?

我们应该怎么做？

## **Move semantics**

What if, instead of having our copy constructor and assignment operator copy the pointer (“copy semantics”), we instead transfer/move ownership of the pointer from the source to the destination object? This is the core idea behind move semantics. **Move semantics** means the class will transfer ownership of the object rather than making a copy.

要是这样的话，我们不做拷贝，我们做转移，转移所有权。我们把管理权从一个地方转移到另一个地方。

Let’s update our Auto_ptr1 class to show how this can be done:

```
#include <iostream>
 
template<class T>
class Auto_ptr2
{
	T* m_ptr;
public:
	Auto_ptr2(T* ptr=nullptr)
		:m_ptr(ptr)
	{
	}
	
	~Auto_ptr2()
	{
		delete m_ptr;
	}
 
	// A copy constructor that implements move semantics
	Auto_ptr2(Auto_ptr2& a) // note: not const
	{
		m_ptr = a.m_ptr; // transfer our dumb pointer from the source to our local object
		a.m_ptr = nullptr; // make sure the source no longer owns the pointer
	}
	
	// An assignment operator that implements move semantics
	Auto_ptr2& operator=(Auto_ptr2& a) // note: not const
	{
		if (&a == this)
			return *this;
 
		delete m_ptr; // make sure we deallocate any pointer the destination is already holding first
		m_ptr = a.m_ptr; // then transfer our dumb pointer from the source to the local object
		a.m_ptr = nullptr; // make sure the source no longer owns the pointer
		return *this;
	}
 
	T& operator*() const { return *m_ptr; }
	T* operator->() const { return m_ptr; }
	bool isNull() const { return m_ptr == nullptr;  }
};
 
class Resource
{
public:
	Resource() { std::cout << "Resource acquired\n"; }
	~Resource() { std::cout << "Resource destroyed\n"; }
};
 
int main()
{
	Auto_ptr2<Resource> res1(new Resource());
	Auto_ptr2<Resource> res2; // Start as nullptr
 
	std::cout << "res1 is " << (res1.isNull() ? "null\n" : "not null\n");
	std::cout << "res2 is " << (res2.isNull() ? "null\n" : "not null\n");
 
	res2 = res1; // res2 assumes ownership, res1 is set to null
 
	std::cout << "Ownership transferred\n";
 
	std::cout << "res1 is " << (res1.isNull() ? "null\n" : "not null\n");
	std::cout << "res2 is " << (res2.isNull() ? "null\n" : "not null\n");
 
	return 0;
}
```

This program prints:

```
Resource acquired
res1 is not null
res2 is null
Ownership transferred
res1 is null
res2 is not null
Resource destroyed
```

Note that our overloaded operator= gave ownership of m_ptr from res1 to res2! Consequently, we don’t end up with duplicate copies of the pointer, and everything gets tidily cleaned up.

注意看我们重载了的赋值运算符，所以我们没有多个指针指向同意对象的情况。活干的很漂亮

## **std::auto_ptr, and why to avoid it**

Now would be an appropriate time to talk about std::auto_ptr. std::auto_ptr, introduced in C++98, was C++’s first attempt at a standardized smart pointer. std::auto_ptr opted to implement move semantics just like the Auto_ptr2 class does.

现在是合适的时间来讲一讲auto_ptr了。这个auto_ptr是在C++98的时候引入的。auto_ptr实现了移动语义，就像我们上面写的Auto_ptr2 做的事情一样。

However, std::auto_ptr (and our Auto_ptr2 class) has a number of problems that makes using it dangerous.

然而auto_ptr有一堆的问题使得它变得非常危险。

First, because std::auto_ptr implements move semantics through the copy constructor and assignment operator, passing a std::auto_ptr by value to a function will cause your resource to get moved to the function parameter (and be destroyed at the end of the function when the function parameters go out of scope). Then when you go to access your auto_ptr argument from the caller (not realizing it was transferred and deleted), you’re suddenly dereferencing a null pointer. Crash!

首先，移动语义是通过拷贝构造和复制运算实现的。将auto_ptr通过值传递给function的话，会导致你管理的资源因为参数的生命期结束销毁而释放。所以之后你再使用你的auto_ptr（你没有意识到她现在已经是空指针了），你其实是在解引用一个空指针，崩溃！

Second, std::auto_ptr always deletes its contents using non-array delete. This means auto_ptr won’t work correctly with dynamically allocated arrays, because it uses the wrong kind of deallocation. Worse, it won’t prevent you from passing it a dynamic array, which it will then mismanage, leading to memory leaks.

第二，auto_ptr 总是用非数组的方法删除它管理的对象。这意味着auto_ptr 不会和动态分配的数组很好的适应。因为它使用错误的析构方法，她也不会阻止你传一个动态数组给它，然后它管理出问题，就导致内存泄露。

Finally, auto_ptr doesn’t play nice with a lot of the other classes in the standard library, including most of the containers and algorithms. This occurs because those standard library classes assume that when they copy an item, it actually makes a copy, not a move.

最后，auto_ptr 也没有和标准库中的其他类适应的很好，包括大部分容器类和算法类。这是因为标准库中的类假定他们拷贝一个东西的时候，是真的做了一份拷贝，而不是进行移动。

Because of the above mentioned shortcomings, std::auto_ptr has been deprecated in C++11, and it should not be used. In fact, std::auto_ptr is slated for complete removal from the standard library as part of C++17!

因为上面说到的这些问题，auto_ptr 已经在C++11中被不建议使用。事实上，在C++17中，已经被完全删掉了。

*Rule: std::auto_ptr is deprecated and should not be used. (Use std::unique_ptr or std::shared_ptr instead).*.

## **Moving forward**

The core problem with the design of std::auto_ptr is that prior to C++11, the C++ language simply had no mechanism to differentiate “copy semantics” from “move semantics”. Overriding the copy semantics to implement move semantics leads to weird edge cases and inadvertent bugs. For example, you can write `res1 = res2` and have no idea whether res2 will be changed or not!

设计std :: auto_ptr的核心问题是，在C ++ 11之前，C ++语言根本没有区分“复制语义”和“移动语义”的机制。 覆盖复制语义来实现移动语义会导致奇怪的情况和无意间的错误。 例如，你可以编写“ res1 = res2”这样的代码，却不知道res2是不是会被更改！（不知道这是转移还是拷贝）

Because of this, in C++11, the concept of “move” was formally defined, and “move semantics” were added to the language to properly differentiate copying from moving. Now that we’ve set the stage for why move semantics can be useful, we’ll explore the topic of move semantics throughout the rest of this chapter. We’ll also fix our Auto_ptr2 class using move semantics.

因此，在C ++ 11中，正式定义了“移动”的概念，并在语言中添加了“移动语义”以适当地区分复制与移动。 现在，我们为移动语义为何有用作了准备，我们将在本章其余部分中探讨移动语义。 我们还会使用移动语义来修复我们的Auto_ptr2类。

In C++11, std::auto_ptr has been replaced by a bunch of other types of “move-aware” smart pointers: std::unique_ptr, std::weak_ptr, and std::shared_ptr. We’ll also explore the two most popular of these: unique_ptr (which is a direct replacement for auto_ptr) and shared_ptr.

在C ++ 11中，std :: auto_ptr被其他类型的“移动感知”智能指针所替代：std :: unique_ptr，std :: weak_ptr和std :: shared_ptr。 我们还将探讨其中两个最受欢迎的方法：unique_ptr（直接替代auto_ptr）和shared_ptr。