---
title: C++ 对象内存布局
date: 2022-05-28 11:05:49
categories:
- C/C++开发
tags:
- C/C++开发
---

可能会对 C++ 对象的内存布局产生影响的因素：
<!--more-->
1. 对象的数据成员变量
2. 对象的一般成员函数
3. 对象的虚成员函数
4. 对象继承 - 父类不包含虚函数
5. 对象继承 - 父类包含虚函数
6. 对象多继承 - 父类不包含虚函数
7. 对象多继承 - 父类包含虚函数
8. 对象虚继承
9. 动态类型信息（这里不讨论）
10. 通过汇编代码看虚函数调用过程

这些特性的实现常常因操作系统平台和编译器/ABI 而异，这里的分析主要基于 linux x86_64 平台上的  GCC 11.2.0/G++ 11.2.0 的行为进行。

## 一般 C++ 对象的内存布局

一般 C++ 对象指只有若干数据成员变量的 C++ 对象，它们是 C++ 中最简单的类对象。用 `class` 关键字定义的这种类，除了成员变量的默认可见性之外，与 C 语言中的结构体完全一样。

这里定义一个一般 C++ 类，实例化一个对象，并查看对象的地址和各个数据成员变量的地址：
```
class Object;
void print_field_address(Object *obj);

class Object {
public:
  int32_t data_1 = 0;
  int16_t data_2 = 0;
  int64_t data_3 = 0;
  int32_t data_4 = 0;
private:
  int32_t data_5 = 0;
  int32_t data_6 = 0;

  friend void print_field_address(Object *obj);
};

void print_field_address(Object *obj) {
  printf("Address of obj %p, object size %zu bytes\n", obj, sizeof(*obj));
  printf("Address of Field data_1 %p\n", &obj->data_1);
  printf("Address of Field data_2 %p\n", &obj->data_2);
  printf("Address of Field data_3 %p\n", &obj->data_3);
  printf("Address of Field data_4 %p\n", &obj->data_4);
  printf("Address of Field data_5 %p\n", &obj->data_5);
  printf("Address of Field data_6 %p\n", &obj->data_6);
}

int main(int argc, char *argv[]) {
  Object obj;
  print_field_address(&obj);

  return 0;
}
```

上面这段代码的输出可能像下面这样：
```
Address of obj 0x7fff3c605fa0, object size 32 bytes
Address of Field data_1 0x7fff3c605fa0
Address of Field data_2 0x7fff3c605fa4
Address of Field data_3 0x7fff3c605fa8
Address of Field data_4 0x7fff3c605fb0
Address of Field data_5 0x7fff3c605fb4
Address of Field data_6 0x7fff3c605fb8
```

在不同的机器上执行，或者在同一台机器上多次执行，每次执行的输出都可能不同，但对象和各个成员变量的具体内存地址的差异不影响分析结论。

通过上面输出的地址可以看到：

 * 一般 C++ 对象的地址和它的第一个成员变量的地址相同；
 * 一般 C++ 对象的各个数据成员在内存中按它们声明的顺序逐个排列；
 * 各个数据成员之间可能由于成员变量的数据类型而存在填充。
 * 对象的大小由于填充，比实际各个成员大小之和要大。在例子的对象中，填充发生在两个地方。

## 具有成员函数的一般 C++ 对象的内存布局

这里我们给上面的一般 C++ 类添加几个成员函数，再来看它的类对象内存布局。示例代码如下：
```
class Object;
void print_field_address(Object *obj);

class Object {
public:
  Object();
  ~Object();
  void funcA();
  void funcB();
  int32_t data_1 = 0;
  int16_t data_2 = 0;
  int64_t data_3 = 0;
  void funcC();
  int32_t data_4 = 0;
private:
  void funcD();
  int32_t data_5 = 0;
  void funcE();
  int32_t data_6 = 0;

  friend void print_field_address(Object *obj);
};

Object::Object() {}
Object::~Object() {}
void Object::funcA() {}
void Object::funcB() {}
void Object::funcC() {}
void Object::funcD() {}
void Object::funcE() {}

void print_field_address(Object *obj) {
  printf("Address of obj %p, object size %zu bytes\n", obj, sizeof(*obj));
  printf("Address of Field data_1 %p\n", &obj->data_1);
  printf("Address of Field data_2 %p\n", &obj->data_2);
  printf("Address of Field data_3 %p\n", &obj->data_3);
  printf("Address of Field data_4 %p\n", &obj->data_4);
  printf("Address of Field data_5 %p\n", &obj->data_5);
  printf("Address of Field data_6 %p\n", &obj->data_6);
}

int main(int argc, char *argv[]) {
  Object obj;
  print_field_address(&obj);

  return 0;
}
```

上面这段代码的输出可能像下面这样：
```
Address of obj 0x7ffe6d94ba00, object size 32 bytes
Address of Field data_1 0x7ffe6d94ba00
Address of Field data_2 0x7ffe6d94ba04
Address of Field data_3 0x7ffe6d94ba08
Address of Field data_4 0x7ffe6d94ba10
Address of Field data_5 0x7ffe6d94ba14
Address of Field data_6 0x7ffe6d94ba18
```

由于类成员函数属于类，而不属于类对象，因而添加成员函数对类对象的内存布局没有任何影响。从打印的类对象地址和类对象各个成员变量的地址可以看出来。

## 具有虚函数的 C++ 类对象的内存布局

这里我们将上面具有成员函数的类的一部分成员函数改为虚函数，再来看它的类对象内存布局。具有虚函数的类，其对象中会隐式包含一个指针，指向虚函数表。为了确定虚函数表指针的位置，这里添加一个类，其中包含两个数据成员变量，并将包含虚函数的类的对象嵌在这两个成员变量中间。示例代码如下：
```
class Object;
void print_field_address(Object *obj);

class Object {
public:
  Object();
  virtual ~Object();
  void funcA();
  virtual void funcB();
  int32_t data_1 = 0;
  int16_t data_2 = 0;
  int64_t data_3 = 0;
  void funcC();
  int32_t data_4 = 0;
private:
  virtual void funcD();
  int32_t data_5 = 0;
  void funcE();
  int64_t data_6 = 0;

  friend void print_field_address(Object *obj);
};

Object::Object() {}
Object::~Object() {}
void Object::funcA() {}
void Object::funcB() {}
void Object::funcC() {}
void Object::funcD() {}
void Object::funcE() {}

void print_field_address(Object *obj) {
  printf("Address of obj %p, object size %zu bytes\n", obj, sizeof(*obj));
  printf("Address of Field data_1 %p\n", &obj->data_1);
  printf("Address of Field data_2 %p\n", &obj->data_2);
  printf("Address of Field data_3 %p\n", &obj->data_3);
  printf("Address of Field data_4 %p\n", &obj->data_4);
  printf("Address of Field data_5 %p\n", &obj->data_5);
  printf("Address of Field data_6 %p\n", &obj->data_6);
}

class Object1 {
public:
  int64_t data_10 = 0;
  int64_t data_11 = 0;
  Object obj;
  int32_t data_12 = 0;
};

int main(int argc, char *argv[]) {
  Object1 obj;
  printf("Pointer size %zu bytes\n", sizeof(void *));
  printf("Address of Field data_10 %p\n", &obj.data_10);
  printf("Address of Field data_11 %p\n", &obj.data_11);
  print_field_address(&obj.obj);
  printf("Address of Field data_12 %p\n", &obj.data_12);


  return 0;
}
```

