---
layout : post
title: C++对象模型实例分析
tagline: 虚函数，虚继承以及对象内存结构
category: C++
tags:
- C++
---
* 
{:toc}

## C++ 多态的底层支持
C++ 中使用非指针，非引用的对象来调用函数时，在编译阶段其调用的函数地址就已经确定下来，没有（也不需要）动态绑定的效果。即使是多个重名函数，也可以经过函数匹配确定为其中一个，即静态多态。

此时对象的数据和函数其实是”分裂“的，对象所在的内存当中没有关于类型以及所属函数的信息。

通过父类的指针或引用来调用虚函数，能调用到子类对象实际上实现的函数，即子类中重写的虚函数。这就是动态多态。而要实现延迟绑定，需要类的类型信息与对象的数据”同步“存在。即可以通过检查对象数据中的某个字段，得到整块内存的类型信息。

因此 C++ 中引入了虚函数表来指明对象的类型、关联所属的函数。通过在对象中放置虚函数表指针 `vptr` 来关联内存与类型。在运行时通过读取虚表中的内容，来实现多态性。

本文通过分析一些简单的例子，尝试解释这一整套流程。
 
## 虚函数表的基本结构

### 最基础的情况
最基础的情况就是单继承，没有虚继承的情况。

```c++
#include <iostream>
using namespace std;

class B
{
public:
    int ib;
    char cb;

    B() : ib(0xBB), cb('A') {}
    virtual void f()  { ++ib; cout << "B::f()" << endl; }
    virtual void Bf() { --ib; cout << "B::Bf()" << endl; }
};

class D : public B
{
public:
    int id;
    char cd;

    D() : id(0xDD), cd('D') {}
    virtual void f()  { ++cb; cout << "D::f()" << endl; }
    virtual void Df() { --cb; cout << "D::Df()" << endl; }
};

int main()
{
    D d;
    B * b_ptr = &d;
    d.Bf();
    b_ptr->Bf();
}
```

在 g++ 中使用 `-fdump-class-hierarchy` 或者 `-fdump-lang-class` 选项，又或者在 VisualStudio 的 `项目 -> 属性 -> 配置属性 -> C/C++ -> 命令行 -> 其他选项` 中添加选项 `/d1reportAllClassLayout` 可以导出对象的虚函数表结构。

这里我用的 g++ 版本是 gcc version 7.5.0 (Ubuntu 7.5.0-3ubuntu1~18.04) ，通过  
`g++ -g -o single ./single_hierarchy.cpp -fdump-class-hierarchy`   
得到上述两个类的虚函数表和对象内存布局，再经过 `c++filt` 进行 `demangle` 得到易于阅读的符号名称。

#### 虚表布局
```
Vtable for B
B::vtable for B: 4 entries
0     (int (*)(...))0
8     (int (*)(...))(& typeinfo for B)
16    (int (*)(...))B::f
24    (int (*)(...))B::Bf

Class B
   size=16 align=8
   base size=13 base align=8
B (0x0x7f3d6ce36720) 0
    vptr=((& B::vtable for B) + 16)

Vtable for D
D::vtable for D: 5 entries
0     (int (*)(...))0
8     (int (*)(...))(& typeinfo for D)
16    (int (*)(...))D::f
24    (int (*)(...))B::Bf
32    (int (*)(...))D::Df

Class D
   size=24 align=8
   base size=21 base align=8
D (0x0x7f3d6ce6f958) 0
    vptr=((& D::vtable for D) + 16)
  B (0x0x7f3d6ce76660) 0
      primary-for D (0x0x7f3d6ce6f958)
```

再结合`gdb`来查看实际程序中的内存布局：

```c++
(gdb) info locals
d = {<B> = {_vptr.B = 0x8201d28 <vtable for D+16>, ib = 187, cb = 65 'A'}, id = 221, cd = 68 'D'}
b_ptr = 0x7ffffffee180
(gdb) x/3xg &d
0x7ffffffee180: 0x0000000008201d28      0x00000041000000b9
0x7ffffffee190: 0x00007f44000000dd
(gdb) x/5xg 0x8201d18
0x8201d18 <_ZTV1D>:     0x0000000000000000      0x0000000008201d60
0x8201d28 <_ZTV1D+16>:  0x0000000008000caa      0x0000000008000c26
0x8201d38 <_ZTV1D+32>:  0x0000000008000cf2
```
这里查看对象 d 的内存布局 `x/3xg &d` 中的 3 是从类 D 的大小（24/8）得到。  
查看虚函数表的布局 `x/5xg 0x8201d18` 中的 5 是从 `D::vtable for D: 5 entries` 这里看到有 5 个表项， `0x8201d18` 则是 `_vptr.B = 0x8201d28` 减去偏移 16 得到。  

