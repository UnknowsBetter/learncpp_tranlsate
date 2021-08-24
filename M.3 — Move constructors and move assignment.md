# M.3 — Move constructors and move assignment

In lesson [M.1 -- Intro to smart pointers and move semantics](https://www.learncpp.com/cpp-tutorial/intro-to-smart-pointers-move-semantics/), we took a look at std::auto_ptr, discussed the desire for move semantics, and took a look at some of the downsides that occur when functions designed for copy semantics (copy constructors and copy assignment operators) are redefined to implement move semantics.

In this lesson, we’ll take a deeper look at how C++11 resolves these problems via move constructors and move assignment.

在本节课程，我们会看一下C++11怎么解决这些问题

### **Copy constructors and copy assignment**

First, let’s take a moment to recap copy semantics.

回忆一下拷贝的情况。

Copy constructors are used to initialize a class by making a copy of an object of the same class. Copy assignment is used to copy one class to another existing class. By default, C++ will provide a copy constructor and copy assignment operator if one is not explicitly provided. These compiler-provided functions do shallow copies, which may cause problems for classes that allocate dynamic memory. So classes that deal with dynamic memory should override these functions to do deep copies.

拷贝构造函数是用已有对象创建新对象。拷贝赋值运算是用一个对象赋值给另一个已有对象的过程。如果没有明确提供这二者的话，编译器会提供一个做浅拷贝的版本。可能对那些要申请内存的类造成一些问题。所以跟动态内存分配有关系的类应该重写这两个函数。

Returning back to our Auto_ptr smart pointer class example from the first lesson in this chapter, let’s look at a version that implements a copy constructor and copy assignment operator that do deep copies, and a sample program that exercises them:

看看前面的AutoPtr，让我们看一下做深拷贝的版本，还有一个使用它的简单例子：

```
template<class T>
class Auto_ptr3
{
	T* m_ptr;
public:
	Auto_ptr3(T* ptr = nullptr)
		:m_ptr(ptr)
	{
	}
 
	~Auto_ptr3()
	{
		delete m_ptr;
	}
 
	// Copy constructor
	// Do deep copy of a.m_ptr to m_ptr
	Auto_ptr3(const Auto_ptr3& a)
	{
		m_ptr = new T;
		*m_ptr = *a.m_ptr;
	}
 
	// Copy assignment
	// Do deep copy of a.m_ptr to m_ptr
	Auto_ptr3& operator=(const Auto_ptr3& a)
	{
		// Self-assignment detection
		if (&a == this)
			return *this;
 
		// Release any resource we're holding
		delete m_ptr;
 
		// Copy the resource
		m_ptr = new T;
		*m_ptr = *a.m_ptr;
 
		return *this;
	}
 
	T& operator*() const { return *m_ptr; }
	T* operator->() const { return m_ptr; }
	bool isNull() const { return m_ptr == nullptr; }
};
 
class Resource
{
public:
	Resource() { std::cout << "Resource acquired\n"; }
	~Resource() { std::cout << "Resource destroyed\n"; }
};
 
Auto_ptr3<Resource> generateResource()
{
	Auto_ptr3<Resource> res(new Resource);
	return res; // this return value will invoke the copy constructor
}
 
int main()
{
	Auto_ptr3<Resource> mainres;
	mainres = generateResource(); // this assignment will invoke the copy assignment
 
	return 0;
}
```

In this program, we’re using a function named generateResource() to create a smart pointer encapsulated resource, which is then passed back to function main(). Function main() then assigns that to an existing Auto_ptr3 object.

在这个程序中，我们使用一个叫generateReource的函数来创建一个智能指针，这个指针封装了一个Resource对象，然后传回给了main，函数main把这个返回值给了一个已经存在的autoptr3对象。

When this program is run, it prints:

```
Resource acquired
Resource acquired
Resource destroyed
Resource acquired
Resource destroyed
Resource destroyed
```

(Note: You may only get 4 outputs if your compiler elides the return value from function generateResource())

你可能在你的机器上只得到四个输出。

That’s a lot of resource creation and destruction going on for such a simple program! What’s going on here?

Let’s take a closer look. There are 6 key steps that happen in this program (one for each printed message):

1) Inside generateResource(), local variable res is created and initialized with a dynamically allocated Resource, which causes the first “Resource acquired”.