上面这段代码输出如下：
```
Pointer size 8 bytes
Address of Field data_10 0x7ffca60e3cf0
Address of Field data_11 0x7ffca60e3cf8
Address of obj 0x7ffca60e3d00, object size 40 bytes
Address of Field data_1 0x7ffca60e3d08
Address of Field data_2 0x7ffca60e3d0c
Address of Field data_3 0x7ffca60e3d10
Address of Field data_4 0x7ffca60e3d18
Address of Field data_5 0x7ffca60e3d1c
Address of Field data_6 0x7ffca60e3d20
Address of Field data_12 0x7ffca60e3d28
```

在笔者这台 64 位的机器上，指针变量的长度为 8 个字节。具有虚函数的类的对象，其对象地址与第一个数据成员变量的地址之差为 8，即一个指针变量的长度，对象的起始地址可以与它前面的数据之间没有任何空隙，对象的最后一个数据成员也可以与它后面的数据之间没有任何空隙。

对于具有虚函数的 C++ 类对象，其内存布局为：

 * 虚函数表指针位于类对象的开头位置，虚函数表指针的地址与类对象的地址相同；
 * 虚函数的个数，虚函数声明的位置等不影响类对象的内存布局；
 * 虚函数表指针后面是其它数据成员变量，它们的排列就像一般 C++ 类对象那样排列。

## 单继承中子类对象的内存布局

这里我们设计一个类，让它继承一个具有成员函数和数据成员变量的一般 C++ 类，并查看其类对象的内存布局。示例代码如下：
```
class Object;
void print_field_address(Object *obj);

class Object {
public:
  Object();
  ~Object();
  void funcA();
  void funcB();
  int32_t data_1 = 0;
  int16_t data_2 = 0;
  int64_t data_3 = 0;
  void funcC();
  int32_t data_4 = 0;
private:
  void funcD();
  int32_t data_5 = 0;
  void funcE();
  int64_t data_6 = 0;

  friend void print_field_address(Object *obj);
};

Object::Object() {}
Object::~Object() {}
void Object::funcA() {}
void Object::funcB() {}
void Object::funcC() {}
void Object::funcD() {}
void Object::funcE() {}

void print_field_address(Object *obj) {
  printf("Address of obj %p, object size %zu bytes\n", obj, sizeof(*obj));
  printf("Address of Field data_1 %p\n", &obj->data_1);
  printf("Address of Field data_2 %p\n", &obj->data_2);
  printf("Address of Field data_3 %p\n", &obj->data_3);
  printf("Address of Field data_4 %p\n", &obj->data_4);
  printf("Address of Field data_5 %p\n", &obj->data_5);
  printf("Address of Field data_6 %p\n", &obj->data_6);
}

class Object1 : public Object {
public:
  int64_t data_10 = 0;
  int64_t data_11 = 0;
  int32_t data_12 = 0;
};

int main(int argc, char *argv[]) {
  Object1 obj;
  printf("Pointer size %zu bytes, obj1 address %p, obj1 size %zu bytes\n",
      sizeof(void *), &obj, sizeof(obj));
  print_field_address(&obj);
  printf("Address of Field data_10 %p\n", &obj.data_10);
  printf("Address of Field data_11 %p\n", &obj.data_11);
  printf("Address of Field data_12 %p\n", &obj.data_12);

  return 0;
}
```

上面这段代码的输出如下：
```
Pointer size 8 bytes, obj1 address 0x7ffff0033c90, obj1 size 56 bytes
Address of obj 0x7ffff0033c90, object size 32 bytes
Address of Field data_1 0x7ffff0033c90
Address of Field data_2 0x7ffff0033c94
Address of Field data_3 0x7ffff0033c98
Address of Field data_4 0x7ffff0033ca0
Address of Field data_5 0x7ffff0033ca4
Address of Field data_6 0x7ffff0033ca8
Address of Field data_10 0x7ffff0033cb0
Address of Field data_11 0x7ffff0033cb8
Address of Field data_12 0x7ffff0033cc0
```

在单继承中，子类对象的内存布局如下：

 * 子类对象的开始位置保存父类对象的内容；
 * 子类自己的数据成员排在父类的所有内容之后，两者之间可能有填充；
 * 子类的数据成员在内存中按声明的顺序排列。

当父类具有虚函数时，单继承的基本逻辑是一样的，即子类对象的开始位置保存父类对象的内容，只是这时父类对象的内容会包含父类的虚函数表指针。示例代码如下：
```
class Object;
void print_field_address(Object *obj);

class Object {
public:
  Object();
  virtual ~Object();
  virtual void funcA();
  void funcB();
  int32_t data_1 = 0;
  int16_t data_2 = 0;
  int64_t data_3 = 0;
  virtual void funcC();
  int32_t data_4 = 0;
private:
  void funcD();
  int32_t data_5 = 0;
  void funcE();
  int64_t data_6 = 0;

  friend void print_field_address(Object *obj);
};

Object::Object() {}
Object::~Object() {}
void Object::funcA() {}
void Object::funcB() {}
void Object::funcC() {}
void Object::funcD() {}
void Object::funcE() {}

void print_field_address(Object *obj) {
  printf("Address of obj %p, object size %zu bytes\n", obj, sizeof(*obj));
  printf("Address of Field data_1 %p\n", &obj->data_1);
  printf("Address of Field data_2 %p\n", &obj->data_2);
  printf("Address of Field data_3 %p\n", &obj->data_3);
  printf("Address of Field data_4 %p\n", &obj->data_4);
  printf("Address of Field data_5 %p\n", &obj->data_5);
  printf("Address of Field data_6 %p\n", &obj->data_6);
}

class Object1 : public Object {
public:
  void funcA() override;
  void funcC() override;
  int64_t data_10 = 0;
  int64_t data_11 = 0;
  int32_t data_12 = 0;
};

void Object1::funcA() {}
void Object1::funcC() {}

int main(int argc, char *argv[]) {
  Object1 obj;
  printf("Pointer size %zu bytes, obj1 address %p, obj1 size %zu bytes\n",
      sizeof(void *), &obj, sizeof(obj));
  print_field_address(&obj);
  printf("Address of Field data_10 %p\n", &obj.data_10);
  printf("Address of Field data_11 %p\n", &obj.data_11);
  printf("Address of Field data_12 %p\n", &obj.data_12);

  return 0;
}
```

