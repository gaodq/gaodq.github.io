---
layout: post
title: C++11中的Rvalue references, Move semantics
---

* contents
{:toc}


右值引用、移动语义是C++11中新增的一个非常重要的概念。

### 右值引用（Rvalue references）

#### 左值右值表达式

左值右值是C/C++中表达式的概念，简单来说，可以用两句话来区分左值右值

- 左值是可以放在赋值运算符=左边或者右边的表达式，而右值只能放在右边


- 左值表达式可以用取指操作符&，而右值表达式不能

例如：

```c
int main() {
  int a = 100;
  int *b = &(++a);
  int c = (a + *b);
  c = a++;
  ++a = *b;
  // a++ = 10; // wrong
  // 10 = a; // wrong
  // (a + c) = 10; // wrong
  // int *d = &(a + c); //wrong
  std::string str1 = "hello";
  std::string str2 = std::string("world");
  std::string str3 = str1 + str2;
  printf("b: %d\n", *b);
  printf("a: %d\n", a);
  return 0;
}
```

再举几个左值右值表达式的例子

*lvalue*：`++a` `a`  `*b` `str1[0]`

*rvalue*: `a++` `100` `std::string("world")` `str1 + str2`



#### 右值引用

C++11中引入了新的运算符`Type&&`来表示右值引用，为了区别称`Type&`为左值引用。跟左值引用一样，右值引用不会发生拷贝，并且右值引用等号右边必须是右值，如果是左值则会编译出错，当然这里也可以进行强制转换，这将在后面提到。

作为一个新的类型，右值引用能区分重载函数，`void foo(int&& a)` `void foo(int& a)` 在编译阶段会由编译器自动选择，因此在类的构造函数中如果添加了右值引用版本，就会覆盖其它默认的构造函数，必须显示声明定义。

```c++
class Foo {
  Foo() {}
  Foo(Foo& f) {}
  Foo(Foo&& f) {}
};
```



右值引用是C++11中新增的概念，主要用来解决C++遗留的移动语义问题。

### 移动语义（Move Semantics）

在C++11之前，要将一个对象移动到另一个对象所做的操作是先复制一份到新对象，然后将原对象删除。但是高效的做法就是将原对象的资源直接转移到新对象，但是C++只提供了拷贝构造函数，并不能完成这项任务。C++语言设计者们注意到右值的资源是可以被安全转移的，所以右值引用就作为被用来表达移动语义<sup>[4]</sup>，同时如果确定可以安全转移的左值可以用`std::move()`强制转换为右值引用。`std::move()`是C++11新增的用于将参数强制转换为右值引用，它的实现为：

```c++
template<typename T> 
decltype(auto) move(T&& param) {
  using ReturnType = remove_reference_t<T>&&;
  return static_cast<ReturnType>(param);
}
```



下面介绍几种减少拷贝的优化，前两种是编译器为了减少拷贝所做的优化，最后是C++11新增的移动语义的相关实现。

- 返回值优化（RVO--Return Value Optimization, NRVO--Named Return Value Optimization）

```c++
class Foo {
  Foo() {
    std::cout << "constructor" << std::endl;
  }
  ~Foo() {
    std::cout << "deconstructor" << std::endl;
  }
};

Foo generateFooNRVO() { // NRVO
  Foo f;
  return f;
}

Foo generateFooRVO() { // RVO
  return Foo();
}

int main() {
  Foo mf = generateFooNRVO();
  std::cout << "main function" << std::endl;
  return 0;
}

// output g++ (GCC) 4.7.2 20121015 (Red Hat 4.7.2-5)
// constructor
// main function
// deconstructor
```

现代编译器会对函数返回值进行优化，直接将函数栈内对象的地址赋值，以上例子可以看到`mf`对象的析构延后了。

如果返回结果存在分支<sup>[5]</sup>则编译器无法进行RVO，例如：

```c++
Foo generateFoo(int n) {
  Foo f, b;
  if (n > 1)
    return f;
  else
    return b;
}
```

