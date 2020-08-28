# C++11
奇怪的知识增加了！！！
## Table of contents
1. [设计目标](#Chapter1)
2. [long long整型](#Chapter2)
3. [快速初始化成员变量](#Chapter3)
4. [右尖括号>的改进](#Chapter4)
5. [右值引用及移动语义](#Chapter5)
    1. [移动构造函数](#Chapter5.1)
    2. [左值、右值与右值引用](#Chapter5.2)
    3. [std::move](#Chapter5.3)
6. [smart pointers(智能指针)](#Chapter6)
    1. [unique_ptr和shared_ptr](#Chapter6.1)
    2. [static type and dynamic type](#Chapter6.2)
7. [基于范围的for循环](#Chapter7)
8. [auto类型推导](#Chapter8)
9. [decltype与typeid](#Chapter9)
10. [nullptr](#Chapter10)
11. [lambda函数](#Chapter11)
12. [一些有用的特性](#Chapter12)
    1. [std::array](#Chapter12.1)
    2. [std::optional](#Chapter12.2)

<a name="Chapter1"></a>
## 设计目标
C++11的整体设计目标如下：
1. 使得C++成为更好的适用于系统开发及库开发的语言。
2. 使得C++成为更易于教学的语言（语法更加一致和简单）。
3. 保证语言的稳定性，以及和C++03及C语言的兼容性。

尽管C++11的一个设计目标是倾向于使用库而不是扩展语言来实现特性，但标准仍然增加了一些关键词或者扩充了一些核心语法特性。

<a name="Chapter2"></a>
## long long整型
标准要求long long整型可以在不同平台上有不同的长度，但至少有64位。我们在写常数字面量时，可以用LL或者ll标识一个long long类型的字面量。比如：

```cpp
long long a = -90000000000000000LL;
```

同其他的整型一样，可以用<climits>中定义的LLONG_MIN，LLONG_MAX和ULLONG_MIN查看平台上long long的表示范围。

<a name="Chapter3"></a>
## 快速初始化成员变量
C++11允许使用等号=或者花括号\{\}进行**就地**的非静态成员变量初始化。例如：

```cpp
class Person {
public:
	Person() {}
	Person(int a) : age(a) {}
	int getAge() { return age; }

private:
	int age = 0;
	string name{"unnamed"};
};

int main() {
	Person p;
	cout << p.getAge(); // output 0
	Person ljl(18);
	cout << ljl.getAge(); // output 18
	return 0;
}
```

注意到，同时使用初始化列表和就地初始化并不冲突，但是初始化列表的效果后生效（或理解为优先级更高），因此上述程序第二个输出为18.

对于非常量的静态成员变量，例如static int a，C++11与C++98保持了一致，程序员还是需要到头文件以外去定义它，这会保证编译时，类静态成员的定义最后只存在于一个目标文件中。

<a name="Chapter4"></a>
## 右尖括号>的改进
在C++98中，有一条令人头疼的规则：如果在实例化模板的时候出现了连续的两个右尖括号>，那么它们之间需要一个空格来分隔。例如：

```cpp
vector<vector<int>> vec; // 在C++98标准下编译失败
```

C++98会将>>优先解析为右移；在C++11中，这种限制被取消了，它要求编译器智能地去判断在哪些情况下>>不是右移符号。

有趣的是，这个改进引起了C++11与C++98的一点小小的不兼容性，例如：

```cpp
template <int i> class X {};
X<1 >> 5> x;
```

对于上述代码，C++98会认为连续的右尖括号是位移符号，因此结果是一个形如X<0> x的模板实例。而C++11会优先将遇到的第一个>与此前的<配对，导致编译错误。避免的方式是用圆括号将"1>>5"括起来。

<a name="Chapter5"></a>
## 右值引用：移动语义
为了讲清楚右值引用以及移动语义等概念的实际意义，我们先用一些典型场景来引出C++编程中的一些实际问题。

考虑如下一段代码：

```cpp
int *p = new int(42);
int *q = p;
delete p;
delete q; // ouch!
```

这段代码中，p和q两个指针指向同一个地址。对p所指内存进行释放后，q指向的是一个非法地址（此时q成为一个“悬挂指针”，即dangling pointer），因此不能再对其调用delete。

因为这个原因，我们在编写自定义类的时候，往往要很小心地编写拷贝构造函数。例如，以下代码就会导致错误。

```cpp
class HasPtrMem {
public:
    HasPtrMem() : d(new int(0)) {}
    HasPtrMem(const HasPtrMem & h) : d(h.d) {}
    ~HasPtrMem() { delete d; }
    int *d;
};

int main() {
    HasPtrMem a;
    HasPtrMem b(a);
}
```

在main作用域结束时，a和b的析构函数纷纷被调用，其中之一完成析构之后，另一个对象的d指针就会变成悬挂指针。在该悬挂指针上释放内存就会造成错误。

如果我们不定义拷贝构造函数，C++也会为类生成一个类似的构造函数。这种拷贝称为**浅拷贝（shallow copy）**。通常最佳的解决方案是用户自定义拷贝构造函数来实现**深拷贝（deep copy）**，如下所示。

```cpp
class HasPtrMem {
public:
    HasPtrMem() : d(new int(0)) {}
    HasPtrMem(const HasPtrMem & h) : d(new int(*h.d)) {}
    ~HasPtrMem() { delete d; }
    int *d;
};

HasPtrMem getTemp() { return HasPtrMem(); }

int main() {
    HasPtrMem a = getTemp();
    return 0;
}
```

深拷贝解决了不同指针指向同一个内存区域的问题，但如果指针d指向的是一个非常大的堆内存数据，它也将带来巨大的复制开销。例如，在上面的代码中，拷贝构造函数被调用了两次，一次是从getTemp函数中拷贝构造出一个**临时值**，以用作getTemp的返回值，另一次是由临时值构造出main中变量a调用的。

C++因此使用了**移动语义**来避免构造临时值时多余的堆上数据的拷贝。

<a name="Chapter5.1"></a>
### 移动构造函数
我们用如下方式实现HasPtrMem类的移动构造函数。

```cpp
class HasPtrMem {
public:
    HasPtrMem() : d(new int(0)) {}
    HasPtrMem(const HasPtrMem & h) : d(new int(*h.d)) {}
    HasPtrMem(HasPtrMem && h) : d(h.d) {
    	h.d = nullptr;
    }
    ~HasPtrMem() { delete d; }
    int *d;
};
```

移动构造函数以一个**右值引用**作为参数，关于右值引用我们稍后将介绍，现在可以把它简单地看作对于临时值的一个引用。注意，在函数体中我们将右值引用的d指针设置为空指针nullptr，因此当临时值被析构时，delete nullptr不会有任何效果，而堆上的数据，已经被“移交”给了构造的对象的指针。

接下来我们要回答几个问题：移动构造函数何时被触发？C++如何判断临时对象的产生？是否只有临时对象可以用于移动构造？为了回答这些问题，我们先介绍C++中值的分量，以及右值引用的概念。

<a name="Chapter5.2"></a>
### 左值、右值与右值引用
在C语言中，要判断**左值（lvalue）**和**右值（rvalue）**，一个典型的方法就是，在赋值表达式中，等号左边的是左值，右边的是右值。例如a = b + c中，a是左值，b + c 是一个右值。

一个更被广泛认可的说法是，可以取地址的、有名字的是左值，不能取地址的、没有名字的就是右值。

在C++11中，右值由两个概念构成，一个是将亡值（xvalue, eXpring Value），一个是纯右值（prvalue, Pure Rvalue）。临时变量（例如函数返回的）和不跟对象关联的字面量值，也就是C++98中右值的概念，称为纯右值。将亡值是C++11新增的根右值引用相关的表达式，例如将要被移动的对象（std::move的返回值）。所有的值必属于左值、纯右值、将亡值之一。

**右值引用**就是对一个右值进行引用的类型。假设ReturnRvalue()返回一个右值，那么如下代码则声明了一个名为a的右值引用。

```cpp
T && a = ReturnRvalue();
```

为了区别，C++98的引用称为**左值引用**。无论左值引用还是右值引用，都必须立即进行初始化。左值引用是具名变量值的别名，而右值引用是不具名变量的别名。

之所以引入右值引用，是因为我们希望在移动语义中改变所引用对象的属性，例如将指针设为nullptr。如果是左值引用，我们通常希望保留其拥有的内存，因此不会修改所引用对象的属性，而临时量的内存我们是可以“窃为己用”的。


<a name="Chapter5.3"></a>
### std::move
标准库<utility>提供了一个有用的函数std::move，该函数将一个左值强制转化为右值引用。结合起来，我们看一个正确使用移动语义的例子。

```cpp
#include <iostream>
using namespace std;

class HugeMem {
public:
    HugeMem(int size) : sz(size > 0 ? size : 1) {
        c = new int[sz];
    }
    ~HugeMem() { delete [] c; }
    HugeMem(HugeMem && hm) : sz(hm.sz), c(hm.c) {
        hm.c = nullptr;
    }
    int *c;
    int sz;
};

class Moveable {
public:
    Moveable() : i(new int(3)), h(1024) {}
    ~Moveable() { delete i; }
    Moveable(Moveable && m) : i(m.i), h(move(m.h)) {
        m.i = nullptr;
    }
    int *i;
    HugeMem h;
};

Moveable getTemp() {
    Moveable temp = Moveable();
    cout << hex << "Huge Mem from " << __func__ << " @" << temp.h.c << endl;
    return temp;
}

int main() {
    Moveable a(getTemp());
    cout << hex << "Huge Mem from " << __func__ << " @" << a.h.c << endl;
}
```

程序输出:
Huge Mem from getTemp @0000023564BC2410
Huge Mem from main @0000023564BC2410

在Moveable的移动构造函数中，我们使用了std::move，这里因为m是临时变量，即将被析构，所以对m.h进行move之后，我们也不存在还会去访问m.h的可能，因此从生存期的角度不会存在依然可以访问到的非法对象。

<a name="Chapter6"></a>
## smart pointers(智能指针)
C++11用智能指针来自动回收堆分配的对象。

<a name="Chapter6.1"></a>
### unique_ptr和shared_ptr
unique_ptr独自占有所指的堆内存空间，而shared_ptr可以与其他shared_ptr共享所指的内存空间。所谓独享，也就是Resource Acquisition Is Initialization(RAII)，该技术保证对象生存期内，它所占有的资源可以被访问，并且在对象生存期结束后释放它占用的所有资源。利用smart pointers，程序员不需要显示地调用delete。几个例子如下：

```cpp
std::unique_ptr<float> x = std::make_unique<float>(0.f);
std::shared_ptr<float> y = std::make_shared<float>(5.f);
std::shared_ptr<float> z = std::shared_ptr<float>(y); // call copy constructor
std::unique_ptr<float> x2 = std::move(x); // now x does not own that memory
```

其中，make_unique是C++14提供的。可以看到，对于unique_ptr所指的内存空间，始终只允许一个unique_ptr指向该地址，若要将所有权赋予另一个unique_ptr，我们可以用前面提到的std::move函数来移动。我们在构造smart pointer时，参数是模板类的构造函数的参数列表。smart pointer只能指向堆上的内存地址。

我们可以使用\*操作符对指针进行dereference，，也可以用->操作符访问所指对象的成员。此外，我们可以通过smart pointer的get函数获取一个指向同一个堆内存的普通指针。例子：

```cpp
class Person {
public:
    Person() : age(0), name("unnamed") {}
    Person(int _age, string _name) : age(_age), name(_name) {}
    int getAge() const { return age; }
    string name;
private:
    int age;
};

int main() {
    std::unique_ptr<Person> sp = std::make_unique<Person>(23, "Jason");
    cout << sp->getAge() << endl;
    cout << sp->name << endl;
    Person *p = sp.get();
    cout << p->name << endl;
}
```

对于通过get函数返回的普通指针，我们唯一不能做的就是delete，因为该内存地址应由smart pointer管理。

<a name="Chapter6.2"></a>
### static type and dynamic type
我们可以写出如下代码来支持多态

```cpp
class Geometry {};
class Square : public Geometry {};
std::unique_ptr<Geometry> x = std::make_unique<Square>(); // Geometry is a static type
                                                          // Square is a dynamic type
```

我们还可以用下列方式，对shared_ptr进行类型转换:

```cpp
static_pointer_cast<T>
```
Cast a pointer to a sub-class into a pointer to a super-class. We cast to the “static” type of the class.
```cpp
dynamic_pointer_cast<T>
```
Cast a pointer to a super-class into a pointer to a sub-class. We cast to the “dynamic” type of the class.
```cpp
reinterpret_pointer_cast<T>
```
Cast a pointer to any type, even unrelated ones.

对于普通的指针类型，我们使用dynamic_cast<T*>和static_cast<T*>。例如一个典型场景中，我们要决定某个对象的运行时类型，则可以将它转换为对应类型，如果转换结果不为nullptr，则表示它确实是这个类型，如下所示：

```cpp
Child *c = dynamic_cast<Child*>(node);
if (c != nullptr) {
    // ...
}
```

<a name="Chapter7"></a>
## 基于范围的for循环
C++11引入了基于范围的for循环，例如：
```cpp
vector<int> vec = {1, 2, 3, 4, 5};
for (int &a : vec) {
    a += 10;
}
for (auto a : vec) {
    cout << a << endl;
}
```

是否能使用基于范围的for循环，必须依赖一些条件。首先，for循环迭代的范围是确定的，对于类来说需要有begin和end函数，而对于数组而言需要第一个和最后一个元素间的范围确定。其次，基于范围的for循环还要求迭代的对象实现++和==操作符。对于标准库中的容器，如string, array, vector, deque, list, queue, map, set等，不会有问题。

<a name="Chapter8"></a>
## auto类型推导
C++11提供了简单易用的auto，进行类型推导。我们通过下面的例子看看auto类型推导的基本用法。
```cpp
int main() {
    double foo();
    auto x = 1; // x的类型为int
    auto y = foo(); // y的类型为double
    struct m { int i; }str;
    auto str1 = str; // str1的类型是struct m
    
    auto z; // 无法推导，无法通过编译
    z = x;
}
```

auto并非一种“类型”声明，而是一个类型声明时的“占位符”，编译器在编译时将auto替代为变量实际的类型。

auto的一大优势在于简化代码，在前面的部分我们看到了它可以在基于范围的for循环内，推导出容器中每个对象的类型。另外，当我们使用某些容器的迭代器时，我们往往要写很冗长的代码，而用auto则可以简化这种代码。如下所示。

```cpp
void printStringList(std::list<std::string> &sl) {
    std::list<std::string>::iterator it = sl.begin();
    while (it != sl.end()) {
    	std::cout << *it << std::endl;
    	it++;
    }
    
    // instead
    for (auto it = sl.begin(); it != sl.end(); it++) {
        std::cout << *it << std::endl;
    }
}
```

事实上，auto只是C++11中类型推导的一部分，其余的则会在decltype中得到体现。

<a name="Chapter9"></a>
## decltype与typeid
我们可以在程序中使用typeid随时查询一个变量的类型，typeid就会返回变量相应的type_info数据。type_info的name()会返回类的名字，而hash_code()会返回该类型唯一的哈希值，以供我们对变量的类型进行比较。

```cpp
#include <iostream>
#include <typeinfo>

using namespace std;

class White {};
class Black {};

int main() {
    White a;
    Black b;
    cout << typeid(a).name() << endl;
    cout << typeid(b).name() << endl;    
    cout << "a and b same type? " << (typeid(a).hash_code() == typeid(b).hash_code()) << endl;
}
```

与auto相同的是，decltype也可以用于类型推导，并且可以将获得的类型用以定义另外一个变量。例如：

```cpp
#include <iostream>
#include <typeinfo>
using namespace std;

int main() {
    int i;
    decltype(i) j = 0;
    cout << typeid(j).name() << endl; // g++打印出"i"
    
    float a;
    double b;
    decltype(a + b) c;
    cout << typeid(c).name() << endl; // g++打印出"d"
}
```

<a name="Chapter10"></a>
## nullptr
由于0和NULL的二义性，字面常量0既可以是0，也可以是一个无类型指针，因此会导致错误的重载函数被调用等问题。

现在，我们有了nullptr来避免二义性，它是一个所谓“指针空值类型”的常量。并且与以往先定义类型再通过类型声明值的做法完全相反，nullptr的类型nullptr_t是如下定义的：

```cpp
typedef decltype(nullptr) nullptr_t;
```
这种方式非常有趣，充分利用了decltype的功能。

<a name="Chapter11"></a>
## lambda函数
这里我们只介绍一些基本的例子，对于详细的使用规则请参阅官方说明。

```cpp
int main() {
    int girls = 3, boys = 4;
    auto totalChildren = [](int x, int y)->int{ return x + y; };
    return totalChildren(girsl, boys);
}
```

该lambda函数起到了求和的一个效果。

通常情况下，lambda函数的语法定义如下：

```
[capture](parameters) mutable ->return-type{statement}
```

```[capture]```为捕捉列表。[]相当于lambda引出符，编译器根据该引出符判断接下来的代码是否为lambda函数。捕捉列表可捕捉上下文中的变量以供lambda函数使用。

```(parameters)```为参数列表。如果不需要参数，则可以连同括号()一起省略。

```mutable```为mutable修饰符。默认情况下lambda是const函数，mutable可以取消其常量性。在使用该修饰符时，参数列表不可省略（即使参数为空）。

```->return-type```为返回类型，若不需要返回值或者在返回类型明确的情况下，可以省略。

```{statement}```为函数体。它可以使用捕获的变量。

再看一个典型的例子：

```cpp
vector<int> vec = {1, 3, 5, 2, 4};
sort(vec.begin(), vec.end(), [](const int& a, const int& b){ return a > b; });
```

这里，我们调用的sort函数加入了第三个参数comp，也就是用于比较的函数。现在，sort将按递减的顺序排序。另一个使用priority_queue的例子如下所示，注意我们对auto和decltype的利用。

```cpp
int main() {
    auto cmp = [](pair<int, int> &a, pair<int, int> &b) {
    	return a.second > b.second;
    };
    priority_queue<pair<int, int>, vector<pair<int, int>>, decltype(cmp)> que(cmp);
    
    pair<int, int> p1(1, 2);
    pair<int, int> p2(2, 3);
    pair<int, int> p3(4, 1);
    for (auto p : { p1, p2, p3 }) {
    	que.push(p);
    }
    
    while (!que.empty()) {
    	cout << que.top().first << " " << que.top().second << endl;
    	que.pop();
    }
}
```

程序打印结果：

```
4 1
1 2
2 3
```

在C++11之前，我们在使用STL算法时，通常会用到一种函数对象，即仿函数(functor)。例如：

```cpp
class _functor {
public:
    int operator() (int x, int y) { return x + y; }
};

int main() {
    int girls = 3, boys = 4;
    _functor totalChildren;
    return totalChildren(girls, boys);
}
```

这样的仿函数也可以用于比较函数的编写，而使用lambda函数则可以取代这样的仿函数。

此外，通过捕捉列表，我们可以实现类似于“局部函数”的效果。这部分内容请读者自查。
    
<a name="Chapter12"></a>
## 一些有用的特性
这里列举一些有用的特性，详细内容请查阅官方文档。

<a name="Chapter12.1"></a>
### std::array
std::array提供了类似于普通数组的一个封装容器。

<a name="Chapter12.2"></a>
### std::optional
std::optional类型的数据可以表示数据是否存在，这种类型常用于返回值的函数失败的情景。