上面这段代码的输出如下：
```
Pointer size 8 bytes, obj1 address 0x7ffd42bad690, obj1 size 64 bytes
Address of obj 0x7ffd42bad690, object size 40 bytes
Address of Field data_1 0x7ffd42bad698
Address of Field data_2 0x7ffd42bad69c
Address of Field data_3 0x7ffd42bad6a0
Address of Field data_4 0x7ffd42bad6a8
Address of Field data_5 0x7ffd42bad6ac
Address of Field data_6 0x7ffd42bad6b0
Address of Field data_10 0x7ffd42bad6b8
Address of Field data_11 0x7ffd42bad6c0
Address of Field data_12 0x7ffd42bad6c8
```

子类是否覆写父类的虚函数，以及覆写的父类虚函数的个数都不影响子类对象的内存布局。

## 多继承中子类对象的内存布局

这里设计一个类，继承两个父类。偷个懒，让两个父类的定义基本上一样。示例代码如下：
```
template<typename Object>
void print_field_address(Object *obj);

class Object {
public:
  Object();
  ~Object();
  void funcA();
  void funcB();
  int32_t data_1 = 0;
  int16_t data_2 = 0;
  int64_t data_3 = 0;
  void funcC();
  int32_t data_4 = 0;
private:
  void funcD();
  int32_t data_5 = 0;
  void funcE();
  int64_t data_6 = 0;

  friend void print_field_address<Object>(Object *obj);
};

Object::Object() {}
Object::~Object() {}
void Object::funcA() {}
void Object::funcB() {}
void Object::funcC() {}
void Object::funcD() {}
void Object::funcE() {}

class Object1 {
public:
  Object1();
  ~Object1();
  void funcA();
  void funcB();
  int32_t data_1 = 0;
  int16_t data_2 = 0;
  int64_t data_3 = 0;
  void funcC();
  int32_t data_4 = 0;
private:
  void funcD();
  int32_t data_5 = 0;
  void funcE();
  int64_t data_6 = 0;

  friend void print_field_address<Object1>(Object1 *obj);
};

Object1::Object1() {}
Object1::~Object1() {}
void Object1::funcA() {}
void Object1::funcB() {}
void Object1::funcC() {}
void Object1::funcD() {}
void Object1::funcE() {}

template<typename Object>
void print_field_address(Object *obj) {
  printf("Address of obj %p, object size %zu bytes\n", obj, sizeof(*obj));
  printf("Address of Field data_1 %p\n", &obj->data_1);
  printf("Address of Field data_2 %p\n", &obj->data_2);
  printf("Address of Field data_3 %p\n", &obj->data_3);
  printf("Address of Field data_4 %p\n", &obj->data_4);
  printf("Address of Field data_5 %p\n", &obj->data_5);
  printf("Address of Field data_6 %p\n", &obj->data_6);
}

class Object2: public Object, public Object1 {
public:
  int64_t data_10 = 0;
  int64_t data_11 = 0;
  int32_t data_12 = 0;
};

int main(int argc, char *argv[]) {
  Object2 obj2;
  printf("obj2 address %p, obj2 size %zu bytes\n", &obj2, sizeof(obj2));
  print_field_address<Object>(&obj2);
  print_field_address<Object1>(&obj2);
  printf("Address of Field data_10 %p\n", &obj2.data_10);
  printf("Address of Field data_11 %p\n", &obj2.data_11);
  printf("Address of Field data_12 %p\n", &obj2.data_12);

  return 0;
}
```

上面这段代码的输出如下：
```
obj2 address 0x7fff4a92dea0, obj2 size 88 bytes
Address of obj 0x7fff4a92dea0, object size 32 bytes
Address of Field data_1 0x7fff4a92dea0
Address of Field data_2 0x7fff4a92dea4
Address of Field data_3 0x7fff4a92dea8
Address of Field data_4 0x7fff4a92deb0
Address of Field data_5 0x7fff4a92deb4
Address of Field data_6 0x7fff4a92deb8
Address of obj 0x7fff4a92dec0, object size 32 bytes
Address of Field data_1 0x7fff4a92dec0
Address of Field data_2 0x7fff4a92dec4
Address of Field data_3 0x7fff4a92dec8
Address of Field data_4 0x7fff4a92ded0
Address of Field data_5 0x7fff4a92ded4
Address of Field data_6 0x7fff4a92ded8
Address of Field data_10 0x7fff4a92dee0
Address of Field data_11 0x7fff4a92dee8
Address of Field data_12 0x7fff4a92def0
```

在多继承中，子类对象的内存布局如下：

 * 子类对象的开始位置依次排列着各个父类对象的内容，各个父类对象的内容之间，按声明的顺序依次排列，各个父类对象的内容之间可能存在填充；
 * 子类自己的数据成员排在所有父类的所有内容之后，两者之间可能有填充；
 * 子类的数据成员在内存中按声明的顺序排列。

这里可以看下在多继承中，父类对象指针转换为子类对象指针时，`static_cast` `reinterpret_cast` 的不同行为。示例代码如下：
```
int main(int argc, char *argv[]) {
  Object2 obj2;
  Object1 *obj1p = &obj2;
  Object2 *obj2p = static_cast<Object2 *>(obj1p);
  Object2 *obj2p2 = reinterpret_cast<Object2 *>(obj1p);
  printf("obj1p %p, obj2p %p, obj2p2 %p\n", obj1p, obj2p, obj2p2);

  return 0;
}
```

上面这段代码的输出如下：
```
obj1p 0x7ffc763ca260, obj2p 0x7ffc763ca240, obj2p2 0x7ffc763ca260
```

可以看到，通过 `static_cast` 转换指针类型时，指针的值可能会根据类型信息作一些调整，而通过 `reinterpret_cast` 转换指针类型时，则是无脑将一个指针的类型强制转换为另一个类型，指针值不会根据类型作调整。

