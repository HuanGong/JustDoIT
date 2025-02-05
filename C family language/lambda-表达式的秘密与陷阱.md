# lambda 表达式的秘密与陷阱

简单示例:
---

```c++
#include <algorithm>
#include <cmath>
void abssort(float* x, unsigned n) {
    std::sort(x, x + n,
        // Lambda expression begins
        [](float a, float b) {
            return (std::abs(a) < std::abs(b));
        } // end of lambda expression
    );

}
```
定义:
---

盗取的微软的图🔫
![lambda图示](/MacIOS_development/img/IC251606.jpeg)

1. Capture 子句（在 C++ 规范中也称为 lambda 引导。）
2. 参数列表（可选）。 （也称为 lambda 声明符)
3. 可变规范（可选）。
4. 异常规范（可选）。
5. 尾随返回类型（可选）。
6. “lambda 体”

Lambda 可在其主体中引入新的变量（用 C++14），它还可以访问（或“捕获”）上下文范围内的变量。 Lambda 表达式以 Capture 子句（也就是`[]`）开头，它指定要捕获的变量以及是通过值还是引用进行捕获。 有与号 `(&)` 前缀的变量通过引用访问，没有该前缀的变量通过值访问。空 capture 子句 `[ ]` 指示 lambda 表达式的主体不访问封闭范围中的变量。

可以使用默认捕获模式（标准语法中的 capture-default）来指示如何捕获 lambda 中引用的任何外部变量：`[&]` 表示通过引用捕获引用的所有变量，而 `[=]` 表示通过值捕获它们。 可以使用默认捕获模式，然后为特定变量显式指定相反的模式。 例如，如果 lambda 体通过引用访问外部变量 total 并通过值访问外部变量 factor，则以下 capture 子句等效：

```c++
//下面所有的写法都表示 对变量`factor`使用值捕获， 对`total`使用应用捕获
[&total, factor]
[factor, &total]
[&, factor]
[factor, &]
[=, &total]
[&total, =]
```

lambda表达式的实现
---
函数对象；创造匿名类对象，重载`()`运算符；

在C++中， lambda表达式不是C++11才特有的，boost中就有它自己的实现；lambda表达式就像所有其他c++对象一样就是一个对象；在初始化一个lambda表达式时，就是在stack上创建的一个临时的匿名的对象
在c++11的spec中：
> A Note About Function Pointers

>Under the final C++11 spec, if you have a lambda with an empty capture specification, then it can be treated like a regular function and assigned to a function pointer. Here's an example of using a function pointer with a capture-less lambda:

所以这句话是成立的`auto lamb = []() {return 5;};` 这里的lamb就是stack object， 它有一个构造函数也有一个析构函数！和你创建的类一样；而所谓的capture list（捕获列表）就是dambda对象的成员变量而已；因为他是个对象，所以上面创造的lamb对象也可以赋值给 std::function (函数对象) `auto f_lamb = std::function<int()>(lamb)`; 而这里发生的其实是对lamb对象的一个拷贝；

std::function 对象在语义上是值类型`value semantics`；与`int， struct`一样；将一个函数对象赋值给另一个对象，是完完全全的copy，

下面是对C++11 spec的一个解析原文
> When the C++ compiler encounters a lambda expression, it generates a new anonymous class. Each captured variable becomes a member of that anonymous class, and the member is initialized from the variable in the outer scope. Finally, the anonymous class is given an operator() implementation whose parameter list is the parameter list of the lambda, whose body is the lambda body, and whose return value is the lambda return value.

我们可以看这样一个例子来说明这个问题：

```cpp
void A::SomeMethod() {
 int i = 0, j = 1;
 auto f = [this, i, &j](int k) -> int
    { return this->calc(i + j + k); };
 ...
}
//The compiler internally converts this to something like this:

void A::SomeMethod() {
 int i = 0, j = 1;

 // Autogenerated by the compiler
 class AnonymousClass$0 {
 public:
  AnonymousClass$0(A* this$, int i$, int& j$) :
   this$0(this$), i$0(i$), j$0(j$) { }
  int operator(int k) const
     { return this$0->calc(i$0 + j$0 + k); }
 private:
  A* this$0;			// this captured by value
  int i$0;                 // i captured by value
  int& j$0;                // j captured by reference
 };

 auto f = AnonymousClass$0(this, i, j);
 ...
}
```

上面说到的当没有捕获列表时， lambda表达式对象可以赋值给一个函数对象；据说在C++11的spec中并没有明确， 所以具体实现看具体编译器和平台的支持；具体有精力可以去查查spec文档； 而这种等效， 在lambda对象内部实现其事我们很容易的就想到只要在匿名类中如果做一个转换就可以；比如说：
```cpp
 class AnonymousClass$0
 {
 public:
  AnonymousClass$0()  { }
  operator int (__cdecl *)(int k) { return cdecl_static_function; }
  operator int (__stdcall *)(int k) { return stdcall_static_function; }
  operator int (__fastcall *)(int k) { return fastcall_static_function; }
  int operator(int k) { return cdecl_static_function(k); }
 private:
  static int __cdecl cdecl_static_function(int k) { return calc(k + 42); }
  static int __stdcall stdcall_static_function(int k) { return calc(k + 42); }
  static int __fastcall fastcall_static_function(int k) { return calc(k + 42); }
 };

 auto f = AnonymousClass$0();
```

陷阱
---

陷阱往往来自于我们自己的错误😊！ 虽然看着到处都在说什么任何语言都`图灵等价`, 当时语言各有千秋，我们不是语言的设计者， 难免在不精通的时候掉进陷阱里；因为cpp没有gc机制，也没有swift中｀weak，unowner｀这种默认的实现， 所以，如果你在用lambda表达式的时候；**当传入的是指针或者引用时，请一定一定要确保lambda表达式的函数体执行的时候，引用捕获的值没有被析构！！！** 不然你一定躺枪的;

reference:
---

- [lambda expressions](http://www.stroustrup.com/N1968-lambda-expressions.pdf)
- [demystifying-c-lambdas](https://blog.feabhas.com/2014/03/demystifying-c-lambdas/)



