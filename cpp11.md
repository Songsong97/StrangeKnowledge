# cpp11
奇怪的知识增加了！！！
## Table of contents
1. [设计目标](#Chapter1)
2. [long long整型](#Chapter2)
3. [快速初始化成员变量](#Chapter3)
4. [std::move与smart pointers](#Chapter4)
5. [右尖括号>的改进](#Chapter5)
6. [auto类型推导](#Chapter6)
7. [decltype与typeid](#Chapter7)
8. [基于范围的for循环](#Chapter8)
9. [nullptr](#Chapter9)
10. [lambda函数](#Chapter10)
11. [std::array]
12. [std::optional]

<a name="Chapter1"></a>
## 设计目标
C++11的整体设计目标如下：
1. 使得C++成为更好的适用于系统开发及库开发的语言。
2. 使得C++成为更易于教学的语言（语法更加一致和简单）。
3. 保证语言的稳定性，以及和C++03及C语言的兼容性。

尽管C++11的一个设计目标是倾向于使用库而不是扩展语言来实现特性，但标准仍然增加了一些关键词或者扩充了一些核心语法特性。

<a name="Chapter2"></a>
### long long整型
标准要求long long整型可以在不同平台上有不同的长度，但至少有64位。我们在写常数字面量时，可以用LL或者ll标识一个long long类型的字面量。比如：

```cpp
long long a = -90000000000000000LL;
```

同其他的整型一样，可以用<climits>中定义的LLONG_MIN，LLONG_MAX和ULLONG_MIN查看平台上long long的表示范围。

<a name="Chapter3"></a>
### 快速初始化成员变量
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

<a name="Chapter4"></a>
### std::move与smart pointers