多继承的这段示例代码中，两个父类都没有虚函数。当两个父类有虚函数时，只要两个父类的虚函数的函数签名不同，则对子类对象的内存布局就没有任何影响，只是父类对象的内容会包含父类的虚函数表指针。示例代码如下：
```
template<typename Object>
void print_field_address(Object *obj);

class Object {
public:
  Object();
  virtual ~Object();
  void funcA();
  virtual void funcB();
  int32_t data_1 = 0;
  int16_t data_2 = 0;
  int64_t data_3 = 0;
  void funcC();
  int32_t data_4 = 0;
private:
  void funcD();
  int32_t data_5 = 0;
  void funcE();
  int64_t data_6 = 0;

  friend void print_field_address<Object>(Object *obj);
};

Object::Object() {}
Object::~Object() {}
void Object::funcA() {}
void Object::funcB() {}
void Object::funcC() {}
void Object::funcD() {}
void Object::funcE() {}

class Object1 {
public:
  Object1();
  virtual ~Object1();
  void funcA();
  void funcB();
  int32_t data_1 = 0;
  int16_t data_2 = 0;
  int64_t data_3 = 0;
  virtual void funcC();
  int32_t data_4 = 0;
private:
  void funcD();
  int32_t data_5 = 0;
  void funcE();
  int64_t data_6 = 0;

  friend void print_field_address<Object1>(Object1 *obj);
};

Object1::Object1() {}
Object1::~Object1() {}
void Object1::funcA() {}
void Object1::funcB() {}
void Object1::funcC() {}
void Object1::funcD() {}
void Object1::funcE() {}

template<typename Object>
void print_field_address(Object *obj) {
  printf("Address of obj %p, object size %zu bytes\n", obj, sizeof(*obj));
  printf("Address of Field data_1 %p\n", &obj->data_1);
  printf("Address of Field data_2 %p\n", &obj->data_2);
  printf("Address of Field data_3 %p\n", &obj->data_3);
  printf("Address of Field data_4 %p\n", &obj->data_4);
  printf("Address of Field data_5 %p\n", &obj->data_5);
  printf("Address of Field data_6 %p\n", &obj->data_6);
}

class Object2: public Object, public Object1 {
public:
  virtual ~Object2();
  virtual void funcF();
  int64_t data_10 = 0;
  int64_t data_11 = 0;
  int32_t data_12 = 0;
};

Object2::~Object2() {}

void Object2::funcF(){}

int main(int argc, char *argv[]) {
  Object2 obj2;
  printf("obj2 address %p, obj2 size %zu bytes\n", &obj2, sizeof(obj2));
  print_field_address<Object>(&obj2);
  print_field_address<Object1>(&obj2);
  printf("Address of Field data_10 %p\n", &obj2.data_10);
  printf("Address of Field data_11 %p\n", &obj2.data_11);
  printf("Address of Field data_12 %p\n", &obj2.data_12);

  return 0;
}
```

为了进一步了解声明虚函数对于类对象内存布局的影响，除了给两个父类声明了几个虚函数之外，还为子类专门声明了另外的虚函数。上面这段代码的输出如下：
```
obj2 address 0x7ffe732ac990, obj2 size 104 bytes
Address of obj 0x7ffe732ac990, object size 40 bytes
Address of Field data_1 0x7ffe732ac998
Address of Field data_2 0x7ffe732ac99c
Address of Field data_3 0x7ffe732ac9a0
Address of Field data_4 0x7ffe732ac9a8
Address of Field data_5 0x7ffe732ac9ac
Address of Field data_6 0x7ffe732ac9b0
Address of obj 0x7ffe732ac9b8, object size 40 bytes
Address of Field data_1 0x7ffe732ac9c0
Address of Field data_2 0x7ffe732ac9c4
Address of Field data_3 0x7ffe732ac9c8
Address of Field data_4 0x7ffe732ac9d0
Address of Field data_5 0x7ffe732ac9d4
Address of Field data_6 0x7ffe732ac9d8
Address of Field data_10 0x7ffe732ac9e0
Address of Field data_11 0x7ffe732ac9e8
Address of Field data_12 0x7ffe732ac9f0
```

这个子类对象的大小，仅比前面的整个继承体系中不包含任何虚函数的版本多了 16 个字节，也就是两个指针的大小。这也就意味着子类没有专门的虚函数表指针。

这里的 Object2 类对象的内存布局如下图所示。
```
 Low   |                                  |          
   |   |----------------------------------| <------ Object2 class object memory layout
   |   |           Object::vptr.          |---------|
   |   |----------------------------------|         |---------> |----------------------------|
   |   |     int32_t Object:: data_1      |                     |           . . .            |
  \|/  |----------------------------------|                     |----------------------------|
       |     int32_t Object:: data_2      |                     |           . . .            |
       |----------------------------------|                     |----------------------------|
       |     int32_t Object:: data_3      |
       |----------------------------------|
       |     int32_t Object:: data_4      | 
       |----------------------------------|
       |     int32_t Object:: data_5      |
       |----------------------------------|
       |     int32_t Object:: data_6      | 
       |----------------------------------|
       |          Object1::vptr           |---------|
       |----------------------------------|         |
       |     int32_t Object1:: data_1     |         |---------> |----------------------------|
       |----------------------------------|                     |           . . .            |
       |     int32_t Object1:: data_2     |                     |----------------------------|
       |----------------------------------|                     |           . . .            |
       |     int32_t Object1:: data_3     |                     |----------------------------|
       |----------------------------------|
       |     int32_t Object1:: data_4     |
       |----------------------------------|
       |     int32_t Object1:: data_5     |
       |----------------------------------|
       |     int32_t Object1:: data_6     |
       |----------------------------------|
       |    int64_t Object2:: data_10     |
       |----------------------------------|
       |    int64_t Object2:: data_11     |
       |----------------------------------|
       |    int64_t Object2:: data_12     |
-------|----------------------------------|------------
       |                o                 |
       |                o                 |
       |                o                 |
       |                                  |
       |                                  |
```

## C++ 虚函数的调用及虚函数表的结构