此时可以用std::move()来强制使用移动语义，当然返回的类型要实现移动构造函数才行。



- std::string的写时拷贝（新STL已废弃）

与Linux的写时拷贝类似，但是多线程环境下会不安全，有时还会引起性能损耗，所以在新编译器版本中已经废弃。<sup>[6]</sup>



- 自定义类移动构造与移动赋值（C++11）

我们简单实现一个MyString类来举例说明移动构造函数的作用

```c++
class MyString {
 public: 
  MyString(const char* str) {
    std::cout << "constructor" << std::endl;
    size_ = strlen(str);
    data_ = new char[size_];
    memcpy(data_, str, size_);
    empty_ = false;
  }
  MyString(const MyString& s) {
    std::cout << "copy constructor" << std::endl;
    size_ = s.size_;
	data_ = new char[size_];
    memcpy(data_, s.data_, s.size_);
    empty_ = false;
  }
  MyString(MyString&& s) {
    std::cout << "move constructor" << std::endl;
    size_ = s.size_;
    data_ = s.data_;
    s.size_ = 0;
    s.data_ = nullptr;
    s.empty_ = true;
  }
  ~MyString() {
    delete data_;
    empty_ = true;
  }
  
  char* data() {
    return data_;
  }
  
  bool empty() {
    return empty_;
  }
 private:
  char* data_;
  int size_;
  bool empty_;
};

int main() {
  MyString str("hello");
  printf("str.data(): %p\n", str.data());
  MyString str1(std::move(str));
  printf("str1.data(): %p\n", str1.data());
  printf("str.empty(): %p\n", str.empty());
  
  return 0;
}

// output:
// constructor
// str.data(): 0xe90010
// move constructor
// str1.data(): 0xe90010
// str.empty(): 1
```

可以看到，通过移动构造函数可以窃取右值的资源地址实现移动资源而非拷贝。



### 常用方法

移动语义在C++11中的应用主要是在STL中，例如`std::vector` `std::map`等容器的减少拷贝，还有`unique_ptr` `function`等不能被拷贝的类。



### 性能测试

简单的小例子展示移动语义带来的性能提升。

```c++
#include <iostream>
#include <chrono>

int main() {
  std::string buf = "dkkbidaksldaj;skjlgitnbkdlaksjdlaf";

  int count = 10000000;
  std::chrono::duration<double, std::milli> diff;
  for (int i = 0; i < count; i++) {
    std::string buf2 = buf;
    auto start = std::chrono::system_clock::now();
    // 1. std::string buf3 = std::move(buf2);
    // 2. std::string buf3 = buf2;
    auto end = std::chrono::system_clock::now();
    diff += end - start;
  }

  std::cout << diff.count() << " ms" << std::endl;

  return 0;
}

// output: Apple LLVM version 7.3.0 (clang-703.0.31) Mac OS X 10.11.1
// 1. 687.988 ms
// 2. 1388.08 ms
```





### 参考资料

1. [http://thbecker.net/articles/rvalue_references/section_01.html](http://thbecker.net/articles/rvalue_references/section_01.html)
2. [http://en.cppreference.com/w/cpp/language/reference](http://en.cppreference.com/w/cpp/language/reference)
3. [http://blog.csdn.net/yapian8/article/details/42341307](http://blog.csdn.net/yapian8/article/details/42341307)
4. [https://www.zhihu.com/question/22111546](https://www.zhihu.com/question/22111546)
5. [https://www.ibm.com/developerworks/community/blogs/5894415f-be62-4bc0-81c5-3956e82276f3/entry/RVO_V_S_std_move?lang=en](https://www.ibm.com/developerworks/community/blogs/5894415f-be62-4bc0-81c5-3956e82276f3/entry/RVO_V_S_std_move?lang=en)
6. [http://www.cnblogs.com/promise6522/archive/2012/03/22/2412686.html](http://www.cnblogs.com/promise6522/archive/2012/03/22/2412686.html)
