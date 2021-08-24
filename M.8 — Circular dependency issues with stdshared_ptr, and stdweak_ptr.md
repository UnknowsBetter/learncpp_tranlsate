# M.8 — Circular dependency issues with std::shared_ptr, and std::weak_ptr

In the previous lesson, we saw how std::shared_ptr allowed us to have multiple smart pointers co-owning the same resource. However, in certain cases, this can become problematic. Consider the following case, where the shared pointers in two separate objects each point at the other object:

在前面我们展示了共享智能指针的用法。然而，在特定的情形下，共享指针可能会带来问题，思考下面的例子。

两个共享指针在类里面，指向了对方：

```
#include <iostream>
#include <memory> // for std::shared_ptr
#include <string>
 
class Person
{
	std::string m_name;
	std::shared_ptr<Person> m_partner; // initially created empty
 
public:
		
	Person(const std::string &name): m_name(name)
	{ 
		std::cout << m_name << " created\n";
	}
	~Person()
	{
		std::cout << m_name << " destroyed\n";
	}
 
	friend bool partnerUp(std::shared_ptr<Person> &p1, std::shared_ptr<Person> &p2)
	{
		if (!p1 || !p2)
			return false;
 
		p1->m_partner = p2;
		p2->m_partner = p1;
 
		std::cout << p1->m_name << " is now partnered with " << p2->m_name << "\n";
 
		return true;
	}
};
 
int main()
{
	auto lucy = std::make_shared<Person>("Lucy"); // create a Person named "Lucy"
	auto ricky = std::make_shared<Person>("Ricky"); // create a Person named "Ricky"
 
	partnerUp(lucy, ricky); // Make "Lucy" point to "Ricky" and vice-versa
 
	return 0;
}
```

In the above example, we dynamically allocate two Persons, “Lucy” and “Ricky” using make_shared() (to ensure lucy and ricky are destroyed at the end of main()). Then we partner them up. This sets the std::shared_ptr inside “Lucy” to point at “Ricky”, and the std::shared_ptr inside “Ricky” to point at “Lucy”. Shared pointers are meant to be shared, so it’s fine that both the lucy shared pointer and Rick’s m_partner shared pointer both point at “Lucy” (and vice-versa).

在上面的示例中，我们使用make_shared（）动态分配了两个Persons“ Lucy”和“ Ricky”（以确保在main（）的末尾销毁了lucy和ricky）。 然后，我们让他们相互合作。 这会使“ Lucy”内部的std :: shared_ptr设置为指向“ Ricky”，而将“ Ricky”内部的std :: shared_ptr指向为“ Lucy”。 共享指针是要共享的，因此lucy共享指针和Rick的m_partner共享指针都指向“ Lucy”是可以的（反之亦然）。

However, this program doesn’t execute as expected:

```
Lucy created
Ricky created
Lucy is now partnered with Ricky
```

And that’s it. No deallocations took place. Uh. oh. What happened?

没有发生析构，我靠！为什么？

After partnerUp() is called, there are two shared pointers pointing to “Ricky” (ricky, and Lucy’s m_partner) and two shared pointers pointing to “Lucy” (lucy, and Ricky’s m_partner).

在parterUp调用之后，有两个共享指针指向Ricky，也有两个共享指针指向Lucy

At the end of main(), the ricky shared pointer goes out of scope first. When that happens, ricky checks if there are any other shared pointers that co-own the Person “Ricky”. There are (Lucy’s m_partner). Because of this, it doesn’t deallocate “Ricky” (if it did, then Lucy’s m_partner would end up as a dangling pointer). At this point, we now have one shared pointer to “Ricky” (Lucy’s m_partner) and two shared pointers to “Lucy” (lucy, and Ricky’s m_partner).

在main的末尾，ricky的共享指针首先超出范围。 ricky会检查是否有其他共享指针共同拥有“ Ricky”人员。 答案是有（露西的m_partner）。 因此，它不会析构“ Ricky”（如果析构了的话，那么Lucy的m_partner最终将成为悬空指针）。 现在，我们现在有一个共享的指针指向“ Ricky”（露西的m_partner）和两个共享的指针指向“ Lucy”（露西和Ricky的m_partner）。

Next the lucy shared pointer goes out of scope, and the same thing happens. The shared pointer lucy checks if there are any other shared pointers co-owning the Person “Lucy”. There are (Ricky’s m_partner), so “Lucy” isn’t deallocated. At this point, there is one shared pointer to “Lucy” (Ricky’s m_partner) and one shared pointer to “Ricky” (Lucy’s m_partner).

接下来，露西共享指针超出范围，发生同样的事情。 共享指针lucy检查是否有其他个人共同拥有“ Lucy”。 答案是有（Ricky的m_partner），因此不会释放“ Lucy”。 在这里，有一个共享的指针指向“ Lucy”（Ricky的m_partner）和一个共享的指针指向“ Ricky”（Lucy的m_partner）。

