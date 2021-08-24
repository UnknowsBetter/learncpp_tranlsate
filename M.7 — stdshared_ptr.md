# M.7 — std::shared_ptr

Unlike std::unique_ptr, which is designed to singly own and manage a resource, std::shared_ptr is meant to solve the case where you need multiple smart pointers co-owning a resource.

跟unique_ptr不一样，shared_ptr是用来解决你需要多个指针同时拥有一个资源的情况的。

This means that it is fine to have multiple std::shared_ptr pointing to the same resource. Internally, std::shared_ptr keeps track of how many std::shared_ptr are sharing the resource. As long as at least one std::shared_ptr is pointing to the resource, the resource will not be deallocated, even if individual std::shared_ptr are destroyed. As soon as the last std::shared_ptr managing the resource goes out of scope (or is reassigned to point at something else), the resource will be deallocated.

这也就是说，使用多个shared_ptr指向同一资源是没问题的。在shared_ptr内部，跟踪了有多少个shared_ptr正在使用这个资源，只要还有一个shared_ptr指向资源，这个资源就不会被析构。只要最后一个持有资源的指针死亡（或者被重新赋值，指向了一个别的东西），这个管理的资源会被释放。

Like std::unique_ptr, std::shared_ptr lives in the <memory> header.

和unique_ptr一样, shared_ptr也在memory头文件中。

```
#include <iostream>
#include <memory> // for std::shared_ptr
 
class Resource
{
public:
	Resource() { std::cout << "Resource acquired\n"; }
	~Resource() { std::cout << "Resource destroyed\n"; }
};
 
int main()
{
	// allocate a Resource object and have it owned by std::shared_ptr
	Resource *res = new Resource;
	std::shared_ptr<Resource> ptr1(res);
	{
		std::shared_ptr<Resource> ptr2(ptr1); // use copy initialization to make another std::shared_ptr pointing to the same thing
 
		std::cout << "Killing one shared pointer\n";
	} // ptr2 goes out of scope here, but nothing happens
 
	std::cout << "Killing another shared pointer\n";
 
	return 0;
} // ptr1 goes out of scope here, and the allocated Resource is destroyed
```

This prints:

```
Resource acquired
Killing one shared pointer
Killing another shared pointer
Resource destroyed
```

In the above code, we create a dynamic Resource object, and set a std::shared_ptr named ptr1 to manage it. Inside the nested block, we use copy initialization (which is allowed with std::shared_ptr, since the resource can be shared) to create a second std::shared_ptr (ptr2) that points to the same Resource. When ptr2 goes out of scope, the Resource is not deallocated, because ptr1 is still pointing at the Resource. When ptr1 goes out of scope, ptr1 notices there are no more std::shared_ptr managing the Resource, so it deallocates the Resource.

在上面的代码中，我们创建了一个动态的Resource对象，然后让一个shared_ptr来管理它。在嵌套块里面，我们使用拷贝构造函数（拷贝构造在shared_ptr是允许的，因为资源可以被功效）来构造第二个指向相同对象的shared_ptr变量。当ptr2走出作用域的时候，资源没有被释放，因为ptr1仍然指向这个资源。当ptr1走出作用域的时候，ptr1注意到已经没有的shared_ptr指向管理的这个对象了，所以它释放了这个资源。

Note that we created a second shared pointer from the first shared pointer (using copy initialization). This is important. Consider the following similar program:

注意我们用第一个智能指针拷贝构造了第二个智能指针，这是关键，看看下面的例子：

```
#include <iostream>
#include <memory> // for std::shared_ptr
 
class Resource
{
public:
	Resource() { std::cout << "Resource acquired\n"; }
	~Resource() { std::cout << "Resource destroyed\n"; }
};
 
int main()
{
	Resource *res = new Resource;
	std::shared_ptr<Resource> ptr1(res);
	{
		std::shared_ptr<Resource> ptr2(res); // create ptr2 directly from res (instead of ptr1)
 
		std::cout << "Killing one shared pointer\n";
	} // ptr2 goes out of scope here, and the allocated Resource is destroyed
 
	std::cout << "Killing another shared pointer\n";
 
	return 0;
} // ptr1 goes out of scope here, and the allocated Resource is destroyed again
```

This program prints:

```
Resource acquired
Killing one shared pointer
Resource destroyed
Killing another shared pointer
Resource destroyed
```

and then crashes (at least on the author’s machine).

崩溃了，至少在作者的电脑上崩溃了。

The difference here is that we created two std::shared_ptr independently from each other. As a consequence, even though they’re both pointing to the same Resource, they aren’t aware of each other. When ptr2 goes out of scope, it thinks it’s the only owner of the Resource, and deallocates it. When ptr1 later goes out of the scope, it thinks the same thing, and tries to delete the Resource again. Then bad things happen.

这里和上面的区别在于，我们创建的两个shared_ptr之间是独立的，没有联系。因此结果就是，尽管他们都指向同一个资源，他们感知不到对方的存在。当ptr2死亡的时候，它认为它自己是最后一个拥有者，然后就把资源释放掉了。当ptr1死亡的时候，它也觉得自己是最后的拥有者，然后尝试delete这个资源。坏事就发生了！

Fortunately, this is easily avoided by using copy assignment or copy initialization when you need multiple shared pointers pointing to the same Resource.

幸运的是，这个事情很容易就能避免，你只需要使用像上面那样拷贝构造或者是拷贝赋值运算符来使得多个智能指针指向同一资源就可以了。

***Rule: Always make a copy of an existing std::shared_ptr if you need more than one std::shared_ptr pointing to the same resource*.**

如果你需要让超过一个shared_ptr指向相同的资源的时候，额外的多保留一个副本。

### **std::make_shared**

Much like std::make_unique() can be used to create a std::unique_ptr in C++14, std::make_shared() can (and should) be used to make a std::shared_ptr. std::make_shared() is available in C++11.

就像make_unique一样，你也应该用make_shared。

Here’s our original example, using std::make_shared():

下面是一个使用了make_shared的例子。

```
#include <iostream>
#include <memory> // for std::shared_ptr
 
class Resource
{
public:
	Resource() { std::cout << "Resource acquired\n"; }
	~Resource() { std::cout << "Resource destroyed\n"; }
};
 
int main()
{
	// allocate a Resource object and have it owned by std::shared_ptr
	auto ptr1 = std::make_shared<Resource>();
	{
		auto ptr2 = ptr1; // create ptr2 using copy initialization of ptr1
 
		std::cout << "Killing one shared pointer\n";
	} // ptr2 goes out of scope here, but nothing happens
 
	std::cout << "Killing another shared pointer\n";
 
	return 0;
} // ptr1 goes out of scope here, and the allocated Resource is destroyed
```

The reasons for using std::make_shared() are the same as std::make_unique() -- std::make_shared() is simpler and safer (there’s no way to directly create two std::shared_ptr pointing to the same resource using this method). However, std::make_shared() is also more performant than not using it. The reasons for this lie in the way that std::shared_ptr keeps track of how many pointers are pointing at a given resource.

使用make_shared的理由和使用make_unique的理由是一样的。make_shared是更简单而且是更快的（没有办法同时创建两个shared_ptr指向同一个资源）。然而使用make_shared比不使用它的效率更高。原因就在shared_ptr跟踪多少个指针指向相同资源的方式里面。

### **Digging into std::shared_ptr** 深入共享指针

Unlike std::unique_ptr, which uses a single pointer internally, std::shared_ptr uses two pointers internally. One pointer points at the resource being managed. The other points at a “control block”, which is a dynamically allocated object that tracks of a bunch of stuff, including how many std::shared_ptr are pointing at the resource. When a std::shared_ptr is created via a std::shared_ptr constructor, the memory for the managed object (which is usually passed in) and control block (which the constructor creates) are allocated separately. However, when using std::make_shared(), this can be optimized into a single memory allocation, which leads to better performance.

不像unique_ptr，内部使用一个指针。shared_ptr内部使用两个指针，一个指针指向被管理的对象。另一个指向一个叫控制块的东西。是一个动态分配的对象，用来跟踪一堆东西，包括有多少个共享指针正在指向这个资源。当一个共享指针被创建的时候，内部为这个被管理的对象（通常是传进来的）和控制块（构造函数创建的）各自分配内存。然而，在使用make_shared函数的时候，这个可以优化为一次内存分配，这样性能更好。

