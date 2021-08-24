# M.5 — std::move_if_noexcept

(h/t to reader Koe for providing the first draft of this lesson!)

In lesson [20.9 -- Exception specifications and noexcept](https://www.learncpp.com/cpp-tutorial/exception-specifications-and-noexcept/), we covered the `noexcept` exception specifier and operator, which this lesson builds on.

前面讲到的异常规范和noexcept是本节内容的基础。

We also covered the `strong exception guarantee`, which guarantees that if a function is interrupted by an exception, no memory will be leaked and the program state will not be changed. In particular, all constructors should uphold the strong exception guarantee, so that the rest of the program won’t be left in an altered state if construction of an object fails.

我们在前面也提到了`strong exception guarantee`，保证了如果函数因为异常而终止，不会有内存泄露，而且整个程序的状态也不会被改变。特别的，所有的构造函数都应该坚持`strong exception guarantee`，这样的我们程序就不会因为构造失败而发生一些不期望的变化。

### The move constructors exception problem

Consider the case where we are copying some object, and the copy fails for some reason (e.g. the machine is out of memory). In such a case, the object being copied is not harmed in any way, because the source object doesn’t need to be modified to create a copy. We can discard the failed copy, and move on. The `strong exception guarantee` is upheld.

假设我们正在拷贝一些对象，而且拷贝失败了（没有可用内存了）。在这样的情况下，被拷贝的对象是安全的，因为他没被修改任何东西。我们可以放弃掉失败的拷贝，然后继续执行。`strong exception guarantee`在这里被保证了！

Now consider the case where we are instead moving an object. A move operation transfers ownership of a given resource from the source to the destination object. If the move operation is interrupted by an exception after the transfer of ownership occurs, then our source object will be left in a modified state. This isn’t a problem if the source object is a temporary object and going to be discarded after the move anyway -- but for non-temporary objects, we’ve now damaged the source object. To comply with the `strong exception guarantee`, we’d need to move the resource back to the source object, but if the move failed the first time, there’s no guarantee the move back will succeed either.

现在思考一下我们如果不是在做拷贝，而是做移动。一个移动操作会转移给定资源的所有权。如果一个移动操作在交出所有权之后，被异常中断。那么我们的对象就会处于一个被修改的状态。如果source对象是一个临时对象的话，这不是什么问题。但是如果不是临时对象的话，我们现在就损坏了这个对象了。为了遵守storng exception guarrantee，我们需要把从source拿出来的资源给他送回去，但是如果移动失败了的话，移动回去也不一定能成功呀。

How can we give move constructors the `strong exception guarantee`? It is simple enough to avoid throwing exceptions in the body of a move constructor, but a move constructor may invoke other constructors that are `potentially throwing`. Take for example the move constructor for `std::pair`, which must try to move each subobject in the source pair into the new pair object.

我们怎么给move constructor一个强的异常保证呢？很简单，只要避免在移动构造函数体里面抛出异常就行。但其实移动构造函数可能会调用其他潜在地要抛出异常的构造函数，举个例子，这个pair的移动构造函数，必须尝试把source pair里面的子对象移动到新pair对象里面。

```
// Example move constructor definition for std::pair
// Take in an 'old' pair, and then move construct the new pair's 'first' and 'second' subobjects from the 'old' ones
template <typename T1, typename T2>
pair<T1,T2>::pair(pair&& old)
  : first(std::move(old.first)),
    second(std::move(old.second))
{}
```

Now lets use two classes, `MoveClass` and `CopyClass`, which we will `pair` together to demonstrate the `strong exception guarantee` problem with move constructors:

我们现在用两个类来展示一下与移动构造函数相关的`strong exception guarantee`问题。

```c++
#include <iostream>
#include <utility> // For std::pair, std::make_pair, std::move, std::move_if_noexcept
#include <stdexcept> // std::runtime_error
#include <string>
#include <string_view>
 
class MoveClass
{
private:
  int* m_resource{};
 
public:
  MoveClass() = default;
 
  MoveClass(int resource)
    : m_resource{ new int{ resource } }
  {}
 
  // Copy constructor
  MoveClass(const MoveClass& that)
  {
    // deep copy
    if (that.m_resource != nullptr){
      m_resource = new int{ *that.m_resource };
    }
  }
 
  // Move constructor
  MoveClass(MoveClass&& that) noexcept
    : m_resource{ that.m_resource }{
    that.m_resource = nullptr;
  }
 
  ~MoveClass(){
    std::cout << "destroying " << *this << '\n';
 
    delete m_resource;
  }
 
  friend std::ostream& operator<<(std::ostream& out, const MoveClass& moveClass){
    out << "MoveClass(";
 
    if (moveClass.m_resource == nullptr){
      out << "empty";
    }
    else{
      out << *moveClass.m_resource;
    }
 
    out << ')';
    
    return out;
  }
};
class CopyClass
{
public:
  bool m_throw{};
 
  CopyClass() = default;
 
  // Copy constructor throws an exception when copying from a CopyClass object where its m_throw is 'true'
  CopyClass(const CopyClass& that)
    : m_throw{ that.m_throw }
  {
    if (m_throw)
    {
      throw std::runtime_error{ "abort!" };
    }
  }
};
 
int main()
{
  // We can make a std::pair without any problems:
  std::pair my_pair{ MoveClass{ 13 }, CopyClass{} };
 
  std::cout << "my_pair.first: " << my_pair.first << '\n';
  // But the problem arises when we try to move that pair into another pair.
  try
  {
    my_pair.second.m_throw = true; // To trigger copy constructor exception
    // The following line will throw an exception
    std::pair moved_pair{ std::move(my_pair) }; // We'll comment out this line later
    // std::pair moved_pair{std::move_if_noexcept(my_pair)}; // We'll uncomment this line later
    std::cout << "moved pair exists\n"; // Never prints
  }
  catch (const std::exception& ex)
  {
      std::cerr << "Error found: " << ex.what() << '\n';
  }
  std::cout << "my_pair.first: " << my_pair.first << '\n';
  return 0;
}
```

The above program prints:

```
destroying MoveClass(empty)
my_pair.first: MoveClass(13)
destroying MoveClass(13)
Error found: abort!
my_pair.first: MoveClass(empty)
destroying MoveClass(empty)
```

Let’s explore what happened. The first printed line shows the temporary `MoveClass` object used to initialize `my_pair` gets destroyed as soon as the `my_pair` instantiation statement has been executed. It is `empty` since the `MoveClass` subobject in `my_pair` was move constructed from it, demonstrated by the next line which shows `my_pair.first` contains the `MoveClass` object with value `13`.

我们看一下发生了什么，打印出来的第一行显示了一个MoveClass的临时对象初始化了my_pair之后，因为使用了移动语义，临时对象在走出作用域的时候，死掉了，打印了一条消息。然后第二行展示了mypair里面的MoveClass对象。

It gets interesting in the third line. We created `moved_pair` by copy constructing its `CopyClass` subobject (it doesn’t have a move constructor), but that copy construction threw an exception since we changed the Boolean flag. Construction of `moved_pair` was aborted by the exception, and its already-constructed members were destroyed. In this case, the `MoveClass` member was destroyed, printing `destroying MoveClass(13) variable`. Next we see the `Error found: abort!` message printed by `main()`.

第三行开始变得有意思，我们创建了一个通过拷贝构造的方式来构造movedpair的拷贝子对象（因为他没有一个移动构造函数）。但是我们在这一行代码之前修改了布尔值。movedpair因为异常而构造失败，而且已经构造好的成员也销毁掉了，在这个情况下，MoveClass的成员被销毁掉了，所以打印出来了第三行，这是在异常抛出之前的对象的死亡处理，在这之后异常抛出，catch抓到之后打印了第四行输出。

When we try to print `my_pair.first` again, it shows the `MoveClass` member is empty. Since `moved_pair` was initialized with `std::move`, the `MoveClass` member (which has a move constructor) got move constructed and `my_pair.first` was nulled.

当我们再次尝试打印mypair.first的时候，他打印出来MoveClass成员是空的。因为movedpair是用移动语义来构造的，MoveClass成员被移到了movedpair里面之后被销毁掉了，所以my_pair.first已经变成空了

Finally, `my_pair` was destroyed at the end of main().

最后，my_pairt在最后被销毁（注意这里movedpair没有在退出try块的时候调用析构函数，因为它就没有被成功构造）。

To summarize the above results: the move constructor of `std::pair` used the throwing copy constructor of `CopyClass`. This copy constructor throw an exception, causing the creation of `moved_pair` to abort, and `my_pair.first` to be permanently damaged. The `strong exception guarantee` was not preserved.

总结一下：这移动语义的pair构造使用了抛出异常的拷贝构造函数，这个拷贝构造函数抛出了一个异常，导致movedpair创建失败，而且mypair的first成员也被永远的删除掉了。`strong exception guarantee`没能保证。

### std::move_if_noexcept to the rescue

Note that the above problem could have been avoided if `std::pair` had tried to do a copy instead of a move. In that case, `moved_pair` would have failed to construct, but `my_pair` would not have been altered.

上面的问题可以通过pair的拷贝构造来解决，因为如果是拷贝的话，就算拷贝失败也不会影响被拷贝的值。

But copying instead of moving has a performance cost that we don’t want to pay for all objects -- ideally we want to do a move if we can do so safely, and a copy otherwise.

但是拷贝不是我们想支付的代价。我们想优先安全的进行移动，不行的话才做拷贝

Fortunately, C++ has a two mechanisms that, when used in combination, let us do exactly that. First, because `noexcept` functions are no-throw/no-fail, they implicitly meet the criteria for the `strong exception guarantee`. Thus, a `noexcept` move constructor is guaranteed to succeed.

幸运的是，C++提供了两个机制。结合使用的时候，我们就能达到上面提出的目标。首先因为noexcept函数是不抛出不失败的函数，他们的潜台词就是说他们自己是强异常安全保证的。因此一个noexcept移动构造函数是确保能成功的。

Second, we can use the standard library function `std::move_if_noexcept()` to determine whether a move or a copy should be performed. `std::move_if_noexcept` is a counterpart to `std::move`, and is used in the same way.

然后，我们要使用标准库函数，move_if_noexcept，来决定到底做一个移动操作还是拷贝操作，它的用法和move的用法是一样的。

If the compiler can tell that an object passed as an argument to `std::move_if_noexcept` won’t throw an exception when it is move constructed (or if the object is move-only and has no copy constructor), then `std::move_if_noexcept` will perform identically to `std::move()` (and return the object converted to an r-value). Otherwise, `std::move_if_noexcept` will return a normal l-value reference to the object.

如果编译器能知道在移动构造的时候，一个对象传到参数里面的时候它不会抛出异常，那么move_if_noexcept和move就表现一样。否则的话，move_if_noexcept就会返回一个左值给这个参数。**（这意味着需要考虑默认的异常类型的判断）**.

```
Key insight

std::move_if_noexcept will return a movable r-value if the object has a noexcept move constructor, otherwise it will return a copyable l-value. We can use the noexcept specifier in conjunction with std::move_if_noexcept to use move semantics only when a strong exception guarantee exists (and use copy semantics otherwise).
```

Let’s update the code in the previous example as follows:

我们更新一下前面的代码

```
//std::pair moved_pair{std::move(my_pair)}; // comment out this line now
std::pair moved_pair{std::move_if_noexcept(my_pair)}; // and uncomment this line
```

Running the program again prints:

然后结果如下

```
destroying MoveClass(empty)
my_pair.first: MoveClass(13)
destroying MoveClass(13)
Error found: abort!
my_pair.first: MoveClass(13)
destroying MoveClass(13)
```

As you can see, after the exception was thrown, the subobject `my_pair.first` still points to the value `13`.

你可以看到，在异常抛出之后，mypair的first仍然指向数字13. （没有被移动）

`CopyClass` doesn’t have a `noexcept` move constructor, so `std::move_if_noexcept` returns `my_pair` as an l-value reference. This causes `moved_pair` to be created via the copy constructor (rather than the move constructor). The copy constructor can throw safely, because it doesn’t modify the source object.

因为CopyClass没有一个noexcept的移动构造函数。所以move_if_noexcept返回了一个左值my_pair的左值引用。这导致movedpair是通过拷贝构造的，而不是通过移动构造函数构造的。拷贝构造函数可以安全的抛出异常，因为这不会影响被拷贝的对象（my_pair)。