通过 `nm -Cn ./single` 得到符号的地址信息  

```
// 仅展示相关部分
...
0000000000000bb2 W B::B()
0000000000000bb2 W B::B()
0000000000000bde W B::f()
0000000000000c26 W B::Bf()
0000000000000c6e W D::D()
0000000000000c6e W D::D()
0000000000000caa W D::f()
0000000000000cf2 W D::Df()
...
0000000000201d18 V vtable for D
0000000000201d40 V vtable for B
0000000000201d60 V typeinfo for D
0000000000201d78 V typeinfo for B
...

```
可以看到对象 d 中起始位置存储的是 `vptr`，指向了类 D 的 vtable + 16  `0x0000000008201d28`。  
这里与 `nm` 命令导出的地址信息相差了 `0x8000000`，我想应该是 `nm` 中给出的地址是段内偏移，而 `.text` 段的开始地址是`0x8000000`，所以需要加上这么多来得到运行时的绝对地址。可以查看进程的 `mmap` 信息确认。    

继续查看类 D 的 vtable，发现其中存储的信息确实与`g++`导出的布局一一对应。  
这里可以看出对象内部的 `vptr` 并不是指向虚函数表的起始位置，而是越过了开头的两项。  
第二项从名字上就可以看出指向的是类的类型信息。那么第一项是什么呢？从这个简单的样例还看不出所以然来，不过从其他资料可知，虚函数表的第一项是用来指明继承体系中不同类型的指针指向当前对象实例时，`this` 指针与对象的起始位置之间的偏移。一个派生类通过合法的向上转型成不同的父类或祖类指针后，这些指针指向的其实是对象实例的不同部分。