在generateResource产生了一个Resouce对象，造成了一次Resource acquired

2) Res is returned back to main() by value. We return by value here because res is a local variable -- it can’t be returned by address or reference because res will be destroyed when generateResource() ends. So res is copy constructed into a temporary object. Since our copy constructor does a deep copy, a new Resource is allocated here, which causes the second “Resource acquired”.

Res通过值返回给main，我们通过值返回是因为res是一个局部变量，它不能通过指针或者引用来传回。所以res是拷贝构造了一个临时对象，因为我们拷贝构造函数做深拷贝，所以新的对象在这里构造了，这造成了第二个Resourceacquired。

3) Res goes out of scope, destroying the originally created Resource, which causes the first “Resource destroyed”.

Res死掉的时候，调用了析构函数，所以产生了Resource destroyed。

4) The temporary object is assigned to mainres by copy assignment. Since our copy assignment also does a deep copy, a new Resource is allocated, causing yet another “Resource acquired”.

临时对象赋值给mainRes。因为我们的拷贝复制函数做深拷贝，所以创建了一个新的Resouce对象，又产生了一个“Resouce acquired”。

5) The assignment expression ends, and the temporary object goes out of expression scope and is destroyed, causing a “Resource destroyed”.

赋值表达式结束的时候，这个临时对象走出了表达式作用域，导致了一个“Resouce destroyed”

6) At the end of main(), mainres goes out of scope, and our final “Resource destroyed” is displayed.

在mian函数结尾的时候，mainRes也走出了作用域，最后一个“Resouce destroyed”也出现了。

So, in short, because we call the copy constructor once to copy construct res to a temporary, and copy assignment once to copy the temporary into mainres, we end up allocating and destroying 3 separate objects in total.

简单讲，因为调用拷贝构造函数或者是拷贝复制运算都造成了资源的申请，甚至在赋值运算符里面还进行了一次资源的释放（在本例中，因为mainRes本就不保有一个资源，所以没有打印Destroy信息）。在对象走出作用域的时候也会产生destroy信息。另外函数通过值返回的时候产生了一个临时对象，这里用的是拷贝构造函数，因为这里有新的对象的产生，这是拷贝构造函数和拷贝复制运算的主要区别。

Inefficient, but at least it doesn’t crash!

效率不高，但至少没有崩溃。（因为每个智能指针自己保护自己的对象，不存在二次释放）

However, with move semantics, we can do better.

**但是有了移动语义的话，我们可以做的更好。（效率更高，记得吗？上面的函数返回是一个临时对象，临时对象被认为是右值，我们可以在赋值运算符重载函数中，对于右值引用做特殊的重载）**

### **Move constructors and move assignment**

C++11 defines two **new functions in service of move semantics**: **a move constructor, and a move assignment operator**. Whereas the goal of the copy constructor and copy assignment is to make a copy of one object to another, the goal of the move constructor and move assignment is to move ownership of the resources from one object to another (which is typically much less expensive than making a copy).

C++11 定义了两个新的函数来提供移动语义的服务：一个是移动构造函数，还有一个是移动赋值运算符重载函数。拷贝构造和复制运算的目的是做对象的一份拷贝给另一个对象，而移动构造函数和移动赋值运算符的想法是转移资源的所有权（这样就不需要做深拷贝了）。

Defining a move constructor and move assignment work analogously to their copy counterparts. However, whereas the copy flavors of these functions take a const l-value reference parameter, the move flavors of these functions use non-const r-value reference parameters.

定义一个移动构造函数和移动赋值运算符函数的工作和拷贝构造函数和拷贝赋值运算符是很相似的。然而，这些函数的拷贝版接受一个左值引用参数，而移动版接受一个右值引用参数。

