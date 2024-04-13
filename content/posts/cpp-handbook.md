+++
title = 'C++ Handbook'
date = 2024-03-31T22:31:19+08:00
draft = false
+++

## 构造

### 1. 默认构造函数

如果没有提供任何构造函数，C++ 将自动提供默认构造函数。
如果显式定义了带参数的构造函数，编译器将不再自动生成默认构造函数，而如果此类有派生子类，派生类构造函数被调用时，会自动调用基类默认构造函数，没有默认构造函数，编译器会报错，解决方案：
- 显式定义默认构造函数
- 在派生类的每个构造函数中，都显示调用带参数版本的基类构造函数。

```cpp
class Stock {
	// 默认构造不做任何工作
	Stock() {}
	// 同样，如果没有提供西沟函数，也会自动提供默认析构函数
	~Stock() {}
};
```

### 2. 复制构造函数

用于将一个对象复制到新创建的对象中。
```cpp
ClassName(const ClassName& c);
```
下面几种情况都会调复制构造函数：
```cpp
Time t2(t1);
Time t2 = t1;
Time t2 = Time(t1);
Time* t2 = new Time(t1);
```

> 默认的复制构造函数（隐式复制构造函数），是逐个复制成员变量（非 static 成员）的值，是浅拷贝。这会造成很多问题，因此大多数时候都要显式定义复制构造函数。

```cpp
class Time {
	explicit Time(int t);
	// 在显式复制构造函数里，进行深拷贝
	Time(const Time& t1);
}
```

### 3. 赋值运算符

将一个对象赋值给另一个对象，下面几种情况会调用赋值运算符：
```cpp
Time t1(123);
Time t2(456);
t1 = t2;
Time t3 = t1;  //这种情况复制构造函数+赋值运算符函数都用了
```
和复制构造函数类似，也显示的创建赋值运算符函数，解决浅拷贝的问题：
```cpp
class Time {
public:
	explicit Time(int t);
	Time& operator=(const Time& t1);
}
```

### 4. 转换函数

只有接受一个参数的构造函数，才能作为转换函数。

#### 隐式类型转换（implicit conversion）

```cpp
class Time {
public:
	Time(int t);
	// 不能作为转换函数
	Time(int year, int month);
};

Time now;
now = 1234;
```
隐式类型转换，自动调用了 Time(int t) 构造函数，创建了 Timer 对象并赋值给 now。
可能发生在下面场景：
- int 值赋值给 Time 对象
- int 值传递给 Time 函数参数时
- 函数声明返回参数式 Time，而实际返回 int 值时

#### 显式类型转换（explicit conversion）

显式声明 `explicit` 后，禁用隐式转换
```cpp
class Time {
public:
	explicit Time(int t);
	// 不能作为转换函数
	Time(int year, int month);
};

Time now;
now = 1234;  // 不允许！
now = Time(1234);  // OK
```

#### 类型转换函数

想实现下面的操作，要用到类型转换函数：
```cpp
Time now(123);
int a = now;
int b = int(now);
```

```cpp
class Time {
	explicit Time(int t);
	// 类型转换函数
	operator int() const;
};

Time::operator int() const {
	return 123;
}
```

### 5. 派生子类会调基类构造函数吗？

会
派生类构造函数，会自动调用基类默认构造函数。

### 6. C++中构造函数可以是私有的吗？

可以！
典型的单例模式就是要把构造函数设置为私有：

```cpp
class Singleton {
public:
	static Singleton& getInstance() {
		static Singleotn s_singleton;
		return s_singleton;
	}

private:
	Singleton() {};
	Singleton(const Singleton&) = delete;
	Singleton& operator=(const Singleton&) = delete;
}
```

## 析构

不显示声明析构函数的，编译器会提供一个默认的析构函数，不执行任何操作。

析构函数可以被主动调用，但没必要
析构函数可以声明在 private 域中，但多此一举，如果这样做，在类外面是无法调用到析构函数的，为内存释放主动制造困难了属于是。

### 为什么析构函数最好设置为虚函数？

否则销毁指向子类的基类指针时，就只会调用基类的析构函数，而子类的析构、子类成员对象的析构，就永远不会被调用到，造成内存泄露。

```cpp
class Resource {
  public:
    Resource() { cout << "Resource created!" << endl; }
    ~Resource() { cout << "Resource destoryed!" << endl; }
};

class Super {
  public:
    Super() { cout << "Super constructor called!" << endl; }
    ~Super() { cout << "Super destructor called!" << endl; }
};

class Sub : public Super {
  public:
    Sub() { cout << "Sub constructor called!" << endl; }

    ~Sub() { cout << "Sub destructor called!" << endl; }

  private:
    Resource res;
};

int main() {
    Sub *sub = new Sub;
    Super *super = sub;
    delete super;

    cin.get();
    return 0;
}

//输出：
//Super constructor called!
//Resource created!
//Sub constructor called!
//Super destructor called!
```

