+++
title = 'Cpp Virtual'
date = 2024-03-31T18:37:21+08:00
draft = false
+++

[C++中的虚函数、虚继承和内存模型](https://zhuanlan.zhihu.com/p/41309205)


不同继承关系的对象，调用同一函数，产生的行为不同。
C++ 中，多态类型是指声明或者继承了至少一个虚函数的类型。

## 什么是虚函数？

用关键字 `virtual` 声明的类方法。
虚函数是实现多态调用的关键。
运行时，当虚函数被调用时，会匹配对象的动态类型，如果动态类型是子类，且重写了基类虚函数，则将只调用子类的虚函数。否则调用基类的。

- 构造函数不能是虚函数！（因为虚函数表是在构造函数调用前，创建的）
- 析构函数尽可能声明为虚函数
- 友元不能是虚函数，因为友元不是类成员
- 虚函数不能是 static，也不能是内联函数

```cpp
class A
{
public:
	// 非虚函数
    void print(){ printf("A non-virtual\n"); }
    // 虚函数
    virtual void printV(){ printf("A virtual\n"); }
};
 
class B : public A
{
public:
	// 覆盖非虚函数
    void print(){ printf("B non-virtual\n"); }
    // 覆盖虚函数
    // 子类的 virtual 关键字不重要
    virtual void printV(){ printf("B virtual\n"); }
};
 
void main()
{
    B b;
    b.print();
    b.printV();
 
    A*pa = &b;
    pa->print();
    pa->printV();
}

// 输出：
//B non-virtual
//B virtual
//A non-virtual
//B virtual
```

### override

C++ 提供了关键字 `override`，用它标识虚函数后，代表子类希望重写基类虚函数。
因为子类要重写虚函数，必须与基类方法完全匹配，否则编译器会认为子类创建了一个新方法，而不是重写。
主动加上 `override` 关键字，当出现这种不一致的错误时，编译阶段将报错。

```cpp
class Base {
public:
	virtual int foo(int x) { return x; };
};

class Derive : public Base {
	int foo(int x) override {
		return x;
	}
	// 编译时将报错
	int foo(int x) const override {
		return x;
	}
};
```

### final

- 禁止子类重写虚函数
- 禁止再派生子类

不希望子类重写基类的虚方法，可以用 `finnal` 标识符，如果子类重写了，编译阶段就会报错：

```cpp
class Base {
public:
	virtual int foo(int x) { return x; };
};

class DeriveA : public Base {
	// A 子类重写了基类虚方法了，但想就此打住，不希望后面的派生类再重写，可以加 final 关键字
	virtual int foo(int x) override final {
		return x;
	}
};

class DeriveB final : public DeriveA {
	//编译报错，不可以被重写
	virtual int foo(int x) override {
		return x;
	}
}

// 报错，DeriveB 无法再被派生新子类
class DeriveC : public Derive B {

}
```

## 函数的调用捆绑

要想深刻理解虚函数机理，首先要了解函数调用捆绑机制。
捆绑指的是将标识符（如变量名与函数名）转化为地址。
我们知道每个函数在编译的过程中是存在一个唯一的地址的。如果我们在程序段里面直接调用某个函数，那么编译器或者链接器会直接将函数标识符替换为一个机器地址。这种方式是 `早捆绑`，或者叫 `静态捆绑`。因为捆绑是发生在运行之前完成的。

```cpp
int add(int x, int y) {
	return x + y;
}

int subtract(int x, int y) {
	return x - y;
}

int multiply(int x, int y) {
	return x * y;
}

int main() {
	int x;
	cout << "Enter a number";
	cin >> x;

	int y;
	cout << "Enter another number:";
	cin >> y;

	int op;
	cout << "Enter an operation (0=add, 1=subtract, 2=multiply): ";
	cin >> op;

	int result;
	switch (op) {
		// 使用早绑定来直接调用函数
		case 0:
			result = add(x, y);
			break;
		case 1:
			result = subtract(x, y);
			break;
		case 2:
			result = multiply(x, y);
			break;
	}

	cout << "The answer is: " << result <<endl;
	return 0;
}
```

上面三个函数调用都是直接采用函数名，早捆绑的方式。编译器会将每个函数调用替换为一个跳转指令，告诉 CPU 跳转到函数的地址来执行。

但有时候，我们在程序运行前并不知道用哪个函数，因此必须使用 `晚捆绑` 或叫 `动态捆绑`，比如函数指针：

```cpp
// 修改前面的例子

int main() {
	int x;
	cout << "Enter a number";
	cin >> x;

	int y;
	cout << "Enter another number:";
	cin >> y;

	int op;
	cout << "Enter an operation (0=add, 1=subtract, 2=multiply): ";
	cin >> op;

	// 定义一个函数指针
	int (*opFun)(int, int) = nullptr;
	switch (op) {
		case 0:
			opFun = add;
			break;
		case 1:
			opFun = subtract;
			break;
		case 2:
			opFun = multiply;
			break;
	}
	cout << "The answer is: " << opFun(x, y) <<endl;
	return 0;
}
```

## 虚函数表（Vtable）

C++ 采用虚函数表的晚绑定技术，来实现虚函数。
每个具有虚函数的类，和其子类，都有一张虚函数表，该表是在编译阶段建立的静态数组，包含每个虚函数方法的指针，指向其父类（或基类）的虚函数，如果在子类里被重写，则该子类中虚函数指针指向重写后的函数指针。

这样即使子类的对象，赋值给父类的指针，当动态运行时调用到该对象虚函数，可以查询该对象虚函数表，快速跳转到正确的虚函数。

候杰老师关于虚表的模型图，应该是目前为止最直观的：
![](/images/cpp/virtual_table_object_model.png)

- 虚表是属于类的，一个类只需要一张虚表即可
- 虚表指针指向对应类的虚表，占一个指针大小（8 Bytes, x64）
- 没有虚函数的类（包括父类也没有虚函数），不包含虚表
- 对于基类对象，虚表指针指向它的虚表
- 对于派生子类，当基类构造函数调用时，虚表指针先被设置为基类的虚表，在子类构造函数执行前，被更新为子类的虚表
- 虚函数表中，不仅存储了虚函数的地址，还存储了类型 RTTI 的地址、距离真实类型的偏移量等信息

## 纯虚函数与抽象基类

有时候，基类的某个虚方法并不需要实现，希望子类能够重写。这时需要定义纯虚函数。
包含纯虚函数的类，就是抽象基类，不能被直接实例化，否则编译器报错。必须通过实现派生子类，实例化子类使用。因此叫抽象 **基** 类。
纯虚函数的作用是，为派生类提供一个一致性的接口。

定义纯虚函数方法：声明后面紧跟着 `=0`
```cpp
// 因为包含纯虚方法，所以是抽象基类
class Animal {
public:
	// 定义纯虚函数，不能写实现，且在虚函数后面加 = 0
	virtual string speek() const = 0;
};

class Cat : public Animal {
public:
	// 重写了虚函数
	virtual string speek() const {
		return "Meow";
	}
};

class Dog : public Animal {
public:
	// 没有重写基类虚函数
};

int main() {
	// 编译报错，因为抽象基类无法实例化
	Animal ani;

	// 合法
	Cat c;

	// 报错，无法被实例化
	Dog d;

	return 0;
}
```

## 接口类

一个接口类可以看作纯虚方法的集合。（只有定义，没有具体实现）
接口类是一个抽象概念，使用者只关注功能，而不要求了解实现。

## 多重继承的内存模型

假设有 C 同时继承两个独立的基类 A、B
```cpp
struct A
{
    int ax;
    virtual void f0() {}
};

struct B
{
    int bx;
    virtual void f1() {}
};

struct C : public A, public B
{
    int cx;
    void f0() override {}
    void f1() override {}
};
```

和单链继承不同，A、B 完全独立，这使得 A 和 B 在 C 的内存模型里，必须处于两个不相交的区域，同时采用两个虚指针，指向它们：
![](/images/cpp/multi_inherit_mem_model.png)
如图，C 将 A 作为**主基类**，也就是把 C 的虚函数“并入”A 的虚函数表中，并将 A 的虚指针作为 C 的内存起始点。
如果出现下面情况：
```cpp
C* cc = new C;
B* bb = cc;
// 实际调用的是 C 类的 f1()
bb->f1();
```




## 什么是虚基类，虚继承？

### 虚基类

通过关键字 `virtual` 继承的基类。
虚基类主要用来解决菱形继承中的歧义问题。
如下面一节中，A 就是虚基类，B、C 分别虚继承 A。

## 菱形继承问题

为避免重复包含 A 中的成员，造成调用基类 A 成员时出现二义性（D到底是从 B->A，还是 C->）
解决方案：B 和 C 分别虚继承 A。

可以简化的理解为，通过虚继承，D 中只保留了一份虚基类 A 的实例（同名的虚基类只产生一个虚基类子对象），运行时通过虚表的表 VTT，查询 A 中成员偏移量，然后跳转直行。

```cpp
// 菱形继承
// 运行：g++ -std=c++11 main.cpp -o demo && ./demo

#include <iostream>

using namespace std;

class A {
  public:
    int iValue = 1;
};

class B : public A {
  public:
    void bPrintf() { cout << "This is class B" << endl; };
};

class C : public A {
  public:
    void cPrintf() { cout << "This is class C" << endl; };
};

class D : public B, public C {
  public:
    void dPrintf() { cout << "This is class D" << endl; };
};

int main() {
    D d;
    // cout << d.iValue << endl;    // 错误，不明确的访问
    // cout << d.A::iValue << endl; // 错误，基类A不明确
    cout << d.B::iValue << endl; // 正确
    cout << d.C::iValue << endl; // 正确
    return 0;
}
```


D 中 B、C 的偏移量在编译期可以确定，而 A 的偏移量在运行时确定
![](/images/cpp/cpp_rhombic_inherit.png)

A 的内存模型就是普通的类和虚表模型：
![](/images/cpp/cpp_rhombic_inherit_A.png)

B 的内存模型和 C 的一样，这里只看 B 的：
![](/images/cpp/cpp_rhombic_inherit_B.png)

D 的内存模型更复杂：
![](/images/cpp/cpp_rhombic_inherit_C.png)
