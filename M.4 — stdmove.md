# M.4 — std::move

Once you start using  move semantics more regularly, you’ll start to find cases where you want to invoke move semantics, but the objects you have to work with are l-values, not r-values. Consider the following swap function as an example:

一旦开始更频繁地使用移动语义，你就会开始发现要使用移动语义的情况中，你使用的对象是左值，而不是右值。 以下面的交换函数为例：

```
#include <iostream>
#include <string>
 
template<class T>
void myswap(T& a, T& b) 
{ 
  T tmp { a }; // invokes copy constructor
  a = b; // invokes copy assignment
  b = tmp; // invokes copy assignment
}
 
int main()
{
	std::string x{ "abc" };
	std::string y{ "de" };
 
	std::cout << "x: " << x << '\n';
	std::cout << "y: " << y << '\n';
 
	myswap(x, y);
 
	std::cout << "x: " << x << '\n';
	std::cout << "y: " << y << '\n';
 
	return 0;
}
```

Passed in two objects of type T (in this case, std::string), this function swaps their values by making three copies. Consequently, this program prints:

给这个swap函数传递两个对象，然后通过三次拷贝来交换两个变量的值。因此这个函数打印出了：

```
x: abc
y: de
x: de
y: abc
```

As we showed last lesson, making copies can be inefficient. And this version of swap makes 3 copies. That leads to a lot of excessive string creation and destruction, which is slow.

像我们上节看到的那样，拷贝是效率低的，这个swap函数做了三次拷贝，所以有很多的字符串被创建和销毁，是很慢的。

However, doing copies isn’t necessary here. All we’re really trying to do is swap the values of a and b, which can be accomplished just as well using 3 moves instead! So if we switch from copy semantics to move semantics, we can make our code more performant.

然而，在这里做拷贝是不必要的。我们在这里其实想做的事情是交换所有权。而这只需要做三次move就可以达到！所以如果我们从拷贝语义变成移动语义，我们就能让我们的代码性能更好。

But how? The problem here is that parameters a and b are l-value references, not r-value references, so we don’t have a way to invoke the move constructor and move assignment operator instead of copy constructor and copy assignment. By default, we get the copy constructor and copy assignment behaviors. What are we to do?

但是这应该是怎么做呢？问题在于参数a和b是一个左值引用而不是右值引用，所以我们没有一个方式来使用移动构造函数和移动赋值运算符符。默认地，我们用的是拷贝构造函数来构造参数，拷贝复制运算来进行函数体内的操作。我们现在怎么办呢？

### **std::move**

In C++11, std::move is a standard library function that serves a single purpose -- to convert its argument into an r-value. We can pass an l-value to std::move, and it will return an r-value reference. std::move is defined in the utility header.

在C++11中，std::move是一个标准库函数，只提供一个功能就是把传给他的参数转换成一个右值。我们可以传一个左值给std::move，然后它会返回一个右值引用。std::move在utility头文件中定义。

Here’s the same program as above, but with a myswap() function that uses std::move to convert our l-values into r-values so we can invoke move semantics:

这跟上面的程序是一样的，但这里用move函数把左值变成了右值，所以我们可以引入移动语义。

```
#include <iostream>
#include <string>
#include <utility> // for std::move
 
template<class T>
void myswap(T& a, T& b) 
{ 
  T tmp { std::move(a) }; // invokes move constructor
  a = std::move(b); // invokes move assignment
  b = std::move(tmp); // invokes move assignment
}
 
int main()
{
	std::string x{ "abc" };
	std::string y{ "de" };
 
	std::cout << "x: " << x << '\n';
	std::cout << "y: " << y << '\n';
 
	myswap(x, y);
 
	std::cout << "x: " << x << '\n';
	std::cout << "y: " << y << '\n';
 
	return 0;
}
```

This prints the same result as above:

这和上面打印出来的结果是一样的

```
x: abc
y: de
x: de
y: abc
```

But it’s much more efficient about it. When tmp is initialized, instead of making a copy of x, we use std::move to convert l-value variable x into an r-value. Since the parameter is an r-value, move semantics are invoked, and x is moved into tmp.

但是这个会更高效。当tmp初始化的时候，这里没有做拷贝，而是做了转移。因为这个参数是一个右值，所以移动语义被引入了，x直接移到了temp里面。

With a couple of more swaps, the value of variable x has been moved to y, and the value of y has been moved to x.

在几次移动之后，x里面的值已经被移动到了y里面，y里面原来的值也移动到了x里面。

### **Another example**

We can also use std::move when filling elements of a container, such as std::vector, with l-values.

我们也可以在填充容器元素的时候，使用move函数，比如stdvector。