### 子类析构函数被调用时，基类析构函数会被调用吗？

会！

## 运算符重载

示例：
```cpp
// mytime.h
class Time {
public:
	Time& operator+(const Time& t) const;
};
```

限制：
1. 运算符两边至少有一个，是用户定义的类型，防止用户为标准类型重载运算符
2. 重载后的运算符，与原来有相同的优先级
3. 重载后的运算符，与原来有相同的句法规则（如：a++，重载后还是一个操作数++，a+b，重载后还是两个操作数）
4. 可重载的运算符：
![](/images/cpp/overload-operator-1.png)
![](/images/cpp/overload-operator-2.png)
5. 禁止重载的运算符：
	- `sizeof`:sizeof 运算符
	- `.`: 成员运算符
	- `.*`: 成员指针运算符
	- `::`: 作用域解析运算符
	- `?:`: 条件运算符
	- `typeid`: 一个 RTTI 运算符
	- `const_cast`: 强制类型转换运算符
	- `dynamic_cast`: 强制类型转换运算符
	- `reinterpret_cast`: 强制类型转换运算符
	- `static_cast`: 强制类型转换运算符

## const

const 原意为声明常量，不可修改。
必须初始化。

### 总结几个常见的 const 用法：

```cpp
const char* text = "hello";

void func(const Time& t);
Time& method(const Time& t) const;

int a = 1;
// 通过 b 指针无法修改 a 值
const int* b = &a;
// 但 b 还能被赋值为其它指针
b = &c;

// b 指针被锁定为不可修改
int* const b = &a;
// b = &c; // NO
```

### 引用

```cpp
int a = 1;
int& b = a;
b = 2;  // a == 2，a和b的值和地址都相同

// 因此引用必须在声明时初始化，下面的做法不允许
int a = 1;
int& b; // 报错，引用变量必须在声明时初始化
//b=a;  // NO
```

### 在声明引用参数时，尽可能声明为 const 的原因：

```cpp
void display(const Time& t);
```

- 避免无意中修改被引用数据
- 兼容带 const 和不带 const
- 函数可以正确生成和使用临时变量（赋值、传参时，如果两边类型不一致，可能会生成临时变量）

## static

C/C++ 内存分布：
![](/images/cpp/cpp_mem_model.png)
可以看到，单独有一块静态内存区，既不在栈上，也不在堆上。

### 静态局部变量

作用域：函数内部
生存期：程序运行期间
特点：
1. 分配在全局数据区（普通局部变量在栈上分配）；
2. 函数首次执行到声明处，静态局部变量首次被初始化，以后再次调用函数，也不会再被初始化；
3. 一般在声明时初始化，否则被自动初始化为0
4. 作用域只在局部

### 静态全局变量（函数）

作用域：文件内部
生存期：程序运行期间
特点：
被限定在该文件内部，其它文件无法使用。
（普通全局变量，其它文件可以通过 `extern int abc;` 的方式，调用另一个文件的全局变量）
静态函数和全局变量表现类似。

### 静态成员变量

```cpp
class Time {
public:
	//静态成员函数
	static void show();
	// 静态成员变量
	static int aa;

	// 普通成员变量
	int bb;
};

// 初始化静态成员变量
int Time::aa = 1;
```
静态成员也存储在全局数据区，也遵循 public, private, protected 访问原则
不能在类中初始化，一般系统默认初始化为0
属于类，不属于某个特定 instance

### 静态成员函数

调用：
```cpp
Time::show();
```
- 没有类型实例，也可以直接调用静态成员函数
- 属于类，不属于某个特定 instance，不能访问 `this` 指针
- 只能访问静态成员变量，静态成员函数，不能访问普通变量
- 普通类方法，可以访问静态成员函数，静态函数不能访问普通类方法

## 类型转换



### static_cast

`static_cast<>` 可以实现基本的类型转换：
```cpp
int a=1;
float b=3.14;
a=static_cast<int>(b);    //将float类型转换为int类型
b=static_cast<float>(a);  //将int类型转换为float类型
int *q=NULL;
void* p = NULL;
q=p;      //将空指针转换为int类型，C语言允许，C++不允许
p=q;
q=static_cast<int*>(p);   //将空指针转换为int类型指针
```
不要用它做类对象的类型转换

### const_cast

`const_cast<>` 用于：
- 移除 const 对象常量性质
- 只支持指针、引用

```cpp
int num = 100;
const int* p1 = &num;
//将常量指针转换为普通类型指针,去除const属性
int* p2 = const_cast<int*>(p1);
*p2 = 200;
int a=100;
const int & ra=a;
//将常量引用转换为普通类型引用，去除const属性
const_cast<int&>(ra)=200;
```