Here’s the same Auto_ptr3 class as above, with a move constructor and move assignment operator added. We’ve left in the deep-copying copy constructor and copy assignment operator for comparison purposes.

下面是加了移动构造函数和移动赋值运算符的AutorPtr，我们把深拷贝版本也放在了里面，为了做对比。

```
#include <iostream>
 
template<class T>
class Auto_ptr4
{
	T* m_ptr;
public:
	Auto_ptr4(T* ptr = nullptr)
		:m_ptr(ptr)
	{
	}
 
	~Auto_ptr4()
	{
		delete m_ptr;
	}
 
	// Copy constructor
	// Do deep copy of a.m_ptr to m_ptr
	Auto_ptr4(const Auto_ptr4& a)
	{
		m_ptr = new T;
		*m_ptr = *a.m_ptr;
	}
 
	// Move constructor
	// Transfer ownership of a.m_ptr to m_ptr
	Auto_ptr4(Auto_ptr4&& a) noexcept
		: m_ptr(a.m_ptr)
	{
		a.m_ptr = nullptr; // we'll talk more about this line below
	}
 
	// Copy assignment
	// Do deep copy of a.m_ptr to m_ptr
	Auto_ptr4& operator=(const Auto_ptr4& a)
	{
		// Self-assignment detection
		if (&a == this)
			return *this;
 
		// Release any resource we're holding
		delete m_ptr;
 
		// Copy the resource
		m_ptr = new T;
		*m_ptr = *a.m_ptr;
 
		return *this;
	}
 
	// Move assignment
	// Transfer ownership of a.m_ptr to m_ptr
	Auto_ptr4& operator=(Auto_ptr4&& a) noexcept
	{
		// Self-assignment detection
		if (&a == this)
			return *this;
 
		// Release any resource we're holding
		delete m_ptr;
 
		// Transfer ownership of a.m_ptr to m_ptr
		m_ptr = a.m_ptr;
		a.m_ptr = nullptr; // we'll talk more about this line below
 
		return *this;
	}
 
	T& operator*() const { return *m_ptr; }
	T* operator->() const { return m_ptr; }
	bool isNull() const { return m_ptr == nullptr; }
};
 
class Resource
{
public:
	Resource() { std::cout << "Resource acquired\n"; }
	~Resource() { std::cout << "Resource destroyed\n"; }
};
 
Auto_ptr4<Resource> generateResource()
{
	Auto_ptr4<Resource> res(new Resource);
	return res; // this return value will invoke the move constructor
}
 
int main()
{
	Auto_ptr4<Resource> mainres;
	mainres = generateResource(); // this assignment will invoke the move assignment
 
	return 0;
}
```

The move constructor and move assignment operator are simple. Instead of deep copying the source object (a) into the implicit object, we simply move (steal) the source object’s resources. This involves shallow copying the source pointer into the implicit object, then setting the source pointer to null.

**移动构造函数和移动赋值运算符重载是非常简单的，不需要做深拷贝，我们只需要移动源对象的资源。**

When run, this program prints:

```
Resource acquired
Resource destroyed
```

That’s much better!

好了太多！

The flow of the program is exactly the same as before. However, instead of calling the copy constructor and copy assignment operators, this program calls the move constructor and move assignment operators. Looking a little more deeply:

main函数和之前是一模一样的。然而，不调用拷贝构造函数和拷贝赋值运算，这个程序调用的是移动构造函数和移动赋值运算，我们仔细看一下：

1) Inside generateResource(), local variable res is created and initialized with a dynamically allocated Resource, which causes the first “Resource acquired”.

在generateResource中，本地变量被创建，这是第一次Resouce acquired消息产生的地方

2) Res is returned back to main() by value. Res is move constructed into a temporary object, transferring the dynamically created object stored in res to the temporary object. We’ll talk about why this happens below.

**Res通过值返回给main，Res被移动到了一个临时对象，进行了资源所有权的转移。我们下面会讲为什么这里会移动一个局部变量。**

3) Res goes out of scope. Because res no longer manages a pointer (it was moved to the temporary), nothing interesting happens here.