了解了 C++ 类对象中虚函数表指针的位置，我们也就不难借助于虚函数表指针，模拟虚函数的调用，示例代码如下：
```
template<typename Object>
void print_field_address(Object *obj);

class Object {
public:
  Object();
  virtual ~Object();
  void funcA();
  virtual void funcB();
  int32_t data_1 = 0;
  int16_t data_2 = 0;
  int64_t data_3 = 0;
  void funcC();
  int32_t data_4 = 0;
private:
  void funcD();
  int32_t data_5 = 0;
  void funcE();
  int64_t data_6 = 0;

  friend void print_field_address<Object>(Object *obj);
};

Object::Object() { printf("Object::ctor()\n"); }
Object::~Object() { printf("Object::~dtor()\n"); }
void Object::funcA() { printf("Object::funcA()\n"); }
void Object::funcB() { printf("Object::funcB()\n"); }
void Object::funcC() { printf("Object::funcC()\n"); }
void Object::funcD() { printf("Object::funcD()\n"); }
void Object::funcE() { printf("Object::funcE()\n"); }

class Object1 {
public:
  Object1();
  virtual ~Object1();
  void funcA();
  void funcB();
  int32_t data_1 = 0;
  int16_t data_2 = 0;
  int64_t data_3 = 0;
  virtual void funcC();
  int32_t data_4 = 0;
private:
  void funcD();
  int32_t data_5 = 0;
  void funcE();
  int64_t data_6 = 0;

  friend void print_field_address<Object1>(Object1 *obj);
};

Object1::Object1() { printf("Object1::ctor()\n"); }
Object1::~Object1() { printf("Object1::~dtor()\n"); }
void Object1::funcA() { printf("Object1::funcA()\n"); }
void Object1::funcB() { printf("Object1::funcB()\n"); }
void Object1::funcC() { printf("Object1::funcC()\n"); }
void Object1::funcD() { printf("Object1::funcD()\n"); }
void Object1::funcE() { printf("Object1::funcE()\n"); }

class Object2: public Object, public Object1 {
public:
  Object2();
  virtual ~Object2();
  virtual void funcF();
  int64_t data_10 = 0;
  int64_t data_11 = 0;
  int32_t data_12 = 0;
};

Object2::Object2() { printf("Object2::ctor()\n"); }
Object2::~Object2() { printf("Object2::~dtor()\n"); }

void Object2::funcF(){ printf("Object2::funcF()\n"); }

int main(int argc, char *argv[]) {
  Object2 obj2;
  Object2 obj2_01;

  printf("obj2 address %p, obj2 size %zu bytes\n", &obj2, sizeof(obj2));

  printf("Object vtable address %p, Object1 vtable address %p\n",
      *reinterpret_cast<void **>(&obj2),
      *reinterpret_cast<void **>(static_cast<Object1 *>(&obj2)));

  printf("Object vtable address %p, Object1 vtable address %p\n",
        *reinterpret_cast<void **>(&obj2_01),
        *reinterpret_cast<void **>(static_cast<Object1 *>(&obj2_01)));

  void ** vtable = (void**)(*reinterpret_cast<void **>(&obj2));
  void ** child_vtable = (void**)(*reinterpret_cast<void **>(static_cast<Object1 *>(&obj2)));

  printf("vtable[0] %p, vtable[1] %p, child_vtable[0] %p, child_vtable[1] %p\n",
      vtable[0], vtable[1], child_vtable[0], child_vtable[1]);

  for (int i = 2; i < 10; ++i) {
    void (*vmem_func)(Object *) = reinterpret_cast<void (*)(Object *)>(vtable[i]);
    printf("vmem_func %p start %d\n", vmem_func, i);
    vmem_func(&obj2);
    printf("vmem_func end\n");
  }

  return 0;
}
```

在上面这段代码中，我们首先获得虚函数表指针的地址，也就是类对象的地址，我们把类对象的地址强制类型转换为指针的指针，然后对它解引用，这样就得到了虚函数表的地址；虚函数表是一个函数指针的数组，也就是指针数组，因而将指向虚函数表的指针强制类型转换为指针数组；虚函数表中各个函数指针指向的函数的原型，除了第一个参数为类对象指针外，其它参数就是类虚函数声明中的参数，在这里由于各个虚函数的函数原型都一样，所以省去了一些麻烦。

上面这段代码的输出如下：
```
Object::ctor()
Object1::ctor()
Object2::ctor()
Object::ctor()
Object1::ctor()
Object2::ctor()
obj2 address 0x7ffd921d19e0, obj2 size 104 bytes
Object vtable address 0x558b593eec80, Object1 vtable address 0x558b593eecb0
Object vtable address 0x558b593eec80, Object1 vtable address 0x558b593eecb0
vtable[0] 0x558b593ec5c2, vtable[1] 0x558b593ec628, child_vtable[0] 0x558b593ec61d, child_vtable[1] 0x558b593ec657
vmem_func 0x558b593ec300 start 2
Object::funcB()
vmem_func end
vmem_func 0x558b593ec662 start 3
Object2::funcF()
vmem_func end
vmem_func 0xffffffffffffffd8 start 4
```

通过上面的方法调用各个 C++ 类对象的虚函数表中指向的各个函数，可以发现虚函数表的结构：

 * 虚函数表属于类，而不是类对象，尽管每个类对象中都可能有一个或多个虚函数表指针，但同一个类的不同实例之间，相对应的虚函数表指针都指向相同的虚函数表；
 * 虚函数表的前两个元素都指向类对象的析构函数，其中第二个析构函数是会释放内存的版本；
 * 两个父类的虚函数表中，指向的析构函数不完全一样，但这些析构函数执行的代码几乎完全一样；
 * 子类没有单独的虚函数表，而是与它的第一个父类共用一个虚函数表；
 * 子类的第一个父类的虚函数表的结构为，第一个父类的虚函数，后跟子类特有的虚函数；
 * 第一个父类的虚函数表的地址与第二个父类的虚函数表的地址非常接近，就像同一个一样。

补充上虚函数表的内部结构，Object2 类对象的内存布局如下图所示。

```
 Low   |                                  |          
   |   |----------------------------------| <------ Object2 class object memory layout
   |   |           Object::vptr.          |---------|
   |   |----------------------------------|         |---------> |----------------------------|
   |   |     int32_t Object:: data_1      |                     |    Object2::~Object2()     |
  \|/  |----------------------------------|                     |----------------------------|
       |     int32_t Object:: data_2      |                     |    Object2::~Object2()     |
       |----------------------------------|                     |----------------------------|
       |     int32_t Object:: data_3      |                     |      Object::funcB()       |
       |----------------------------------|                     |----------------------------|
       |     int32_t Object:: data_4      |                     |      Object2::funcF()      |
       |----------------------------------|                     |----------------------------|
       |     int32_t Object:: data_5      |
       |----------------------------------|
       |     int32_t Object:: data_6      |
       |----------------------------------|
       |          Object1::vptr           |---------|
       |----------------------------------|         |
       |     int32_t Object1:: data_1     |         |---------> |----------------------------|
       |----------------------------------|                     |    Object2::~Object2()     |
       |     int32_t Object1:: data_2     |                     |----------------------------|
       |----------------------------------|                     |    Object2::~Object2()     |
       |     int32_t Object1:: data_3     |                     |----------------------------|
       |----------------------------------|                     |      Object1::funcC()      |
       |     int32_t Object1:: data_4     |                     |----------------------------|
       |----------------------------------|
       |     int32_t Object1:: data_5     |
       |----------------------------------|
       |     int32_t Object1:: data_6     |
       |----------------------------------|
       |    int64_t Object2:: data_10     |
       |----------------------------------|
       |    int64_t Object2:: data_11     |
       |----------------------------------|         |
       |    int64_t Object2:: data_12     |
-------|----------------------------------|------------
       |                o                 |
       |                o                 |
       |                o                 |
       |                                  |
       |                                  |
```

