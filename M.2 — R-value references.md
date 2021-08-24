# M.2 — R-value references

Way back in chapter 1, we mentioned l-values and r-values, and then told you not to worry that much about them. That was fair advice prior to C++11. But understanding move semantics in C++11 requires a re-examination of the topic. So let’s do that now.

理解移动语义需要对左值和右值的理解进行重新检验。

### **L-values and r-values**

Despite having the word “value” in their names, l-values and r-values are actually not properties of values, but rather, properties of expressions.

不要管左值和右值中的值这个字。左值和右值实际上不是值的属性，而是表达式的属性。

Every expression in C++ has two properties: a type (which is used for type checking), and a **value category** (which is used for certain kinds of syntax checking, such as whether the result of the expression can be assigned to). In C++03 and earlier, l-values and r-values were the only two value categories available.

在C++中的每个表达式都有两个属性，一个是类型，一个是**value category**， 在C++03之前，包括C++03, 左值和右值只有两个**value category**可用。

The actual definition of which expressions are l-values and which are r-values is surprisingly complicated, so we’ll take a simplified view of the subject that will largely suffice for our purposes.

表达式到底是左值还是右值的实际定义是非常复杂的，所以我们会简单看看，满足我们的目的就可以。

It’s simplest to think of an **l-value** (also called a locator value) as a function or an object (or an expression that evaluates to a function or object). All l-values have assigned memory addresses.

*简单的想法是认为左值（也叫作locator value）是一个函数或者是一个对象（或者是一个结果值为函数或对象的表达式）。所有的左值都有内存地址。*

When l-values were originally defined, they were defined as “values that are suitable to be on the left-hand side of an assignment expression”. However, later, the const keyword was added to the language, and l-values were split into two sub-categories: modifiable l-values, which can be changed, and non-modifiable l-values, which are const.

*当左值一开始被定义的时候，是那些可以放在赋值表达式左边的变量。然而，添加了const关键字之后，左值就被const分成了可以改变的左值和不可改变的左值（const）。*

It’s simplest to think of an **r-value** as “everything that is not an l-value”. This notably includes literals (e.g. 5), temporary values (e.g. x+1), and anonymous objects (e.g. Fraction(5, 2)). r-values are typically evaluated for their values, have expression scope (they die at the end of the expression they are in), and cannot be assigned to. This non-assignment rule makes sense, because assigning a value applies a side-effect to the object. Since r-values have expression scope, if we were to assign a value to an r-value, then the r-value would either go out of scope before we had a chance to use the assigned value in the next expression (which makes the assignment useless) or we’d have to use a variable with a side effect applied more than once in an expression (which by now you should know causes undefined behavior!).

*对右值最简单的想法是，把r-value认为是那些不能成为左值的所有东西。尤其包括字面量，临时值（x+1），还有匿名对象。右值被执行的时候，通常是计算它的值。他只有表达式作用于（在表达式的结束时死亡），也不能被赋值。此 non-assignment 很有意义，因为赋值会对对象产生副作用。因为右值有表达式作用域。如果我们把一个值赋给右值，右值也会在我们有机会使用它的时候走出作用域（使得这样的赋值没有意义）or we’d have to use a variable with a side effect applied more than once in an expression (which by now you should know causes undefined behavior!).*