#### 虚表指针初试锋芒
将上述代码输入到[这个网站](https://godbolt.org/)可以观察到编译后的汇编代码。
![](/assets/img/c++_assemble_1.png)  
`d.Bf();` 这条语句是使用对象直接调用虚函数，在汇编代码的第 123 行可以看出在编译期就确定了是直接调用函数 `B::Bf()`。而后一条语句`b_ptr->Bf();`通过指针来调用虚函数时，则多出了几步：

```asm
mov     rax, QWORD PTR [rbp-8]      // b_ptr 的数值
mov     rax, QWORD PTR [rax]        // 获取虚函数表的地址（跳过前两项）
add     rax, 8                      // 第二个函数表项的地址（实际是第四项）
mov     rax, QWORD PTR [rax]        // 虚函数 B::Bf() 的地址
mov     rdx, QWORD PTR [rbp-8]      // this 作为参数
mov     rdi, rdx
call    rax                         // 调用 B::Bf()
```
这就是虚表指针发挥作用的时刻之一，动态绑定所运行的函数。

### 多继承的情况
类 C 同时继承类 A 和类 B

```c++
#include <string>
#include <iostream>
using namespace std;
class A {
public:
    int i1;
    char c1;

    A() : i1(0xAA), c1('A') {}
    virtual void f()  { cout << "A::f()"  << endl; }
    virtual void f1() { cout << "A::f1()" << endl; }
};

class B {
public:
    int i2;
    char c2;

    B() : i2(0xBB), c2('B') {}
    virtual void f()  { cout << "B::f()" << endl; }
    virtual void f2() { cout << "B::f2()" << endl; }
};

class C : public B, public A
{
public:
    int i3;
    char c3;
    C() : i3(0xCC), c3('C') {}
    virtual void f()  { cout << "C::f()" << endl; }
    virtual void f2() { cout << "C::f2()" << endl; }
};

int main(void)
{
    C c;
    C * c_ptr = &c;
    B * b_ptr = &c;
    A * a_ptr = &c;

    cout << "c_ptr : " << c_ptr << endl
         << "b_ptr : " << b_ptr << endl
         << "a_ptr : " << a_ptr << endl;

    long* data = (long *)* (long *)b_ptr;   
    cout << hex << data << endl;

    typedef void(*Func)(void *);
    Func fun = (Func) data[0];
    fun(b_ptr);
}
```

#### 虚表布局

```c++
Vtable for A
A::vtable for A: 4 entries
0     (int (*)(...))0
8     (int (*)(...))(& typeinfo for A)
16    (int (*)(...))A::f
24    (int (*)(...))A::f1

Class A
   size=16 align=8
   base size=13 base align=8
A (0x0x7f188fae6720) 0
    vptr=((& A::vtable for A) + 16)

Vtable for B
B::vtable for B: 4 entries
0     (int (*)(...))0
8     (int (*)(...))(& typeinfo for B)
16    (int (*)(...))B::f
24    (int (*)(...))B::f2

Class B
   size=16 align=8
   base size=13 base align=8
B (0x0x7f188fb25660) 0
    vptr=((& B::vtable for B) + 16)

Vtable for C
C::vtable for C: 9 entries
0     (int (*)(...))0
8     (int (*)(...))(& typeinfo for C)
16    (int (*)(...))C::f
24    (int (*)(...))C::f2
32    (int (*)(...))C::f3
40    (int (*)(...))-16
48    (int (*)(...))(& typeinfo for C)
56    (int (*)(...))C::non-virtual thunk to C::f()
64    (int (*)(...))A::f1

Class C
   size=40 align=8
   base size=37 base align=8
C (0x0x7fefa3b89d20) 0
    vptr=((& C::vtable for C) + 16)
  B (0x0x7fefa3b96900) 0
      primary-for C (0x0x7fefa3b89d20)
  A (0x0x7fefa3b96960) 16
      vptr=((& C::vtable for C) + 56)
```

再接着看到 `gdb` 中读取出的实际运行期数据：

```c++
// 运行到了最后一行
(gdb) info locals
c = {<B> = {_vptr.B = 0x8201ca0 <vtable for C+16>, i2 = 187, c2 = 66 'B'}, <A> = {_vptr.A = 0x8201cc8 <vtable for C+56>, i1 = 170, c1 = 65 'A'}, i3 = 204, c3 = 67 'C'}
c_ptr = 0x7ffffffeddf0
b_ptr = 0x7ffffffeddf0
a_ptr = 0x7ffffffede00
data = 0x8201ca0 <vtable for C+16>
fun = 0x800123d <__libc_csu_init+77>
(gdb) x/5xg &c
0x7ffffffeddf0: 0x0000000008201ca0      0x00000042000000bb      // vtable for C+16       B::c2 B::i2
0x7ffffffede00: 0x0000000008201cc8      0x00000041000000aa      // vtable for C+56       A::c1 A::i1
0x7ffffffede10: 0x00007f43000000cc                              // C::c3 C::i3
(gdb) x/9xg data-2
0x8201c90 <_ZTV1C>:     0x0000000000000000      0x0000000008201d18  // 0                 & typeinfo for C
0x8201ca0 <_ZTV1C+16>:  0x0000000008001126      0x0000000008001174  // C::f              C::f2
0x8201cb0 <_ZTV1C+32>:  0x00000000080011ac      0xfffffffffffffff0  // C::f3             -16
0x8201cc0 <_ZTV1C+48>:  0x0000000008201d18      0x000000000800116e  // & typeinfo for C  C::non-virtual thunk to C::f()
0x8201cd0 <_ZTV1C+64>:  0x0000000008000ff6                          // A::f1
```

基本上可以看出，`c` 中有两个虚表指针 `vptr`, 分别"服务"于它的两个直接基类。向上转型到哪个类，其指针就指向基类实例中哪个 `vptr` 的位置。此例中的 `a_ptr` 就指向了对象`c`中第二个`vptr`的地址`0x7ffffffede00`。

之前分析了虚函数调用时的汇编代码，知道了调用虚函数是依靠`vptr`的偏移来定位的，也就是说基类中的虚函数和派生类中重写的虚函数应该是位于虚表中的同一位置。所以基类与派生类两者的虚函数表结构应该是可以”重叠“上的。

具体到当前这个例子，如果将基类和派生类的虚表这样放置：

```
C::vtable for C: 9 entries                                  // B 的虚表
0     (int (*)(...))0                                       0     (int (*)(...))0
8     (int (*)(...))(& typeinfo for C)                      8     (int (*)(...))(& typeinfo for B)
16    (int (*)(...))C::f                                    16    (int (*)(...))B::f
24    (int (*)(...))C::f2                                   24    (int (*)(...))B::f2
32    (int (*)(...))C::f3                                   // A 的虚表
40    (int (*)(...))-16                                     0     (int (*)(...))0
48    (int (*)(...))(& typeinfo for C)                      8     (int (*)(...))(& typeinfo for A)
56    (int (*)(...))C::non-virtual thunk to C::f()          16    (int (*)(...))A::f
64    (int (*)(...))A::f1                                   24    (int (*)(...))A::f1
```
可以看出，除了派生类中新增的虚函数被追加到属于第一个基类的虚函数表尾部，左右其他部分的表项都是一一对应的。派生类中重写的虚函数“顶替”了基类中原来的表项，没有重写的则保持了原来的值。

如果继承的层次比较多的话，虚表的结构也是由上至下一层一层“继承”过来的。

至此，在不涉及到虚继承的情况下，虚表的结构以及对象的结构都算是比较易得的。单继承时虚表结构“父子”相承，多继承时无非多几个 `vptr`。即便是多重继承，从上至下细细推导也不难得出所有的虚表结构。

顺带一提虚表中的`C::non-virtual thunk to C::f()`的作用是当使用基类指针访问子类中重写的虚函数时，将传入的`this`指针减去偏移后，得到对象的首地址（派生类的`this`）再调用该虚函数。不过这里的偏移是编译期就能确定的一个常数，而不是使用`vptr - 2`位置上的偏移值。

## 引入虚继承带来的变化
虚继承一般常见于钻石型继承结构中。
```c++
// 钻石型虚继承
class B
{
public:
    int ib;
    char cb;

public:
    B() : ib(0xBB), cb('A') {}
    virtual void f() { ++ib; cout << "B::f()" << endl; }
    virtual void Bf() { --ib; cout << "B::Bf()" << endl; }
};

class B1 : virtual public B
{
public:
    int ib1;
    char cb1;

public:
    B1() : ib1(0xB1), cb1('B') {}
    virtual void f() { ++ib; cout << "B1::f()" << endl; }
    virtual void f1() { --cb; cout << "B1::f1()" << endl; }
    virtual void Bf1() { cout << "B1::Bf1()" << endl; }
};

class B2 : virtual public B
{
public:
    int ib2;
    char cb2;

public:
    B2() : ib2(0xB2), cb2('C') {}
    virtual void f() { ++ib; cout << "B2::f()" << endl; }
    virtual void f2() { --ib; cout << "B2::f2()" << endl; }
    virtual void Bf2() { cout << "B2::Bf2()" << endl; }
};

class D : public B1, public B2
{
public:
    int id;
    char cd;

public:
    D() : id(0xDD), cd('D') {}
    virtual void f() { ++ib; cout << "D::f()" << endl; }
    virtual void f1() { --ib; cout << "D::f1()" << endl; }
    virtual void f2() { cout << "D::f2()" << endl; }
    virtual void Df() { cout << "D::Df()" << endl; }
};

int main()
{
    D d;
    B1 * b1_ptr = &d;
    B2 * b2_ptr = &d;
    B * b_ptr = &d;
    cout << "b_ptr : " << b_ptr << endl
        << "b1_ptr : " << b1_ptr << endl
        << "b2_ptr : " << b2_ptr << endl
        << "d_ptr : " << &d << endl;
    cout << typeid(*b1_ptr).name() << endl;
    B1 b1;
    b_ptr = &b1;
    b_ptr->Bf();
    b1_ptr = &b1;
    b1_ptr->Bf();
}
```
依旧是先看这些类的布局：

```
Vtable for B
B::vtable for B: 4 entries
0     (int (*)(...))0
8     (int (*)(...))(& typeinfo for B)
16    (int (*)(...))B::f
24    (int (*)(...))B::Bf

Class B
   size=16 align=8
   base size=13 base align=8
B (0x0x7ffad8f56720) 0
    vptr=((& B::vtable for B) + 16)

Vtable for B1                                       Construction vtable for B1 (0x0x7ffad8f8fd00 instance) in D
B1::vtable for B1: 12 entries                       D::construction vtable for B1-in-D: 12 entries
0     16                                            0     40
8     (int (*)(...))0                               8     (int (*)(...))0
16    (int (*)(...))(& typeinfo for B1)             16    (int (*)(...))(& typeinfo for B1)
24    (int (*)(...))B1::f                           24    (int (*)(...))B1::f
32    (int (*)(...))B1::f1                          32    (int (*)(...))B1::f1
40    (int (*)(...))B1::Bf1                         40    (int (*)(...))B1::Bf1
48    0                                             48    0
56    18446744073709551600                          56    18446744073709551576
64    (int (*)(...))-16                             64    (int (*)(...))-40
72    (int (*)(...))(& typeinfo for B1)             72    (int (*)(...))(& typeinfo for B1)
80    (int (*)(...))B1::virtual thunk to B1::f()    80    (int (*)(...))B1::virtual thunk to B1::f()
88    (int (*)(...))B::Bf                           88    (int (*)(...))B::Bf

VTT for B1
B1::VTT for B1: 2 entries
0     ((& B1::vtable for B1) + 24)
8     ((& B1::vtable for B1) + 80)

Class B1
   size=32 align=8
   base size=13 base align=8
B1 (0x0x7ffad8f8f958) 0
    vptridx=0 vptr=((& B1::vtable for B1) + 24)
  B (0x0x7ffad8f96660) 16 virtual
      vptridx=8 vbaseoffset=-24 vptr=((& B1::vtable for B1) + 80)

Vtable for B2                                       Construction vtable for B2 (0x0x7ffad8f8fd68 instance) in D
B2::vtable for B2: 12 entries                       D::construction vtable for B2-in-D: 12 entries
0     16                                            0     24
8     (int (*)(...))0                               8     (int (*)(...))0
16    (int (*)(...))(& typeinfo for B2)             16    (int (*)(...))(& typeinfo for B2)
24    (int (*)(...))B2::f                           24    (int (*)(...))B2::f
32    (int (*)(...))B2::f2                          32    (int (*)(...))B2::f2
40    (int (*)(...))B2::Bf2                         40    (int (*)(...))B2::Bf2
48    0                                             48    0
56    18446744073709551600                          56    18446744073709551592
64    (int (*)(...))-16                             64    (int (*)(...))-24
72    (int (*)(...))(& typeinfo for B2)             72    (int (*)(...))(& typeinfo for B2)
80    (int (*)(...))B2::virtual thunk to B2::f()    80    (int (*)(...))B2::virtual thunk to B2::f()
88    (int (*)(...))B::Bf                           88    (int (*)(...))B::Bf

VTT for B2
B2::VTT for B2: 2 entries
0     ((& B2::vtable for B2) + 24)
8     ((& B2::vtable for B2) + 80)

Class B2
   size=32 align=8
   base size=13 base align=8
B2 (0x0x7ffad8f8fc30) 0
    vptridx=0 vptr=((& B2::vtable for B2) + 24)
  B (0x0x7ffad8f96b40) 16 virtual
      vptridx=8 vbaseoffset=-24 vptr=((& B2::vtable for B2) + 80)

Vtable for D
D::vtable for D: 20 entries
0     40
8     (int (*)(...))0
16    (int (*)(...))(& typeinfo for D)
24    (int (*)(...))D::f
32    (int (*)(...))D::f1
40    (int (*)(...))B1::Bf1
48    (int (*)(...))D::f2
56    (int (*)(...))D::Df
64    24
72    (int (*)(...))-16
80    (int (*)(...))(& typeinfo for D)
88    (int (*)(...))D::non-virtual thunk to D::f()
96    (int (*)(...))D::non-virtual thunk to D::f2()
104   (int (*)(...))B2::Bf2
112   0
120   18446744073709551576      // -40
128   (int (*)(...))-40
136   (int (*)(...))(& typeinfo for D)
144   (int (*)(...))D::virtual thunk to D::f()
152   (int (*)(...))B::Bf

VTT for D
D::VTT for D: 7 entries
0     ((& D::vtable for D) + 24)
8     ((& D::construction vtable for B1-in-D) + 24)
16    ((& D::construction vtable for B1-in-D) + 80)
24    ((& D::construction vtable for B2-in-D) + 24)
32    ((& D::construction vtable for B2-in-D) + 80)
40    ((& D::vtable for D) + 144)
48    ((& D::vtable for D) + 88)

Class D
   size=56 align=8
   base size=37 base align=8
D (0x0x7ffad8fc0150) 0
    vptridx=0 vptr=((& D::vtable for D) + 24)
  B1 (0x0x7ffad8f8fd00) 0
      primary-for D (0x0x7ffad8fc0150)
      subvttidx=8
    B (0x0x7ffad8f96ea0) 40 virtual
        vptridx=40 vbaseoffset=-24 vptr=((& D::vtable for D) + 144)
  B2 (0x0x7ffad8f8fd68) 16
      subvttidx=24 vptridx=48 vptr=((& D::vtable for D) + 88)
    B (0x0x7ffad8f96ea0) alternative-path
```

### 虚单继承
暂时抛开其他部分，只关注`B->B1`这一条单继承&虚继承的结构。可以发现相比于普通的单继承，`B1`的虚表的每一段开头都新增了一项 `16`，而且继承类的虚表不再是重叠在基类的虚表上。那么虚继承为什么要给虚表新增一项？为什么本类的虚函数没有追加到基类的虚表尾部，而是独立一块？

首先我们知道虚继承的作用是使得在钻石型继承层次下（说普遍一点应该是共基类多继承），底层派生类中只存在一个虚基类子对象。如果子类直接使用`this`指针加上一个固定的偏移去访问虚基类中的成员的话，虚基类子对象的位置就会被固定下来，多个子类就一定会带上多个基类子对象。这与虚继承的功能是相违背的。

所以虚基类子对象的位置不能固定。通过`gdb`调试信息可知，虚基类子对象一般是放在子类对象的末尾。类`B`子对象到类`B1`子对象之间的距离确实不是固定的（类`D`还可以继承更多类）。执行一个指向`D`对象的`B1`指针调用的`B1`中没有被`D`重写的虚函数时（比如这里的`b1_ptr->Bf();`），需要动态地确定基类子对象的位置。

```c++

(gdb) x/7xg &d
0x7ffffffede20: 0x0000000008202b40      0x00000042000000b1      // vptr for B1 , {'B', 0xB1}
0x7ffffffede30: 0x0000000008202b80      0x00000043000000b2      // vptr for B2 , {'C', 0xB2}
0x7ffffffede40: 0x00000044000000dd      0x0000000008202bb8      // {'D', 0xDD} , vptr for B 
0x7ffffffede50: 0x00007f41000000bb                              // {'A', 0xBB}
...
(gdb) x/4xg &b1
0x7ffffffede00: 0x0000000008202c68      0x00000042000000b1      // vptr for B1 , {'B', 0xB1}
0x7ffffffede10: 0x0000000008202ca0      0x00000041000000bb      // vptr for B  , {'A', 0xBB}
```

因此需要在虚表中新增一项，用于指出对象中的虚基类子对象相对于当前`this`指针的偏移值。每次访问虚基类的成员时都需要先读取该值，再将`this`加上此偏移才能得到虚基类子对象的地址。

以[`B1::f1()`](https://godbolt.org/z/Ntzexw)中的 `--cb;` 这条语句为例：

```
...
mov     QWORD PTR [rbp-8], rdi
mov     rax, QWORD PTR [rbp-8]      // this -> rax
mov     rax, QWORD PTR [rax]        // 虚表地址(首虚函数项地址) -> rax
sub     rax, 24                     // 向上寻 3 个虚表项
mov     rax, QWORD PTR [rax]        // 取得到虚基类子对象的偏移值
mov     rdx, rax                    // rax -> rdx
mov     rax, QWORD PTR [rbp-8]      // this -> rax
add     rax, rdx                    // 虚基类子对象地址 = this + 偏移值
movzx   edx, BYTE PTR [rax+12]      // 子对象地址 + 12 即为 B::cb 的地址
sub     edx, 1                      // --cb;
mov     BYTE PTR [rax+12], dl       // 寄存器 -> 内存
...
```
这就是虚基类带来的运行时性能损失。以及虚函数表的作用之二，定位虚基类子对象的地址。

同时这也解释了为什么虚继承会使得继承类的虚表中，继承类自己使用的虚函数不再和基类使用的虚函数重叠在一起，不像普通继承里那样追加表项到基类虚表尾部。这是因为继承类访问任何虚基类的成员变量，都需要经过类似上述汇编代码所述的将`this`加上一个偏移值的过程；而这个偏移值相对`B1*`和`B*`是不相同的，`B1*`需要加上`16`，而`B*`只需偏移`0`；所以两者不能共处，继承类和虚基类不再共用同一段虚表。在继承类的虚表的中间还有一项为`0`，我猜就是用来分隔两段的。

而这样的分离又导致了其他后果，且看下图：
![](/assets/img/c++_assemble_2.bmp)

一个指向`B1`对象的`B1*`类型的指针，调用继承而来的虚函数，所需要的步骤比它基类的指针调用这个函数多出这么多。来看看它多干了哪些活：
```asm
mov     rax, QWORD PTR [rbp-8]  \
mov     rax, QWORD PTR [rax]     |
sub     rax, 24                  |
mov     rax, QWORD PTR [rax]      } 虚基类子对象地址 -> rdx
mov     rdx, rax                 |
mov     rax, QWORD PTR [rbp-8]   |
add     rdx, rax                /
mov     rax, QWORD PTR [rbp-8]  \
mov     rax, QWORD PTR [rax]     |
sub     rax, 24                  |
mov     rax, QWORD PTR [rax]     |
mov     rcx, rax                 |
mov     rax, QWORD PTR [rbp-8]    } 虚基类的第二个虚函数地址 -> rax
add     rax, rcx                 |
mov     rax, QWORD PTR [rax]     |
add     rax, 8                   |
mov     rax, QWORD PTR [rax]    /
mov     rdi, rdx
call    rax
```
可以看出有虚继承时，没有重写的虚函数需要先将`B1 *this`转换成`B *this`才能调用。因为`B1`的虚表（前一段）里缺少它没有重写的虚函数，想要调用只能是`B*`来调用。可以说是虚函数调用中最繁琐的一类调用方式了。（这里居然重复计算了虚基类子对象的地址😳。不过这只是`g++`的一家之言，理解就好。）

### 钻石虚继承
在没有引入虚继承前，继承体系中的每个类在构造时只需要调用其直接基类的构造函数即可。基类自然会递归地调用它的基类的构造函数。

而引入虚继承后，虚基类的构造交给了继承体系中最底层的那个类。所以中间的那些类（或者说每一个含有虚基类的类）必须知道什么时候不去构造自己的虚基类。

具体到这个例子，`new`一个类`D`的对象时，`B`的构造函数应该由`D`的构造函数调用。中间的`B1`和`B2`不能重复构造类`B`。而`new`一个类`B1`时，它又需要自己去构造类`B`了。为了处理两种不同的构造需求——只构造自己和同时构造虚基类与自己——`g++`为含有虚基类的类生成了两个不同的构造函数，`base object constructor`和`complete object constructor`。前者专由该类的派生类调用。

虚继承的新增的两个结构 `VTT` 和 `Construction vtable` 正是为了处理虚基类的构造过程而引入的。

`VTT`的意思是`virtual-table table`，用来索引虚表的表，其表项指向了各个虚表的不同位置。

`Construction vtable for xx-in-yy` 顾名思义，就是专门给类`yy`在构造其基类`xx`时所使用的虚表。

先看看在类`D`的`complete object constructor`里这两个结构如何使用：

```asm
mov     QWORD PTR [rbp-8], rdi
mov     rax, QWORD PTR [rbp-8]
add     rax, 40                                 // 虚基类子对象地址
mov     rdi, rax
call    B::B() [base object constructor]        // 构造类 B
mov     rax, QWORD PTR [rbp-8]
mov     edx, OFFSET FLAT:VTT for D+8            // VTT for D 的第二项，是地址而不是内容（构造虚表）
mov     rsi, rdx
mov     rdi, rax
call    B1::B1() [base object constructor]      // 构造类 B1, 使用的是 base object constructor
mov     rax, QWORD PTR [rbp-8]
add     rax, 16
mov     edx, OFFSET FLAT:VTT for D+24           // VTT for D 的第四项
mov     rsi, rdx
mov     rdi, rax
call    B2::B2() [base object constructor]      // 构造类 B2
mov     edx, OFFSET FLAT:vtable for D+24        
mov     rax, QWORD PTR [rbp-8]
mov     QWORD PTR [rax], rdx                    // 类 D 中的 vptr for B1
mov     rax, QWORD PTR [rbp-8]
add     rax, 40
mov     edx, OFFSET FLAT:vtable for D+144       
mov     QWORD PTR [rax], rdx
mov     edx, OFFSET FLAT:vtable for D+88        // 类 D 中的 vptr for B
mov     rax, QWORD PTR [rbp-8]
mov     QWORD PTR [rax+16], rdx                 // 类 D 中的 vptr for B2
mov     rax, QWORD PTR [rbp-8]
mov     DWORD PTR [rax+32], 221                 // 0xDD
mov     rax, QWORD PTR [rbp-8]
mov     BYTE PTR [rax+36], 68                   // 'D'
```
可以看到类 `D` 在构造 `B1` 和 `B2` 的时候不仅使用的是 `base object constructor`，而且还将`construction vtable`的索引作为参数传入。那么这个构造虚表与普通虚表又有何不同呢？

为了便于比较，我把`Construction vtable for B1-in-D` 放到了 `Vtable for B1` 的右边，可以看到构造虚表中除了虚基类的偏移不同（主要作用）之外，其他表项全都相同，这意味着`B1`的构造函数中只能使用自己的虚函数。这也是《Effective C++》中条款9：“绝不在构造和析构函数中调用 virtual 函数”的原因，因为这个时候没有多态，虚函数的调用都是固定的。不可能也不可以在基类的构造函数中调用到派生类的虚函数——派生类的成员都还没有构造好。实际上条款 9 针对是所有继承方式而不只是虚继承，只是与构造虚表结合则更现突出。

至于`VTT`的作用，在这个继承体系中的作用其实还不太明显，完全可以直接传递构造虚表的地址给`base object constructor`。`VTT`服务于更加多层的继承体系，假如又有一个类`E`继承了类`D`，那么在类`D`的`base object constructor`中，需要类`E`提供的`Construction vtable for D-in-E` 以及 `Construction vtable for B1-in-E`，这个时候就不能只传入一个构造虚表的地址，而是传入`VTT`中的相应表项，就可以让`D`的`base object constructor`自己加上偏移找到`Construction vtable for B1-in-E`和`Construction vtable for B2-in-E`了。

## 总结
通过分析编译期生成的各种结构以及各类虚函数的调用过程，可以较为清楚地了解到 C++ 底层对于多态性的支持。而 C++ 多态性的最主要支撑点就是虚函数表，它解决了类型识别，函数延迟绑定，共有基类子对象查找等问题。

本文仅分析了虚函数调用以及构造函数调用的情况，尚有`RTTI`及异常处理等多态性的语义未提及。本文的分析对象仅限于`gcc version 7.5.0 (Ubuntu 7.5.0-3ubuntu1~18.04)`所生成的数据，其他编译器生成的代码则多有不同，比如`MSVC`中就是直接将虚基类子对象的偏移插入到了对象当中。这些都有待继续探索。

## 参考资料
[C++ 对象的内存布局 | | 酷 壳 - CoolShell](https://coolshell.cn/articles/12176.html)  
[C++ vtables - Part 1 - Basics | Shahar Mike's Web Spot](https://shaharmike.com/cpp/vtable-part1/)  
[C++多态及其实现原理](https://www.cnblogs.com/BEN-LK/p/10720300.html)  
[图说C++对象模型：对象内存布局详解](https://www.cnblogs.com/QG-whz/p/4909359.html)  
[从内存布局看C++虚继承的实现原理_C/C++_上善若水，人淡如菊-CSDN博客](https://blog.csdn.net/xiejingfa/article/details/48028491)  
[What is the virtual table table?](https://quuxplusone.github.io/blog/2019/09/30/what-is-the-vtt/)