## C++ 棱形继承与虚继承

复杂的多继承继承层次结构中，一不小心可能就会搞出棱形继承，即某个子类继承了多个父类，而这多个父类又继承了某个相同的父类。如下面这个例子：
```
template<typename Object>
void print_field_address(Object *obj);

class ObjectBase {
public:
  ObjectBase();
  virtual ~ObjectBase();

  void funcA();
  void funcH();

  virtual void funcG();

  int64_t data_31 = 0;
};

ObjectBase::ObjectBase() {}
ObjectBase::~ObjectBase() {}
void ObjectBase::funcA() {}
void ObjectBase::funcG() {}
void ObjectBase::funcH() {}

class Object : public ObjectBase{
public:
  Object();
  virtual ~Object();
  void funcA();
  virtual void funcB(int a);
  int32_t data_1 = 0;
  int16_t data_2 = 0;
  int64_t data_3 = 0;
  virtual void funcC();
  int32_t data_4 = 0;
private:
  void funcD();
  int32_t data_5 = 0;
  void funcE();
  int64_t data_6 = 0;

  void funcG() override;

  friend void print_field_address<Object>(Object *obj);
};

Object::Object() { printf("Object::ctor()\n"); }
Object::~Object() { printf("Object::~dtor()\n"); }
void Object::funcA() { printf("Object::funcA()\n"); }
void Object::funcB(int a) { printf("Object::funcB()\n"); }
void Object::funcC() { printf("Object::funcC()\n"); }
void Object::funcD() { printf("Object::funcD()\n"); }
void Object::funcE() { printf("Object::funcE()\n"); }
void Object::funcG() { printf("Object::funcG()\n"); }

class Object1 : public ObjectBase{
public:
  Object1();
  virtual ~Object1();
  void funcA();
  virtual void funcB();
  int32_t data_1 = 0;
  int16_t data_2 = 0;
  int64_t data_3 = 0;
  virtual void funcC();
  int32_t data_4 = 0;
private:
  void funcD();
  int32_t data_5 = 0;
  void funcE();
  int64_t data_6 = 0;
  void funcG() override;

  friend void print_field_address<Object1>(Object1 *obj);
};

Object1::Object1() { printf("Object1::ctor()\n"); }
Object1::~Object1() { printf("Object1::~dtor()\n"); }
void Object1::funcA() { printf("Object1::funcA()\n"); }
void Object1::funcB() { printf("Object1::funcB()\n"); }
void Object1::funcC() { printf("Object1::funcC()\n"); }
void Object1::funcD() { printf("Object1::funcD()\n"); }
void Object1::funcE() { printf("Object1::funcE()\n"); }
void Object1::funcG() { printf("Object1::funcG()\n"); }

template<typename Object>
void print_field_address(Object *obj) {
  printf("Address of obj %p, object size %zu bytes\n", obj, sizeof(*obj));
  printf("Address of Field data_31 %p\n", &obj->data_31);
  printf("Address of Field data_1 %p\n", &obj->data_1);
  printf("Address of Field data_2 %p\n", &obj->data_2);
  printf("Address of Field data_3 %p\n", &obj->data_3);
  printf("Address of Field data_4 %p\n", &obj->data_4);
  printf("Address of Field data_5 %p\n", &obj->data_5);
  printf("Address of Field data_6 %p\n", &obj->data_6);
}

class Object2: public Object, public Object1 {
public:
  Object2();
  virtual ~Object2();
  virtual void funcF();
  void funcG() override;
  int64_t data_10 = 0;
  int64_t data_11 = 0;
  int32_t data_12 = 0;
};

Object2::Object2() { printf("Object2::ctor()\n"); }
Object2::~Object2() { printf("Object2::~dtor()\n"); }

void Object2::funcF(){ printf("Object2::funcF()\n"); }
void Object2::funcG(){ printf("Object2::funcG()\n"); }

int case1(int argc, char *argv[]) {
  Object2 obj2;
  printf("obj2 address %p, obj2 size %zu bytes\n", &obj2, sizeof(obj2));
  print_field_address<Object>(&obj2);
  print_field_address<Object1>(&obj2);
  obj2.funcG();
//  obj2.funcH();
//  obj2.data_31 = 1;

  return 0;
}
```

在这个例子中，有这样两条继承链，`Object2` -> `Object` -> `ObjectBase`，`Object2` -> `Object1` -> `ObjectBase`，`Object2` 的两个父类 `Object` 和 `Object1` 都继承了相同的 `ObjectBase`，从而形成了棱形继承。

在这个继承层次结构中，最顶层的 `ObjectBase` 有两个虚函数（析构函数和 `funcG()`），一个数据成员和一个一般成员函数。`Object`、`Object1` 和 `Object2` 都覆写了 `ObjectBase` 的虚函数 `funcG()`。

上面这段代码编译执行都没问题，其执行结果如下：
```
Object::ctor()
Object1::ctor()
Object2::ctor()
obj2 address 0x7fff11beed50, obj2 size 120 bytes
Address of obj 0x7fff11beed50, object size 48 bytes
Address of Field data_31 0x7fff11beed58
Address of Field data_1 0x7fff11beed60
Address of Field data_2 0x7fff11beed64
Address of Field data_3 0x7fff11beed68
Address of Field data_4 0x7fff11beed70
Address of Field data_5 0x7fff11beed74
Address of Field data_6 0x7fff11beed78
Address of obj 0x7fff11beed80, object size 48 bytes
Address of Field data_31 0x7fff11beed88
Address of Field data_1 0x7fff11beed90
Address of Field data_2 0x7fff11beed94
Address of Field data_3 0x7fff11beed98
Address of Field data_4 0x7fff11beeda0
Address of Field data_5 0x7fff11beeda4
Address of Field data_6 0x7fff11beeda8
Object2::funcG()
Object2::~dtor()
Object1::~dtor()
Object::~dtor()
```

借助于上面这段示例代码，可以发现：

 * 相对于 `Object` 和 `Object1` 没有继承 `ObjectBase` 的情况，`Object2` 对象的大小仅仅增加了两个 `ObjectBase` 数据成员的大小，`ObjectBase` 的虚函数表在 `Object2` 对象中没有占用额外的空间，实际上它本身也没有专门的虚函数表，与 `Object2` 对象关联的虚函数表有两个，一个用来访问（`ObjectBase` + `Object` + `Object2`）的虚函数，另一个用来访问（`ObjectBase` + `Object1`）的虚函数；
 * 不访问最顶层的基类的成员（成员函数、成员变量和虚成员函数）的话，代码编译成功完全没问题，代码执行也没问题；
 * 当只有 `Object` 和 `Object1` 覆写了 `ObjectBase` 的虚函数 `funcG()`，而 `Object2` 没有，则在通过 `Object2` 的对象调用 `funcG()` 函数时，编译报错，类似于下面：