Then the program ends -- and neither Person “Lucy” or “Ricky” have been deallocated! Essentially, “Lucy” ends up keeping “Ricky” from being destroyed, and “Ricky” ends up keeping “Lucy” from being destroyed.

然后程序结束了。“ Lucy”或“ Ricky”人都没有被释放！ 实际上，“Lucy”使“ Ricky”免遭析构，而“ Ricky”使“ Lucy”免遭析构。

It turns out that this can happen any time shared pointers form a circular reference.

事实证明，在共享指针形成循环引用的任何时间都会发生这种情况。

### **Circular references** 循环引用

A **Circular reference** (also called a **cyclical reference** or a **cycle**) is a series of references where each object references the next, and the last object references back to the first, causing a referential loop. The references do not need to be actual C++ references -- they can be pointers, unique IDs, or any other means of identifying specific objects.

循环引用（也称为循环引用或循环）是一系列引用，其中每个对象都引用下一个，最后一个对象引用回到第一个，从而导致引用 循环。 引用不需要真的是C ++references ，它们可以是指针，唯一ID或是标识特定对象的任何其他方式。

In the context of shared pointers, the references will be pointers.

在共享智能指针的情况下，引用是指针。

This is exactly what we see in the case above: “Lucy” points at “Ricky”, and “Ricky” points at “Lucy”. With three pointers, you’d get the same thing when A points at B, B points at C, and C points at A. The practical effect of having shared pointers form a cycle is that each object ends up keeping the next object alive -- with the last object keeping the first object alive. Thus, no objects in the series can be deallocated because they all think some other object still needs it!

这也就是我们在上面的情况中看到的：“ Lucy”指向“ Ricky”，“ Ricky”指向“ Lucy”。 三个指针，当A指向B，B指向C，C指向A时，你也会得到相同的结果。共享指针形成一个循环的实际效果是，每个对象最终都会使下一个对象保持活动状态，最后一个对象使第一个对象保持活动状态。 因此，这个环里面的任何对象都无法释放，因为他们都认为其他某个对象仍然需要它！

### **A reductive case** 一个还原的案例

It turns out, this cyclical reference issue can even happen with a single std::shared_ptr -- a std::shared_ptr referencing the object that contains it is still a cycle (just a reductive one). Although it’s fairly unlikely that this would ever happen in practice, we’ll show you for additional comprehension:

事实证明，使用单个std :: shared_ptr都可以发生这样的循环引用问题。引用包含这个智能指针的对象的只能指针仍然是一个循环（只是一个还原循环）。 尽管在实践中不太可能会发生这种情况，但我们还是会给你展示这个东西：

```
#include <iostream>
#include <memory> // for std::shared_ptr
 
class Resource
{
public:
	std::shared_ptr<Resource> m_ptr; // initially created empty
	
	Resource() { std::cout << "Resource acquired\n"; }
	~Resource() { std::cout << "Resource destroyed\n"; }
};
 
int main()
{
	auto ptr1 = std::make_shared<Resource>();
 
	ptr1->m_ptr = ptr1; // m_ptr is now sharing the Resource that contains it
 
	return 0;
}
```

In the above example, when ptr1 goes out of scope, it doesn’t deallocate the Resource because the Resource’s m_ptr is sharing the Resource. Then there’s nobody left to delete the Resource (m_ptr never goes out of scope, so it never gets a chance). Thus, the program prints:

在上面的例子中，当ptr1死亡的时候，它不会释放资源，因为资源的成员指针正在共享资源。所以就没有最后一个人来释放资源。问题就发生了：

```
Resource acquired
```

and that’s it.

### **So what is std::weak_ptr for anyway?** 弱指针的设计目的

std::weak_ptr was designed to solve the “cyclical ownership” problem described above. A std::weak_ptr is an observer -- it can observe and access the same object as a std::shared_ptr (or other std::weak_ptrs) but it is not considered an owner. Remember, when a std::shared pointer goes out of scope, it only considers whether other std::shared_ptr are co-owning the object. std::weak_ptr does not count!

std :: weak_ptr旨在解决上面的“循环的所有权”的问题。 std :: weak_ptr是一个观察者。它可以观察和访问与std :: shared_ptr（或其他的std :: weak_ptr）指针管理的对象，但不被视为这个对象的所有者（仅仅是观察者）。 记住：当std :: shared指针该死的时候，它仅考虑其他std :: shared_ptr是否共同拥有该对象。 std :: weak_ptr不被计算在内！

Let’s solve our Person-al issue using a std::weak_ptr:

让我们用weak_ptr来解决上面的人的问题。