Res死掉，因为res不再保有资源，所有这里没有发生有趣的事情。

4) The temporary object is move assigned to mainres. This transfers the dynamically created object stored in the temporary to mainres.

临时对象被移动赋值给了mainres，转移了临时对象创中的创建的资源给了mainres。

5) The assignment expression ends, and the temporary object goes out of expression scope and is destroyed. However, because the temporary no longer manages a pointer (it was moved to mainres), nothing interesting happens here either.

在赋值运算符结束的时候，临时对象走出了表达式作用域，然后临时对象死亡。然而，因为临时对象拥有的资源已经被转移出去了，这里也没有发生别的什么事情。

6) At the end of main(), mainres goes out of scope, and our final “Resource destroyed” is displayed.

在mian函数结束的时候，mainres死掉，然后我们的最后的“Resouce destroyed”被打印了出来。

So instead of copying our Resource twice (once for the copy constructor and once for the copy assignment), we transfer it twice. This is more efficient, as Resource is only constructed and destroyed once instead of three times.

所以我们不会拷贝资源两次（一次是从局部变量到临时变量，一次是临时变量到mainres），我们转移两次。这样效率更高，Resource只构造和销毁一次，而不是三次。

### **When are the move constructor and move assignment called?**

The move constructor and move assignment are called when those functions have been defined, and the argument for construction or assignment is an r-value. Most typically, this r-value will be a literal or temporary value.

在定义了移动构造函数和移动赋值运算符的情况下，并且构造函数参数和赋值运算符的参数是右值的情况下，这两个函数会被调用。通常，这个右值是一个字面量，或者是一个临时值。

In most cases, a move constructor and move assignment operator will not be provided by default, unless the class does not have any defined copy constructors, copy assignment, move assignment, or destructors. However, the default move constructor and move assignment do the same thing as the default copy constructor and copy assignment (make copies, not do moves).

大部分情况下移动构造函数和移动赋值运算符默认不会被提供，除非这个类没有定义任何的拷贝构造函数，拷贝复制，移动赋值和移动拷贝函数。然而，默认的移动构造函数和移动赋值运算符重载函数做的事情和拷贝构造函数和拷贝

*Rule: If you want a move constructor and move assignment that do moves, you’ll need to write them yourself.*

如果你想要移动语义的移动构造函数和移动赋值运算符重载函数，你应该自己写一份。

### **The key insight behind move semantics**

You now have enough context to understand the key insight behind move semantics.

你现在已经有足够的相关知识来理解这背后的关键了。

If we construct an object or do an assignment where the argument is an l-value, the only thing we can reasonably do is copy the l-value. We can’t assume it’s safe to alter the l-value, because it may be used again later in the program. If we have an expression “a = b”, we wouldn’t reasonably expect b to be changed in any way.

如果我们在构造一个新对象，或者给一个对象赋值的时候，参数是左值的话，我们可以做的唯一合理的事情就是拷贝这个左值。我们不能假定修改这个左值是安全，因为可能这个左值稍后还要用。如果我们有一个“a=b"的表达式，我们不能理解b会发生变化，我们应该觉得b不应该变。

However, if we construct an object or do an assignment where the argument is an r-value, then we know that r-value is just a temporary object of some kind. Instead of copying it (which can be expensive), we can simply transfer its resources (which is cheap) to the object we’re constructing or assigning. This is safe to do because the temporary will be destroyed at the end of the expression anyway, so we know it will never be used again!

然后，如果我们在构造一个新的对象，或者做赋值运算符，参数是一个右值的话，我们知道右值仅仅是某种临时的对象（后面不能再用）。这个时候我们就不拷贝它了（拷贝的代价大），我们转移它的拥有物（代价小）。这样做很安全，因为临时对象在表达式结束的时候就被销毁，我们知道它不会再被别的地方使用了。

C++11, through r-value references, gives us the ability to provide different behaviors when the argument is an r-value vs an l-value, enabling us to make smarter and more efficient decisions about how our objects should behave.

