+++
title = 'C++智能指针'
date = 2024-03-29T23:24:48+08:00
draft = false
+++

## 简介

C++ 程序设计中，使用堆内存时非常频繁的操作，堆内存的申请和释放都由程序员自己管理。C++ 引入了智能指针帮助完成这个过程。使用普通指针，容易造成堆内存泄露（忘记 `delete`）或程序异常崩溃（多次 `delete`）。

C++11 摒弃了 `auto_ptr`（如果编译器较老，不支持 C++11，则只有 auto_ptr 可用），并新增了三种智能指针：`unique_ptr`、`shared_ptr`、`weak_ptr`。

## 用法

### 简单用法

- 将 `new` 获得的指针，赋给智能指针对象，创建智能指针，一般直接在栈上创建智能指针对象。
- 智能指针对象很多方面都类似常规指针，可以直接执行解引用操作(\*ps) 、箭头调用（ps->comment())、将智能指针赋值给相同类型的常规指针。
- 智能指针过期后，将在析构函数中使用 `delete` 来释放相应内存。

smrtptrs.cpp:
```cpp
#include <iostream>
#include <string>
// 在标准库 memory 中
#include <memory>

class Report {
private:
    std::string str;
public:
    Report(const std::string s) : str(s) {
        std::cout << "Object created!\n";
    }
    ~Report() {
        std::cout << "Object deleted!\n";
    }
    void comment() const {
        std::cout << str << "\n";
    }
};

int main() {
	// 代码块结束时，智能指针 lifecycle 结束，都自动调用对象析构函数
	{
		std::auto_ptr<Report> ps(new Report("using auto_ptr"));
		ps->comment();
	}

	{
		std::shared_ptr<Report> ps(new Report("using shared_ptr"));
		ps->comment();
	}

	{
		std::unique_ptr<Report> ps(new Report("using unique_ptr"));
		ps->comment();
	}
	return 0;
}

//output:
//Object created!
//using auto_ptr
//Object deleted!
//Object created!
//using shared_ptr
//Object deleted!
//Object created!
//using unique_ptr
//Object deleted!
```


### `auto_ptr`

C++98 提供，C++11 已经 [摒弃](智能指针.md#为何摒弃%20`auto_ptr`%20？) ，`unique_ptr` 用法与之一样。

### `unique_ptr`

`new` 分配的对象，由 `unique_ptr` 维护内存，智能指针过期时负责 `delete`。
一个对象同一时间只能被一个 `unique_ptr` 所有，否则编译时会报错，可以通过 `std::move()` 转移所有权。
```cpp
unique_ptr<string> p3(new string("test"));
unique_ptr<string> p4;
//p4 = p3;  // 编译报错！因为可能造成 p3、p4 分别 delete 同一对象
```

### `shared_ptr`

`new` 分配的对象，由 `shared_ptr` 维护内存，智能指针过期时负责 `delete`。
一个对象同一时间可以被多个 `shared_ptr` 所有，通过 **引用计数** 跟踪该对象，当引用计数降到 0 时，释放该对象。
```cpp
shared_ptr<string> p3(new string("test"));
shared_ptr<string> p4;
p4 = p3;  // p3、p4指向同一对象，引用计数加1，当智能指针陆续生命周期结束，引用计数降到0，释放指向对象的内存
cout<<p3.use_count()<<endl;  // 查看引用计数
cout<<p3.get(); // 获取原始对象指针
```

### `weak_ptr`

弱引用，协助 `shared_ptr` 完成工作，使用前要判断是否为空。
引用计数有个问题，就是可能两个或多个shared_ptr互相引用，导致shared_ptr的引用计数无法到0，而出现内存泄露的情况。
weak_ptr 是一个弱引用，只引用不计数。
如果一块内存被shared_ptr和weak_ptr同时引用，当所有shared_ptr析构了之后，不管还有没有weak_ptr引用该内存，内存也会被释放。所以weak_ptr不保证它指向的内存一定是有效的，**在使用之前需要检查weak_ptr是否为空指针**。

```cpp
#include <iostream>
#include <memory>
 
int main() {
    {
        std::shared_ptr<int> sh_ptr = std::make_shared<int>(10);
        std::cout << sh_ptr.use_count() << std::endl;
 
        std::weak_ptr<int> wp(sh_ptr);
        std::cout << wp.use_count() << std::endl;
 
        if(!wp.expired()){
            std::shared_ptr<int> sh_ptr2 = wp.lock(); //get another shared_ptr
            *sh_ptr = 100;
            std::cout << wp.use_count() << std::endl;
        }
    }
    //delete memory
    return 0;
}
```

### 错误姿势

#### 1. 隐式转换

```cpp
shared_ptr<double> pd;
double *p_reg = new double;
//pd = p_reg; // 不接受隐式类型转换！
pd = shared_ptr<double>(p_reg) // ok
```

#### 2. 不用 `new` 出来的指针创建

```cpp
string vacation("I wandered lonely as a cloud.");
shared_ptr<string> pvac(&vacation); // NO!
```

#### 3. `shared_ptr` 循环引用，造成内存泄露

```cpp
// 声明
class Son;
class Father {
public:
	Father() {}
	~Father() {}
	shared_ptr<Son> son;
}

class Son {
public:
	Son() {}
	~Son() {}
	shared_ptr<Father> fa;
}

int main() {
	Father* fa = new Father();
	Son* so = new Son();

// 循环引用，造成内存泄露
	shared_ptr<Father> fa2(fa);
	shared_ptr<Son> so2(so);
	fa2->son = son2;
	so2->fa = fa2;

	return 0;
}
```

## 为何摒弃 `auto_ptr` ？

```cpp
auto_ptr<string> ps(new string("test"));
auto_ptr<string> vocation;
vocation = ps; // 将可能在运行期崩溃（随系统而异）
...
```
ps、vocation 过期时，将分别试图删除 string 对象，造成同一个对象两次被 delete，而该写法在编译阶段不报错。
这种写法，可能导致不确定的行为或程序神秘崩溃。

`unique_ptr` 编译阶段就会提示报错，相对安全
```cpp
unique_ptr<string> p3(new string("test"));
unique_ptr<string> p4;
//p4 = p3;  // 编译报错！
p4 = move(p3); // 标准库也提供方法 std::move()，让所有权转移，转移后，p3 成为无效指针

unique_ptr<string> p5;
p5 = unique_ptr<string>(new string("test")); // 临时右值，编译器允许赋值
```

## 智能指针的实现

智能指针本质上是一个模版类，构造函数中传入一个普通指针，析构函数中释放该指针。