```
./src/linux_audio.cpp: In function ‘int case1(int, char**)’:
../src/linux_audio.cpp:588:8: error: request for member ‘funcG’ is ambiguous
  588 |   obj2.funcG();
      |        ^~~~~
../src/linux_audio.cpp:490:6: note: candidates are: ‘virtual void ObjectBase::funcG()’
  490 | void ObjectBase::funcG() {}
      |      ^~~~~~~~~~
../src/linux_audio.cpp:552:6: note:                 ‘virtual void Object1::funcG()’
  552 | void Object1::funcG() { printf("Object1::funcG()\n"); }
      |      ^~~~~~~
../src/linux_audio.cpp:522:6: note:                 ‘virtual void Object::funcG()’
  522 | void Object::funcG() { printf("Object::funcG()\n"); }
      |      ^~~~~~
make: *** [src/subdir.mk:26：src/linux_audio.o] 错误 1
"make all" terminated with exit code 2. Build might be incomplete.
```
 * `Object`、`Object1` 和 `Object2` 都覆写了 `ObjectBase` 的虚函数 `funcG()` 时，通过 `Object2` 的对象调用 `funcG()` 函数，无论是编译还是执行，代码都没有问题。
 * 继承层次最顶层的父类中的数据成员，在其每个子类对象中都有一份，在继承层次最底层的类对象中实际上有多份。
 * 通过 `Object2` 类的对象，访问 `ObjectBase` 的成员函数，由于编译器无法确认是应该将 `Object2` 类对象的 `Object` 部分，还是 `Object1` 部分传给 `ObjectBase` 的成员函数，在编译时报错，类似于下面这样：
```
../src/linux_audio.cpp:589:8: error: request for member ‘funcH’ is ambiguous
  589 |   obj2.funcH();
      |        ^~~~~
../src/linux_audio.cpp:491:6: note: candidates are: ‘void ObjectBase::funcH()’
  491 | void ObjectBase::funcH() {}
      |      ^~~~~~~~~~
../src/linux_audio.cpp:491:6: note:                 ‘void ObjectBase::funcH()’
make: *** [src/subdir.mk:26：src/linux_audio.o] 错误 1
"make all" terminated with exit code 2. Build might be incomplete.
```

C++ 中的虚继承机制正是用来解决这里的，通过继承层次结构中最底层的类的对象，访问继承层次结构中最顶层的类的数据成员时，符号解析出现歧义而编译报错的问题的。我们让 `Object1` 和 `Object` 虚继承 `ObjectBase`：
```
template<typename Object>
void print_field_address(Object *obj);

class ObjectBase {
public:
  ObjectBase();
  virtual ~ObjectBase();

  void funcA();
  void funcH();

  virtual void funcG();

  int64_t data_31 = 0;
  int64_t data_32 = 0;
};

ObjectBase::ObjectBase() {}
ObjectBase::~ObjectBase() {}
void ObjectBase::funcA() {}
void ObjectBase::funcG() {}
void ObjectBase::funcH() {}

class Object : virtual public ObjectBase{
public:
  Object();
  virtual ~Object();
  void funcA();
  virtual void funcB(int a);
  int32_t data_1 = 0;
  int16_t data_2 = 0;
  int64_t data_3 = 0;
  virtual void funcC();
  int32_t data_4 = 0;
private:
  void funcD();
  int32_t data_5 = 0;
  void funcE();
  int64_t data_6 = 0;

  void funcG() override;

  friend void print_field_address<Object>(Object *obj);
};

Object::Object() { printf("Object::ctor()\n"); }
Object::~Object() { printf("Object::~dtor()\n"); }
void Object::funcA() { printf("Object::funcA()\n"); }
void Object::funcB(int a) { printf("Object::funcB()\n"); }
void Object::funcC() { printf("Object::funcC()\n"); }
void Object::funcD() { printf("Object::funcD()\n"); }
void Object::funcE() { printf("Object::funcE()\n"); }
void Object::funcG() { printf("Object::funcG()\n"); }

class Object1 : virtual public ObjectBase{
public:
  Object1();
  virtual ~Object1();
  void funcA();
  void funcB();
  int32_t data_1 = 0;
  int16_t data_2 = 0;
  int64_t data_3 = 0;
  void funcC();
  int32_t data_4 = 0;
private:
  virtual void funcD();
  int32_t data_5 = 0;
  virtual void funcE();
  int64_t data_6 = 0;
  void funcG() override;

  friend void print_field_address<Object1>(Object1 *obj);
};

Object1::Object1() { printf("Object1::ctor()\n"); }
Object1::~Object1() { printf("Object1::~dtor()\n"); }
void Object1::funcA() { printf("Object1::funcA()\n"); }
void Object1::funcB() { printf("Object1::funcB()\n"); }
void Object1::funcC() { printf("Object1::funcC()\n"); }
void Object1::funcD() { printf("Object1::funcD()\n"); }
void Object1::funcE() { printf("Object1::funcE()\n"); }
void Object1::funcG() { printf("Object1::funcG()\n"); }

template<typename Object>
void print_field_address(Object *obj) {
  printf("Address of obj %p, object size %zu bytes\n", obj, sizeof(*obj));

  printf("Address of Field data_1 %p\n", &obj->data_1);
  printf("Address of Field data_2 %p\n", &obj->data_2);
  printf("Address of Field data_3 %p\n", &obj->data_3);
  printf("Address of Field data_4 %p\n", &obj->data_4);
  printf("Address of Field data_5 %p\n", &obj->data_5);
  printf("Address of Field data_6 %p\n", &obj->data_6);

  ObjectBase *base_obj = obj;
  printf("Address of base obj %p, object size %zu bytes\n", base_obj, sizeof(*base_obj));
  printf("Address of Field data_31 %p\n", &obj->data_31);
  printf("Address of Field data_32 %p\n", &obj->data_32);
}

class Object2: public Object, public Object1 {
public:
  Object2();
  virtual ~Object2();
  virtual void funcF();
  void funcG() override;
  int64_t data_10 = 0;
  int64_t data_11 = 0;
  int32_t data_12 = 0;
};

Object2::Object2() { printf("Object2::ctor()\n"); }
Object2::~Object2() { printf("Object2::~dtor()\n"); }

void Object2::funcF(){ printf("Object2::funcF()\n"); }
void Object2::funcG(){ printf("Object2::funcG()\n"); }

int case1(int argc, char *argv[]) {
  Object2 obj2;
  printf("obj2 address %p, obj2 size %zu bytes\n", &obj2, sizeof(obj2));
  print_field_address<Object>(&obj2);
  print_field_address<Object1>(&obj2);
  printf("Address of Field data_10 %p\n", &obj2.data_10);
  printf("Address of Field data_11 %p\n", &obj2.data_11);
  printf("Address of Field data_12 %p\n", &obj2.data_12);
  obj2.funcG();
//  obj2.funcH();
  obj2.data_31 = 1;

  void ** vtable = (void**)(*reinterpret_cast<void **>(static_cast<ObjectBase *>(&obj2)));
  for (int i = 2; i < 10; ++i) {
    void (*vmem_func)(ObjectBase *) = reinterpret_cast<void (*)(ObjectBase *)>(vtable[i]);
    printf("vmem_func %p start %d\n", vmem_func, i);
    vmem_func(&obj2);
    printf("vmem_func end\n");
  }
  return 0;
}
```