C++11，通过右值引用。给了我们能力在参数是左值还是右值的时候提供不同的函数行为。允许我们产生更加智能和更加有效率的决定（转移还是移动）。

### **Move functions should always leave both objects in a well-defined state**

In the above examples, both the move constructor and move assignment functions set a.m_ptr to nullptr. This may seem extraneous -- after all, if “a” is a temporary r-value, why bother doing “cleanup” if parameter “a” is going to be destroyed anyway?

在上面的例子中，移动构造和移动赋值函数中都把a的指针设置为了null，这可能看起来有点多余。毕竟，如果a是一个临时对象，干嘛要费劲做清理操作呢？

The answer is simple: When “a” goes out of scope, a’s destructor will be called, and a.m_ptr will be deleted. If at that point, a.m_ptr is still pointing to the same object as m_ptr, then m_ptr will be left as a dangling pointer. When the object containing m_ptr eventually gets used (or destroyed), we’ll get undefined behavior.

答案很简单，因为a走出作用域的时候，a的析构函数就会被调用，然后a的指针就会被删除。如果在那个时候，a拥有的指针仍然指向那个内存对象。那么那个对象会被释放，而新构造的或者被赋值的对象就会拥有一个悬挂指针，在这个对象的悬挂指针被使用的时候，我们不知道会发生什么。使用一个悬挂指针是一个未定义行为。

Additionally, in the next lesson we’ll see cases where “a” can be an l-value. In such a case, “a” wouldn’t be destroyed immediately, and could be queried further before its lifetime ends.

**此外，在下一节我们会看到一个例子，a可以是一个左值，在这样的情况下，a不会被立即销毁。**

### **Automatic l-values returned by value may be moved instead of copied**

In the generateResource() function of the Auto_ptr4 example above, when variable res is returned by value, it is moved instead of copied, even though res is an l-value. The C++ specification has a special rule that says automatic objects returned from a function by value can be moved even if they are l-values. This makes sense, since res was going to be destroyed at the end of the function anyway! We might as well steal its resources instead of making an expensive and unnecessary copy.

在上面的generateResource函数中，变量res通过值返回，尽管res是一个左值而不是右值，然而在这里发生的移动而不是拷贝，C++标准有一个特别的规则是这么描述的：从函数中通过值传回的自动变量可以使用移动语义，尽管这个变量是一个左值。这样做是对的，因为这个自动变量马上就要死了，不会再被使用了，我们不妨窃取其资源，而不是制作昂贵又没必要的拷贝。

Although the compiler can move l-value return values, in some cases it may be able to do even better by simply eliding the copy altogether (which avoids the need to make a copy or do a move at all). In such a case, neither the copy constructor nor move constructor would be called.

尽管编译器可以移动这个被返回的左值，在某些情况下，它甚至能做的更好，连move都不做。在这样的情况下，拷贝构造和移动构造都不会被调用。

### **Disabling copying**

In the Auto_ptr4 class above, we left in the copy constructor and assignment operator for comparison purposes. But in move-enabled classes, it is sometimes desirable to delete the copy constructor and copy assignment functions to ensure copies aren’t made. In the case of our Auto_ptr class, we don’t want to copy our templated object T -- both because it’s expensive, and whatever class T is may not even support copying!

在前面为了比较，我们把拷贝构造函数和拷贝赋值运算符重载函数留下来了。但是在允许移动的类中，有时会我们想把拷贝构造函数和拷贝赋值运算符都删掉，确保拷贝不会发生。在我们的AutoPtr勒种，我们不想拷贝我们的模板类T，因为它的代价很大，而且有可能这个T根本不支持拷贝。

Here’s a version of Auto_ptr that supports move semantics but not copy semantics:

下面是一支持移动但不支持拷贝的智能指针类。