### RTTI：dynamic_cast

RTTI（Runtime Type Identification），运行阶段类型识别。

`dynamic_cast<>` 在运行时做安全的类型转换，主要用于类层次的向上、向下转换，与 `static_cast` 相比，如果转换是不安全的，转换失败会返回 `nullptr`，因此在 `dynamic_cast` 转换后，要判断空指针。

- 只支持多态类的转换（包含虚函数），普通类型转换编译期报错
- 转换失败返回 nullptr

```cpp
Base* p1 = new Base;
Base* p2 = new Derive;

// 不报错，转换完后 p3 也不为空，但不安全，因为子类中有的成员，p1 没有，p3 访问那些成员是不合法的
Derive* p3 = (Derive*)p1;

// 正确的做法是用 dynamic_cast<>
// p4 == nullptr
Derive *p4 = dynamic_cast<Derive *>(p1);
// p5 ok
Derive *p5 = dynamic_cast<Derive *>(p2);
// 不相关的类，也可以合法转换，返回空
// p6 == nullptr
Other *p6 = dynamic_cast<Other *>(p1);
```
### reinterpret_cast

- 属于比较底层的强制转换
- 没任何类型检查和格式转换
- 仅仅是简单的二进制数据拷贝覆盖
- 支持指针、引用或算数类型

```cpp
int *p=NULL; 
float *q=NULL;
p = q;                                //C 语言允许，C++语言不允许
q = p;                                //C 语言允许，C++语言不允许
p = static_cast<int*>(q);         //static_cast无法转换
q = static_cast<int*>(p);         //static_cast无法转换
p = reinterpret_cast<int*>(q);
q = reinterpret_cast<float*>(p)
```

## 友元

友元有三种：
- 友元函数
- 友元类
- 友元成员函数

### 创建友元函数

- 友元函数，声明在类中
- 不是类方法，但可以与类方法一样的访问类权限
- 定义时，不能用类限定符 Time::

```cpp
// 声明在类中
class Time {
public:
	friend Time operator*(double m, const Time& t);
};

// 友元函数实现，不能带类限定符，Time::
Time operator*(double m, const Time& t) {}

// 可以实现
Time A;
Time B = 2.75 * A;
```

常见友元：
```cpp
// 重载 <<
class Time {
public:
	friend ostream& operator<<(ostream& os, const Time& t);
};

// 友元函数实现
ostream& operator<<(ostream& os, const Time& t) {
	os << t.hours << ":" << t.minutes << endl;
	return os;
}
```

### 友元类

- 友元类的所有方法，都可以与原始类方法一样的访问类权限
- 只能由原始类定义，不能在外部强加友情

```cpp
class Tv {
public:
	// Remote 成为 Tv 类的友元类
	friend class Remote;
	int volume;
};

class Remote {
public:
	void setVolum(Tv& t, int x) {t.volume = x;}
}
```

### 友元成员函数

- 有时候不需要整个类都成为友元类，而只希望开放某些方法成为友元成员函数

```cpp
class Remote {
public:
	void setVolum(Tv& t, int x);
}

class Tv {
public:
	// 则只开放了这一个成员函数的 Tv 类访问权限
	friend void Remote::setVolum(Tv& t, int x);
}
```

## 智能指针

[C++智能指针](https://liujunguang.github.io/posts/cpp-smart-ptr/)

## 访问权限: public/protected/private

### 类访问权限

- 默认都是 private
- 顺序任意，次数任意，直到出现另一个限定符号
- public: 可以被该类成员、子类成员、友元成员访问，也可以在通过该类实例对象访问
- protected: 可以被该类成员、子类成员、友元成员访问，但不可以由该类对象访问
- private: 可以被该类成员、友元成员访问，但不可以被子类成员访问，也不能在外部通过该类实例对象访问

### 继承访问权限

![](/images/cpp/inherit_accession.png)

另外，注意：private 继承，**不能**隐式向上转换（implicit upcasting）——不需要显式类型转换，就可以让基类指针或引用，指向派生子类对象。

补充：
- struct 默认的成员访问权限是 public
- struct 默认的继承访问权限是 public

## 模版

### 函数模版

### 类模版

## 右值

- 左值既能出现在 `=` 左边，也能出现在 `=` 右边
- 右值只能出现在 `=` 右边
- 左值可以寻址，一般有持久性
- 右值一般不可寻址，或在表达式运算过程中的临时对象

```cpp
int a; // a 为左值
a = 3; // 3 为右值

int x = 6; // x是左值，6是右值
int &y = x; // 左值引用，y引用x

int &z1 = x * 6; // 错误，x*6是一个右值
const int &z2 =  x * 6; // 正确，可以将一个const引用绑定到一个右值

int &&z3 = x * 6; // 正确，右值引用
int &&z4 = x; // 错误，x是一个左值
```