The standard library uses `std::move_if_noexcept` often to optimize for functions that are `noexcept`. For example, `std::vector::resize` will use move semantics if the element type has a `noexcept` move constructor, and copy semantics otherwise. This means `std::vector` will generally operate faster with objects that have a `noexcept` move constructor (reminder: move constructors are `noexcept` by default, unless they call a function that is `noexcept(false)`).

标准库经常使用move_if_noexcept来优化那些noexcept函数。举个例子，如果元素的类型有一个noexcept的移动构造函数的话，vector::resize会使用移动语义，否则它就使用拷贝构造函数。这意味vector在那些实现了noexcept移动构造函数的类表现的更快。（注意：移动构造函数默认是noexcept的，除非它调用了一个noexcept(false)的函数。

```
Warning

If a type has both potentially throwing move semantics and deleted copy semantics (the copy constructor and copy assignment operator are unavailable), then std::move_if_noexcept will waive the strong guarantee and invoke move semantics. This conditional waiving of the strong guarantee is ubiquitous in the standard library container classes, since they use std::move_if_noexcept often.

如果一个类型的move构造函数和拷贝赋值运算有可能抛出异常，而且这个类型还删掉了拷贝构造函数和拷贝赋值运算符，move_if_noexcept就会放弃异常安全的保证。然后使用移动语义的构造或赋值行为（因为拷贝语义已经被删掉了呀，没得选）。 在标准库容器类中，有条件地放弃强力保证是无处不在的，因为它们经常使用std::move_if_noexcept。
```