```
#include <iostream>
#include <memory> // for std::shared_ptr and std::weak_ptr
#include <string>
 
class Person
{
	std::string m_name;
	std::weak_ptr<Person> m_partner; // note: This is now a std::weak_ptr
 
public:
		
	Person(const std::string &name): m_name(name)
	{ 
		std::cout << m_name << " created\n";
	}
	~Person()
	{
		std::cout << m_name << " destroyed\n";
	}
 
	friend bool partnerUp(std::shared_ptr<Person> &p1, std::shared_ptr<Person> &p2)
	{
		if (!p1 || !p2)
			return false;
 
		p1->m_partner = p2;
		p2->m_partner = p1;
 
		std::cout << p1->m_name << " is now partnered with " << p2->m_name << "\n";
 
		return true;
	}
};
 
int main()
{
	auto lucy = std::make_shared<Person>("Lucy");
	auto ricky = std::make_shared<Person>("Ricky");
 
	partnerUp(lucy, ricky);
 
	return 0;
}
```

This code behaves properly:

```
Lucy created
Ricky created
Lucy is now partnered with Ricky
Ricky destroyed
Lucy destroyed
```

Functionally, it works almost identically to the problematic example. However, now when ricky goes out of scope, it sees that there are no other std::shared_ptr pointing at “Ricky” (the std::weak_ptr from “Lucy” doesn’t count). Therefore, it will deallocate “Ricky”. The same occurs for lucy.

从功能上讲，它的工作原理与有问题的示例几乎相同。 但是，现在当ricky超出范围时，它会发现没有其他std :: shared_ptr指向“ Ricky”（“ Lucy”中的std :: weak_ptr不计算在内）。 因此，它将释放掉Ricky。 lucy也是如此。

### **Using std::weak_ptr** 弱指针的用法

The downside of std::weak_ptr is that std::weak_ptr are not directly usable (they have no operator->). To use a std::weak_ptr, you must first convert it into a std::shared_ptr. Then you can use the std::shared_ptr. To convert a std::weak_ptr into a std::shared_ptr, you can use the lock() member function. Here’s the above example, updated to show this off:

std :: weak_ptr的缺点是std :: weak_ptr不能直接被使用（它们没有->运算符重载）。 要使用std :: weak_ptr，必须首先将其先转换为std :: shared_ptr。 然后，你就可以使用std :: shared_ptr了。 要将std :: weak_ptr转换为std :: shared_ptr的话，可以使用lock成员函数。 看下面：

```
#include <iostream>
#include <memory> // for std::shared_ptr and std::weak_ptr
#include <string>
 
class Person
{
	std::string m_name;
	std::weak_ptr<Person> m_partner; // note: This is now a std::weak_ptr
 
public:
 
	Person(const std::string &name) : m_name(name)
	{
		std::cout << m_name << " created\n";
	}
	~Person()
	{
		std::cout << m_name << " destroyed\n";
	}
 
	friend bool partnerUp(std::shared_ptr<Person> &p1, std::shared_ptr<Person> &p2)
	{
		if (!p1 || !p2)
			return false;
 
		p1->m_partner = p2;
		p2->m_partner = p1;
 
		std::cout << p1->m_name << " is now partnered with " << p2->m_name << "\n";
 
		return true;
	}
 
	const std::shared_ptr<Person> getPartner() const { return m_partner.lock(); } // use lock() to convert weak_ptr to shared_ptr
	const std::string& getName() const { return m_name; }
};
 
int main()
{
	auto lucy = std::make_shared<Person>("Lucy");
	auto ricky = std::make_shared<Person>("Ricky");
 
	partnerUp(lucy, ricky);
 
	auto partner = ricky->getPartner(); // get shared_ptr to Ricky's partner
	std::cout << ricky->getName() << "'s partner is: " << partner->getName() << '\n';
 
	return 0;
}
```

This prints:

```
Lucy created
Ricky created
Lucy is now partnered with Ricky
Ricky's partner is: Lucy
Ricky destroyed
Lucy destroyed
```

We don’t have to worry about circular dependencies with std::shared_ptr variable “partner” since it’s just a local variable inside the function. It will eventually go out of scope at the end of the function and the reference count will be decremented by 1.

我们现在不必担心共享智能指针partner的循环依赖关系，因为它只是函数内部的局部变量。 它最终将在函数末尾超出范围，并且引用计数将减少1。

### **Conclusion**

std::shared_ptr can be used when you need multiple smart pointers that can co-own a resource. The resource will be deallocated when the last std::shared_ptr goes out of scope. std::weak_ptr can be used when you want a smart pointer that can see and use a shared resource, but does not participate in the ownership of that resource.

当你需要可以共同拥有同一资源的多个智能指针时，你可以使用std :: shared_ptr。 当最后一个std :: shared_ptr超出范围时，资源将被释放。 当您想要一个可以访问和使用共享资源但不参与这个资源所有权的智能指针时，使用std :: weak_ptr。