```
#include <iostream>
 
template<class T>
class Auto_ptr5
{
	T* m_ptr;
public:
	Auto_ptr5(T* ptr = nullptr)
		:m_ptr(ptr)
	{
	}
 
	~Auto_ptr5()
	{
		delete m_ptr;
	}
 
	// Copy constructor -- no copying allowed!
	Auto_ptr5(const Auto_ptr5& a) = delete;
 
	// Move constructor
	// Transfer ownership of a.m_ptr to m_ptr
	Auto_ptr5(Auto_ptr5&& a) noexcept
		: m_ptr(a.m_ptr)
	{
		a.m_ptr = nullptr;
	}
 
	// Copy assignment -- no copying allowed!
	Auto_ptr5& operator=(const Auto_ptr5& a) = delete;
 
	// Move assignment
	// Transfer ownership of a.m_ptr to m_ptr
	Auto_ptr5& operator=(Auto_ptr5&& a) noexcept
	{
		// Self-assignment detection
		if (&a == this)
			return *this;
 
		// Release any resource we're holding
		delete m_ptr;
 
		// Transfer ownership of a.m_ptr to m_ptr
		m_ptr = a.m_ptr;
		a.m_ptr = nullptr;
 
		return *this;
	}
 
	T& operator*() const { return *m_ptr; }
	T* operator->() const { return m_ptr; }
	bool isNull() const { return m_ptr == nullptr; }
};
```

If you were to try to pass an Auto_ptr5 l-value to a function by value, the compiler would complain that the copy constructor required to initialize the copy constructor argument has been deleted. This is good, because we should probably be passing Auto_ptr5 by const l-value reference anyway!

如果你尝试传递一个左值给函数以值的形式，编译器会报错，需要的拷贝构造函数已经删掉了，这是非常好的，因为我们总应使用左值常引用来进行参数的传递。

Auto_ptr5 is (finally) a good smart pointer class. And, in fact the standard library contains a class very much like this one (that you should use instead), named std::unique_ptr. We’ll talk more about std::unique_ptr later in this chapter.

现在这个AutoPtr是一个相当好的只能指针，而且实际上标准库中有一个类跟这个很像（你应该用那一个），叫做unique_ptr。我们会在后面讨论这个uniquePtr。

### **Another example**

Let’s take a look at another class that uses dynamic memory: a simple dynamic templated array. This class contains a deep-copying copy constructor and copy assignment operator.

我们看一下另一个使用了动态内存的类。这个类包含一个深拷贝构造函数和拷贝赋值运算符重载函数。

```
#include <iostream>
 
template <class T>
class DynamicArray
{
private:
	T* m_array;
	int m_length;
 
public:
	DynamicArray(int length)
		: m_array(new T[length]), m_length(length)
	{
	}
 
	~DynamicArray()
	{
		delete[] m_array;
	}
 
	// Copy constructor
	DynamicArray(const DynamicArray &arr)
		: m_length(arr.m_length)
	{
		m_array = new T[m_length];
		for (int i = 0; i < m_length; ++i)
			m_array[i] = arr.m_array[i];
	}
 
	// Copy assignment
	DynamicArray& operator=(const DynamicArray &arr)
	{
		if (&arr == this)
			return *this;
 
		delete[] m_array;
		
		m_length = arr.m_length;
		m_array = new T[m_length];
 
		for (int i = 0; i < m_length; ++i)
			m_array[i] = arr.m_array[i];
 
		return *this;
	}
 
	int getLength() const { return m_length; }
	T& operator[](int index) { return m_array[index]; }
	const T& operator[](int index) const { return m_array[index]; }
 
};
```