上面这段代码输出如下：
```
Object::ctor()
Object1::ctor()
Object2::ctor()
obj2 address 0x7ffe8b59eca0, obj2 size 128 bytes
Address of obj 0x7ffe8b59eca0, object size 64 bytes
Address of Field data_1 0x7ffe8b59eca8
Address of Field data_2 0x7ffe8b59ecac
Address of Field data_3 0x7ffe8b59ecb0
Address of Field data_4 0x7ffe8b59ecb8
Address of Field data_5 0x7ffe8b59ecbc
Address of Field data_6 0x7ffe8b59ecc0
Address of base obj 0x7ffe8b59ed08, object size 24 bytes
Address of Field data_31 0x7ffe8b59ed10
Address of Field data_32 0x7ffe8b59ed18
Address of obj 0x7ffe8b59ecc8, object size 64 bytes
Address of Field data_1 0x7ffe8b59ecd0
Address of Field data_2 0x7ffe8b59ecd4
Address of Field data_3 0x7ffe8b59ecd8
Address of Field data_4 0x7ffe8b59ece0
Address of Field data_5 0x7ffe8b59ece4
Address of Field data_6 0x7ffe8b59ece8
Address of base obj 0x7ffe8b59ed08, object size 24 bytes
Address of Field data_31 0x7ffe8b59ed10
Address of Field data_32 0x7ffe8b59ed18
Address of Field data_10 0x7ffe8b59ecf0
Address of Field data_11 0x7ffe8b59ecf8
Address of Field data_12 0x7ffe8b59ed00
Object2::funcG()
vmem_func 0x55b311c2c96b start 2
Object2::funcG()
vmem_func end
vmem_func 0x55b311c317c8 start 3
```

由这段输出可以看到：

 * 相对于前面没有使用虚继承的代码，继承层次结构中最底层的类 `Object2` 的对象大小，跟它的两个直接子类的大小的和相等，`Object2` 自己的成员变量就像没有占用任何内存空间一样，被虚继承的顶层父类的内容被算在了每个继承它的类对象的大小中了，但计算最底层的类对象的大小时只需要被计算一次；
 * `Object2` 类对象的内存布局为：第一个子类的全部内容（不包括被虚继承的类的内容，只包括这个父类的虚函数表指针和数据成员变量） -> 后面是第二个子类的全部内容（不包括被虚继承的类的内容，只包括这个父的虚函数表指针和数据成员变量） -> 继承层次中最底层的子类的数据成员变量（它的虚函数表与它的第一个父类共用） -> 被虚继承的类的内容，首先是虚函数表的指针，然后是各个数据成员。

这里的 Object2 类对象的内存布局如下图所示。左侧的箭头指向内存地址增加的方向。C++ 类对象的内存布局，与对象是在堆上分配还是在栈上分配无关。
```
 Low   |                                  |          
   |   |----------------------------------| <------ Object2 class object memory layout
   |   |           Object::vptr.          |---------|
   |   |----------------------------------|         |---------> |----------------------------|
   |   |     int32_t Object:: data_1      |                     |    Object2::~Object2()     |
  \|/  |----------------------------------|                     |----------------------------|
       |     int32_t Object:: data_2      |                     |    Object2::~Object2()     |
       |----------------------------------|                     |----------------------------|
       |     int32_t Object:: data_3      |                     |      Object::funcB()       |
       |----------------------------------|                     |----------------------------|
       |     int32_t Object:: data_4      |                     |      Object::funcC()       |
       |----------------------------------|                     |----------------------------|
       |     int32_t Object:: data_5      |                     |      Object2::funcG()      |
       |----------------------------------|                     |----------------------------|
       |     int32_t Object:: data_6      |                     |      Object2::funcF()      |
       |----------------------------------|                     |----------------------------|
       |          Object1::vptr           |---------|
       |----------------------------------|         |
       |     int32_t Object1:: data_1     |         |---------> |----------------------------|
       |----------------------------------|                     |    Object2::~Object2()     |
       |     int32_t Object1:: data_2     |                     |----------------------------|
       |----------------------------------|                     |    Object2::~Object2()     |
       |     int32_t Object1:: data_3     |                     |----------------------------|
       |----------------------------------|                     |      Object1::funcD()      |
       |     int32_t Object1:: data_4     |                     |----------------------------|
       |----------------------------------|                     |      Object1::funcE()      |
       |     int32_t Object1:: data_5     |                     |----------------------------|
       |----------------------------------|                     |      Object2::funcG()      |
       |     int32_t Object1:: data_6     |                     |----------------------------|
       |----------------------------------|
       |    int64_t Object2:: data_10     |
       |----------------------------------|
       |    int64_t Object2:: data_11     |         |---------> |----------------------------|
       |----------------------------------|         |           |    Object2::~Object2()     |
       |    int64_t Object2:: data_12     |         |           |----------------------------|
       |----------------------------------|         |           |    Object2::~Object2()     |
       |         ObjectBase::vptr         |---------|           |----------------------------|
       |----------------------------------|                     |      Object2::funcG()      |
       |   int64_t ObjectBase:: data_31   |                     |----------------------------|
       |----------------------------------|
       |   int64_t ObjectBase:: data_31   |
-------|----------------------------------|------------
       |                o                 |
       |                o                 |
       |                o                 |
       |                                  |
       |                                  |
```

**参考文档**

[C++虚继承和虚基类](http://c.biancheng.net/cpp/biancheng/view/238.html)

[Memory Layout of C++ Object in Different Scenarios](http://www.vishalchovatiya.com/memory-layout-of-cpp-object/)

[memory layout C++ objects [closed]](https://stackoverflow.com/questions/1632600/memory-layout-c-objects)