In order to support move semantics, C++11 introduces 3 new value categories: pr-values, x-values, and gl-values. We will largely ignore these since understanding them isn’t necessary to learn about or use move semantics effectively. If you’re interested, [cppreference.com](https://en.cppreference.com/w/cpp/language/value_category) has an extensive list of expressions that qualify for each of the various value categories, as well as more detail about them.

为了支持移动语义，C++引入了三个新的value 类型：pr-values, x-values, and gl-values. 我们会忽略掉这些东西，因为这些对于移动语义的学习不是必要的。你有兴趣，去cppreference看。

### **L-value references**

Prior to C++11, only one type of reference existed in C++, and so it was just called a “reference”. However, in C++11, it’s sometimes called an l-value reference. L-value references can only be initialized with modifiable l-values.

在C++11之前，只有一种类型的引用，所以名字就叫reference了。然而，在C++11中，有时候被叫做l-valuerefence。l-valuereference只能被可改变的左值来进行初始化（前面用const割裂了左值，割成了可改变的和不可改变的）

| L-value reference       | Can be initialized with | Can modify |
| :---------------------- | :---------------------- | :--------- |
| Modifiable l-values     | Yes                     | Yes        |
| Non-modifiable l-values | No                      | No         |
| R-values                | No                      | No         |

L-value references to const objects can be initialized with l-values and r-values alike. However, those values can’t be modified.

对const对象左值引用可以用做左值和右值来进行初始化，但是不能修改。

| L-value reference to const | Can be initialized with | Can modify |
| :------------------------- | :---------------------- | :--------- |
| Modifiable l-values        | Yes                     | No         |
| Non-modifiable l-values    | Yes                     | No         |
| R-values                   | Yes                     | No         |

L-value references to const objects are particularly useful because they allow us to pass any type of argument (l-value or r-value) into a function without making a copy of the argument.

对const对象的左值引用（const T&）是非常有用的，因为这允许我们传递任何类型的参数（左值或者右值）给函数，而不需要做拷贝。

### **R-value references**

C++11 adds a new type of reference called an r-value reference. An r-value reference is a reference that is designed to be initialized with an r-value (only). While an l-value reference is created using a single ampersand, an r-value reference is created using a double ampersand:

C++增加了一个新的引用类型，叫右值引用。右值引用是设计来只被右值初始化的引用。左值引用使用单个的&，而右值引用使用两个&。

```
int x{ 5 };
int &lref{ x }; // l-value reference initialized with l-value x
int &&rref{ 5 }; // r-value reference initialized with r-value 5
```

R-values references cannot be initialized with l-values.

右值引用不能被左值初始化

| R-value reference       | Can be initialized with | Can modify |
| :---------------------- | :---------------------- | :--------- |
| Modifiable l-values     | No                      | No         |
| Non-modifiable l-values | No                      | No         |
| R-values                | Yes                     | Yes        |

| R-value reference to const | Can be initialized with | Can modify |
| :------------------------- | :---------------------- | :--------- |
| Modifiable l-values        | No                      | No         |
| Non-modifiable l-values    | No                      | No         |
| R-values                   | Yes                     | No         |

R-value references have two properties that are useful. First, r-value references extend the lifespan of the object they are initialized with to the lifespan of the r-value reference (l-value references to const objects can do this too). Second, non-const r-value references allow you to modify the r-value!

右值引用有两个有用的属性。第一个，右值引用延展了对象的生命期，把生命周期变成了右值引用的生命周期（对const对象左值引用也可以这样），第二件事情是non-const右值引用允许你修改一个右值（匿名对象）。

Let’s take a look at some examples:

```
#include <iostream>
 
class Fraction
{
private:
	int m_numerator;
	int m_denominator;
 
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
	auto &&rref{ Fraction{ 3, 5 } }; // r-value reference to temporary Fraction
	
    // f1 of operator<< binds to the temporary, no copies are created.
    std::cout << rref << '\n';
 
	return 0;
} // rref (and the temporary Fraction) goes out of scope here
```

This program prints:

```
3/5
```

As an anonymous object, Fraction(3, 5) would normally go out of scope at the end of the expression in which it is defined. However, since we’re initializing an r-value reference with it, its duration is extended until the end of the block. We can then use that r-value reference to print the Fraction’s value.

作为一个匿名对象，通常会在定义的表达式的计算结束的时候没有掉，然而因为我们把它初始化给了一个右值引用，它的生命期就延长至当前这个块结束。我们可以用右值引用来打印这个匿名对象的值。

Now let’s take a look at a less intuitive example:

```
#include <iostream>
 
int main()
{
    int &&rref{ 5 }; // because we're initializing an r-value reference with a literal, a temporary with value 5 is created here
    rref = 10;
    std::cout << rref << '\n';
 
    return 0;
}
```

This program prints:

```
10
```

While it may seem weird to initialize an r-value reference with a literal value and then be able to change that value, when initializing an r-value with a literal, a temporary is constructed from the literal so that the reference is referencing a temporary object, not a literal value.

用一个右值字面量来初始化一个右值引用，随后修改他，看起来有点奇怪。实际上是制造了一个临时变量，然后这个引用指向这个临时对象而不是字面量。

R-value references are not very often used in either of the manners illustrated above.

右值引用通常不会像上面那样使用。

### **R-value references as function parameters**

R-value references are more often used as function parameters. This is most useful for function overloads when you want to have different behavior for l-value and r-value arguments.

右值引用经常用来做函数参数。当您想对左值和右值参数具有不同的行为时就用到这个东西了。

```
void fun(const int &lref) // l-value arguments will select this function
{
	std::cout << "l-value reference to const\n";
}
 
void fun(int &&rref) // r-value arguments will select this function
{
	std::cout << "r-value reference\n";
}
 
int main()
{
	int x{ 5 };
	fun(x); // l-value argument calls l-value version of function
	fun(5); // r-value argument calls r-value version of function
 
	return 0;
}
```

This prints:

```
l-value reference to const
r-value reference
```

As you can see, when passed an l-value, the overloaded function resolved to the version with the l-value reference. When passed an r-value, the overloaded function resolved to the version with the r-value reference (this is considered a better match than a l-value reference to const).

正如你所看到的，传一个左值的时候，调用左值的重载函数。传入右值时，调用接受右值的函数。（右值引用（T&&）会优先于对const的左值引用（const T&）被选择）

Why would you ever want to do this? We’ll discuss this in more detail in the next lesson. Needless to say, it’s an important part of move semantics.

我们会在下一节讲你为什么会想这样做，不言自明，它是移动语义的重要组成部分（前面不是留了一个没办法判断到底是移动还是复制的问题吗，现在对于临时量可以移动，对于左值可以拷贝，可以区别对待了！）。

One interesting note:

```c++
	int &&ref{ 5 };
	fun(ref);
```

actually calls the l-value version of the function! Although variable ref has type *r-value reference to an integer*, it is actually an l-value itself (as are all named variables). The confusion stems from the use of the term r-value in two different contexts. Think of it this way: Named-objects are l-values. Anonymous objects are r-values. The type of the named object or anonymous object is independent from whether it’s an l-value or r-value. Or, put another way, if r-value reference had been called anything else, this confusion wouldn’t exist.

*这里需要注意的是，上面的fun会调用左值版本的函数，尽管ref是一个右值引用。但它实际上是一个左值。混淆源于在两种不同情况下使用右值。 你应该这样想：有名字的对象是左值。 没有名字的对象是右值。这有没有名字的类型跟它是左值还是右值没有关系（要结合上面的代码理解，ref是一个右值引用，但是它有名字，所以它不是右值）。或者换句话说，如果一个右值引用在别的地方被调用，它就变成一个左值。*

### **Returning an r-value reference**

You should almost never return an r-value reference, for the same reason you should almost never return an l-value reference. In most cases, you’ll end up returning a hanging reference when the referenced object goes out of scope at the end of the function.

你基本上不应返回一个右值引用，同样的，你也不应该返回任何左值引用。大部分情况下，当函数结束的时候被引用的对象走出语句块，然后你从函数得到一个悬挂引用。