In the following program, we first add an element to a vector using copy semantics. Then we add an element to the vector using move semantics.

在下面的例子中，我们首先用拷贝的方法添加了一个元素。然后我们使用移动的方法又添加了一个元素。

```
#include <iostream>
#include <string>
#include <utility> // for std::move
#include <vector>
 
int main()
{
	std::vector<std::string> v;
	std::string str = "Knock";
 
	std::cout << "Copying str\n";
	v.push_back(str); // calls l-value version of push_back, which copies str into the array element
	
	std::cout << "str: " << str << '\n';
	std::cout << "vector: " << v[0] << '\n';
 
	std::cout << "\nMoving str\n";
 
	v.push_back(std::move(str)); // calls r-value version of push_back, which moves str into the array element
	
	std::cout << "str: " << str << '\n';
	std::cout << "vector:" << v[0] << ' ' << v[1] << '\n';
 
	return 0;
}
```

This program prints:

```
Copying str
str: Knock
vector: Knock

Moving str
str:
vector: Knock Knock
```

In the first case, we passed push_back() an l-value, so it used copy semantics to add an element to the vector. For this reason, the value in str is left alone.

在第一个例子中，我们用左值传给了pushback函数，所以它使用了拷贝语义来添加一个元素到vector中，处于这个原因，在str中的值被保留了下来。

In the second case, we passed push_back() an r-value (actually an l-value converted via std::move), so it used move semantics to add an element to the vector. This is more efficient, as the vector element can steal the string’s value rather than having to copy it. In this case, str is left empty.

在第二个例子中，我们用右值传给了pushback函数，实际上是用move函数把左值转换成了右值，所以他在这里使用了移动语义来添加一个进去。这样更高效，因为vector的pushback函数可以把这个值的所有权拿过去而不是做一份拷贝，在这个例子中，str就不剩下什么东西了。

At this point, it’s worth reiterating that std::move() gives a hint to the compiler that the programmer doesn’t need this object any more (at least, not in its current state). Consequently, you should not use std::move() on any persistent object you don’t want to modify, and you should not expect the state of any objects that have had std::move() applied to be the same after they are moved!

在这里，需要重申的是，move是给编译器的一个提示，提示编译器程序员不在需要这个变量了（至少在当前状态下是的）。**因此，你不应该把move函数作用在任何持久的你不想改变的对象上，你也不应该期待被move之后的所有对象的状态是相同的。**

### **Move functions should always leave your objects in a well-defined state**

As we noted in the previous lesson, it’s a good idea to always leave the objects being stolen from in some well-defined (deterministic) state. Ideally, this should be a “null state”, where the object is set back to its uninitiatized or zero state. Now we can talk about why: with std::move, the object being stolen from may not be a temporary after all. The user may want to reuse this (now empty) object again, or test it in some way, and can plan accordingly.

我们在前面提到过，让被偷了的对象处于一个良好定义（确定的状态，比如你有一个刚被移动过的对象，你应该知道你的对象现在的状态是什么）的状态是正确想法。通常应该是一个“null state”。现在我们可以讨论一下为什么要这样，因为对象被偷了之后的状态可能也可能不是一个临时的状态。客户端程序员有可能会像重新使用这个被偷了的变量（非空）。或以某种方式对其进行测试，并可以据此进行计划。

In the above example, string str is set to the empty string after being moved (which is what std::string always does after a successful move). This allows us to reuse variable str if we wish (or we can ignore it, if we no longer have a use for it).

在上面的示例中，字符串str在移动后被设置为空字符串（这是成功移动后std :: string始终执行的操作）。 这使我们可以根据需要重用变量str（或者，如果不再使用它，则可以忽略它）。

### **Where else is std::move useful?**

std::move can also be useful when sorting an array of elements. Many sorting algorithms (such as selection sort and bubble sort) work by swapping pairs of elements. In previous lessons, we’ve had to resort to copy-semantics to do the swapping. Now we can use move semantics, which is more efficient.

std :: move在对元素数组进行排序时也很有用。 许多排序算法（例如选择排序和冒泡排序）通过交换元素对来工作。 在上一课中，我们不得不诉诸于复制语义来进行交换。 现在，我们可以使用更高效的移动语义。

It can also be useful if we want to move the contents managed by one smart pointer to another.

如果我们想将一个智能指针管理的内容移动到另一个智能指针，则它也很有用。

### **Conclusion**

std::move can be used whenever we want to treat an l-value like an r-value for the purpose of invoking move semantics instead of copy semantics.

只要我们想将l值视为r值，以调用移动语义而非复制语义，就可以使用std :: move。