This also explains why independently creating two std::shared_ptr pointed to the same resource gets us into trouble. Each std::shared_ptr will have one pointer pointing at the resource. However, each std::shared_ptr will independently allocate its own control block, which will indicate that it is the only pointer owning that resource. Thus, when that std::shared_ptr goes out of scope, it will deallocate the resource, not realizing there are other std::shared_ptr also trying to manage that resource.

这也揭示了为什么我们直接使用resource指针来创建两个共享智能指针会让我们遇上麻烦。因为每个共享智能指针自己独立的管理自己的控制块，告诉自己它就是唯一拥有这个资源的人。因此，当这些智能指针死亡的时候，它会自己析构掉资源，而没有意识到别人也在管理着相同的资源。

However, when a std::shared_ptr is cloned using copy assignment, the data in the control block can be appropriately updated to indicate that there are now additional std::shared_ptr co-managing the resource.

然而，当shared_ptr被通过拷贝赋值运算来克隆的时候，这个控制块中的数据就可以被合理地更新，然后让双方感知到对方的存在。

### **Shared pointers can be created from unique pointers** 共享指针可以通过独占指针来创建

A std::unique_ptr can be converted into a std::shared_ptr via a special std::shared_ptr constructor that accepts a std::unique_ptr r-value. The contents of the std::unique_ptr will be moved to the std::shared_ptr.

一个unique_ptr可以转化成一个shared_ptr，通过一个特别的愿意接受unique_ptr右值参数的构造函数来完成。此时独占指针拥有的资源被转移到了共享智能指针那里。

However, std::shared_ptr can not be safely converted to a std::unique_ptr. This means that if you’re creating a function that is going to return a smart pointer, you’re better off returning a std::unique_ptr and assigning it to a std::shared_ptr if and when that’s appropriate.

然而，共享指针不能够安全的被转换成独占指针。这意味着如果你正在创建一个会返回智能指针的函数，你最好返回一个unique_ptr，然后你可以把它赋值给一个shared_ptr（你也可以不转换），如果这样合适的话你就这样做。

### **The perils of std::shared_ptr** 共享智能指针的危险性

std::shared_ptr has some of the same challenges as std::unique_ptr -- if the std::shared_ptr is not properly disposed of (either because it was dynamically allocated and never deleted, or it was part of an object that was dynamically allocated and never deleted) then the resource it is managing won’t be deallocated either. With std::unique_ptr, you only have to worry about one smart pointer being properly disposed of. With std::shared_ptr, you have to worry about them all. If any of the std::shared_ptr managing a resource are not properly destroyed, the resource will not be deallocated properly.

std :: shared_ptr与std :: unique_ptr面临一些相同的问题。如果没有合适地处理std :: shared_ptr（要么是因为它是动态分配的，并且从未删除过，要么是它是动态分配的对象的一部分并且永远不会删除），**那么它所管理的资源也不会被释放**。 使用std :: unique_ptr，你只需要操心能不能正确处置的了一个智能指针。 使用std :: shared_ptr，您得操心所有的。 如果管理相同资源的任一std :: shared_ptr没能被正确销毁，那么资源就无法合适地释放。

### **std::shared_ptr and arrays** 共享指针和C风格数组

In C++14 and earlier, std::shared_ptr does not have proper support for managing arrays, and should not be used to manage a C-style array. As of C++17, std::shared_ptr does have support for arrays. However, as of C++17, std::make_shared is still lacking proper support for arrays, and should not be used to create shared arrays. This will likely be addressed in C++20.

在C ++ 14和更早版本中，std :: shared_ptr没有适当的支持来管理数组，因此不应用于管理C样式的数组。 从C ++ 17开始，std :: shared_ptr确实支持数组。 但是，从C ++ 17开始，std :: make_shared仍缺乏对数组的适当支持，因此不应该用于创建共享数组。 这可能会在C ++ 20中解决。

### **Conclusion** 总结

std::shared_ptr is designed for the case where you need multiple smart pointers co-managing the same resource. The resource will be deallocated when the last std::shared_ptr managing the resource is destroyed.

std :: shared_ptr设计用于需要多个智能指针共同管理同一资源的情况。 当最后一个管理资源的std :: shared_ptr被销毁时，资源将被释放。