Now let’s use this class in a program. To show you how this class performs when we allocate a million integers on the heap, we’re going to leverage the Timer class we developed in lesson [11.18 -- Timing your code](https://www.learncpp.com/cpp-tutorial/timing-your-code/). We’ll use the Timer class to time how fast our code runs, and show you the performance difference between copying and moving.

然后我们在程序中用一下这个类，来给你看一下拷贝和移动的性能差距。

```
#include <iostream>
#include <chrono> // for std::chrono functions
 
// Uses the above DynamicArray class
 
class Timer
{
private:
	// Type aliases to make accessing nested type easier
	using clock_t = std::chrono::high_resolution_clock;
	using second_t = std::chrono::duration<double, std::ratio<1> >;
	
	std::chrono::time_point<clock_t> m_beg;
 
public:
	Timer() : m_beg(clock_t::now())
	{
	}
	
	void reset()
	{
		m_beg = clock_t::now();
	}
	
	double elapsed() const
	{
		return std::chrono::duration_cast<second_t>(clock_t::now() - m_beg).count();
	}
};
 
// Return a copy of arr with all of the values doubled
DynamicArray<int> cloneArrayAndDouble(const DynamicArray<int> &arr)
{
	DynamicArray<int> dbl(arr.getLength());
	for (int i = 0; i < arr.getLength(); ++i)
		dbl[i] = arr[i] * 2;
 
	return dbl;
}
 
int main()
{
	Timer t;
 
	DynamicArray<int> arr(1000000);
 
	for (int i = 0; i < arr.getLength(); i++)
		arr[i] = i;
 
	arr = cloneArrayAndDouble(arr);
 
	std::cout << t.elapsed();
}
```

On one of the author’s machines, in release mode, this program executed in 0.00825559 seconds.

在作者的机器上，在发布模式下，这个程序执行了0.008秒。

Now let’s run the same program again, replacing the copy constructor and copy assignment with a move constructor and move assignment.

我们把拷贝构造和拷贝赋值运算改成移动构造和移动赋值运算，然后再运行一下这个程序。

```
template <class T>
class DynamicArray
{
private:
	T* m_array;
	int m_length;
 
public:
	DynamicArray(int length)
		: m_array(new T[length]), m_length(length)
	{
	}
 
	~DynamicArray()
	{
		delete[] m_array;
	}
 
	// Copy constructor
	DynamicArray(const DynamicArray &arr) = delete;
 
	// Copy assignment
	DynamicArray& operator=(const DynamicArray &arr) = delete;
 
	// Move constructor
	DynamicArray(DynamicArray &&arr) noexcept
		: m_length(arr.m_length), m_array(arr.m_array)
	{
		arr.m_length = 0;
		arr.m_array = nullptr;
	}
 
	// Move assignment
	DynamicArray& operator=(DynamicArray &&arr) noexcept
	{
		if (&arr == this)
			return *this;
 
		delete[] m_array;
 
		m_length = arr.m_length;
		m_array = arr.m_array;
		arr.m_length = 0;
		arr.m_array = nullptr;
 
		return *this;
	}
 
	int getLength() const { return m_length; }
	T& operator[](int index) { return m_array[index]; }
	const T& operator[](int index) const { return m_array[index]; }
 
};
 
#include <iostream>
#include <chrono> // for std::chrono functions
 
class Timer
{
private:
	// Type aliases to make accessing nested type easier
	using clock_t = std::chrono::high_resolution_clock;
	using second_t = std::chrono::duration<double, std::ratio<1> >;
 
	std::chrono::time_point<clock_t> m_beg;
 
public:
	Timer() : m_beg(clock_t::now())
	{
	}
 
	void reset()
	{
		m_beg = clock_t::now();
	}
 
	double elapsed() const
	{
		return std::chrono::duration_cast<second_t>(clock_t::now() - m_beg).count();
	}
};
 
// Return a copy of arr with all of the values doubled
DynamicArray<int> cloneArrayAndDouble(const DynamicArray<int> &arr)
{
	DynamicArray<int> dbl(arr.getLength());
	for (int i = 0; i < arr.getLength(); ++i)
		dbl[i] = arr[i] * 2;
 
	return dbl;
}
 
int main()
{
	Timer t;
 
	DynamicArray<int> arr(1000000);
 
	for (int i = 0; i < arr.getLength(); i++)
		arr[i] = i;
 
	arr = cloneArrayAndDouble(arr);
 
	std::cout << t.elapsed();
}
```

On the same machine, this program executed in 0.0056 seconds.

在相同的机器上，这个程序执行使用了0.0056秒。

Comparing the runtime of the two programs, 0.0056 / 0.00825559 = 67.8%. The move version was almost 33% faster!

两者比较一下，move的版本要快百分之33.