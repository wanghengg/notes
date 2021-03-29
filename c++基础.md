# 基础知识

## <font color=yellow>inline关键字</font>

`inline`要起作用的话应该和函数的定义放在一起，`inline`是一种用于实现的关键字，而不是用于函数的声明。

虚函数可以是内联函数，但是当虚函数表现为多态性的时候不能内联。**内联是在编译时建议编译器内联，而多态函数的多态性在运行期，编译器在编译时无法知道运行期间该调用那个代码，所以当虚函数表现为多态性时不可以内联。**

## <font color=yellow>sizeof关键字</font>

* 空类的大小为1
* 一个类中，虚函数本身、成员函数（包括静态和非静态）和静态数据成员都不占用类对象的存储空间。
* 对于包含虚函数的类，不管有多少个虚函数，只有一个虚指针`vptr`的大小（对于32为操作系统为4字节，对64为操作系统为8字节）
* 普通继承，派生类继承了所有基类的函数与成员，要按照字节对齐来计算大小。
* 虚函数继承，不管是单继承还是多继承，都是继承了基类的`vptr`

## <font color=yellow>vtable和vptr</font>

### 虚表

1. 为了实现虚函数，`c++`使用虚表（`vtable`）进行后期绑定。该虚拟表是在用在解决动态/后期绑定方式的函数调用的查找表。
2. 每个使用虚函数的类都有自己的虚表，该表是编译器在编译时设置的静态数组。**虚表是一个指针数组，其元素是虚函数的指针，每个元素对应一个虚函数的函数指针。普通函数的调用不需要经过虚表，所以虚表的元素不包含普通函数的函数指针。**
3. 虚表的条目，即虚函数指针的赋值发生在编译器的编译阶段，也就是说在代码的编译阶段，虚表就构造出来了。

### 虚表指针

1. 虚表是属于类的，而不是属于具体的某个对象。一个类只有一个虚表，一个类的所有对象都使用同一个虚表。
2. 为了指定对象使用的虚表，对象内部包含一个指向该类虚表的指针，被称为虚表指针(`vptr`)。当类的对象在构造时就构建了一个指向虚表的指针。

### 静态函数可以是虚函数吗？

静态函数不能是虚函数。static成员函数不属于任何类或者实例，但是虚函数要通过`vptr`才能访问，而`vptr`是对象或者实例所有的，这与静态函数不属于任何类或者实例矛盾。

### 构造函数可以是虚函数吗？

构造函数不能声明为虚函数。尽管虚函数表`vtable`是编译时就已经建立的，但是虚表指针`vptr`是运行过程中对象实例化时创建的，如果构造函数时虚函数，那么访问虚函数需要使用虚表指针，但是此时对象还没有实例化，`vptr`还没有创建，所以自然无法访问虚函数表，因此构造函数不能为虚函数。

### 析构函数可以是虚函数吗？

析构函数可以声明为虚函数。如果我们要删除一个指向派生类的基类指针，应该把析构函数声明为虚函数。这样在执行过程中才能根据指针指向的对象的类型调用动态绑定的析构函数，否则只能使用基类的析构函数。

### 虚函数中默认参数问题

```c++
#include <iostream>
using namespace std;

class Base{
public:
    virtual void fun(int x = 10) {
        cout << "Base::fun(), x = " << x << endl;
    }
};

class Derived : public Base {
public:
    virtual void fun(int x = 20) {
        cout << "Derived::fun(), x = " << x << endl;
    }
};

int main() {
    Derived d1;
    Base *bp = &d1;
    bp->fun();
    return 0;
}
```

**<font color=red>默认参数时静态绑定的，虚函数是动态绑定的。默认参数的使用需要看指针或者引用本身的类型，而不是对象的类型。</font>**

所以上面代码的运行结果为：

```
Derived::fun(), x = 10
```

### 虚函数可以被内联吗？

**通过基类指针或者引用调用的虚函数必定不可能被内联。但是实体对象调用虚函数或者静态调用时可以内联**

* 虚函数可以是内联函数，内联是可以修饰析构函数的。但是当虚函数表现为多态性的时候不能内联。
* 内联是在编译时编译器建议内联，而虚函数的动态绑定在运行时，编译器无法知道运行期调用哪段代码，所以无法对表现出多态性的虚函数进行内联。

## <font color=yellow>volatile关键字</font>

* 使用volatile关键字通知编译器不要对被声明的变量优化，每次对该变量进行读写操作时都直接从内存中读取。
* 多线程应用中被多个任务共享的变量。当多个线程共享一个变量时，通过对此变量声明`volatile`，防止运行时将此变量放在寄存器而未写入内存，切换线程后，其他线程不能及时同步到变量的修改。所以`volatile`的意思是每次访问变量时直接从内存中取。

## <font color=yellow>union关键字</font>

union是一种节省空间的特殊的类，一个union可以有多个数据成员，但是在任意时刻只有一个数据成员可以有值。当某个成员被赋值时其他成员变为未定义状态。union对象有如下特点：

* 默认访问控制符为public
* 可以含有构造函数和析构函数
* 不能含有引用其他类型的成员
* 不能继承自其他类，不能作为基类
* 不能含有虚函数

## <font color=yellow>explicit关键字</font>

`explicit`关键字的作用是使类的构造函数显示调用，避免隐式类型转换。示例：

```c++
#include <iostream>
using namespace std;

class A
{
public:
    explicit A(int i = 1) : m(i)
    {}

    int getMa()
    {
        return m;
    }
private:
    int m;
};

int main()
{
    A a;
    cout << "a.m before a=10: " << a.getMa() << endl;
    a = 2;	// 隐式类型转换
    cout << "a.m after  a=10: " << a.getMa() << endl;

    return 0;
}
```

**把构造函数之前添加`explicit`关键字，禁止隐式转换**

## <font color=yellow>友元</font>

* 友元函数：普通函数访问一个类的私有或保护成员
* 友元类：类A的成员函数访问类B的私有或保护成员。

### 友元函数

* 在类的任何地方声明，定义则在类的外部。

```c++
#include <iostream>

using namespace std;

class A
{
public:
    A(int _a):a(_a){};
    friend int geta(A &ca);  ///< 友元函数
private:
    int a;
};

int geta(A &ca) 
{
    return ca.a;
}

int main()
{
    A a(3);    
    cout<<geta(a)<<endl;

    return 0;
}
```

### 友元类

友元类的声明在该类的声明中，实现在类外。

```c++
#include <iostream>

using namespace std;

class A
{
public:
    A(int _a):a(_a){};
    friend class B;
private:
    int a;
};

class B
{
public:
    int getb(A ca) {
        return  ca.a; 
    };
};

int main() 
{
    A a(3);
    B b;
    cout<<b.getb(a)<<endl;
    return 0;
}
```

**<font color=red>注意</font>**

* **友元关系没有继承性，假如类B是类A的友元，类C继承与类A，那么友元B不能直接访问类C的私有或保护成员。**
* **友元关系没有传递性，假如类类B是类A的友元，类C是类B的友元，那么友元类C不能直接访问类A的私有或保护成员。**

## <font color=yellow>C++派生类对基类的三种访问规则</font>

### 私有继承的访问

**当类的继承方式为私有继承时，基类的public成员和protected成员被继承后成为派生类的private成员，派生类的其它成员可以直接访问它们，但是在类的外部通过派生类的对象无法访问。** **<font color=red>基类的private成员在私有派生类中是不可直接访问的，所以无论是派生类的成员还是通过派生类的对象，都无法直接访问从基类继承来的private成员，但是可以通过基类提供的public成员函数间接访问。</font>**私有继承的访问规则总结如下：

| 基类成员 | private成员 | public成员 | protected成员 |
| -------- | ----------- | ---------- | ------------- |
| 内部访问 | 不可访问    | 可访问     | 可访问        |
| 对象访问 | 不可访问    | 不可访问   | 不可访问      |

### 公有继承的访问规则

当类的继承方式为公有继承时，基类的public成员和protected成员被继承到派生类中仍作为派生类的public成员和protected成员，派生类的其它成员可以直接访问它们。但是，类的外部使用者只能通过派生类的对象访问继承来的public成员。基类的private成员在私有派生类中是不可直接访问的，所以无论是派生类成员还是派生类的对象，都无法直接访问从基类继承来的private成员，但是可以通过基类提供的public成员函数直接访问它们。公有继承的访问规则总结如下：

| 基类成员 | private成员 | public成员 | protected成员 |
| -------- | ----------- | ---------- | ------------- |
| 内部访问 | 不可访问    | 可访问     | 可访问        |
| 对象访问 | 不可访问    | 可访问     | 不可访问      |

### 保护继承的访问规则

 当类的继承方式为保护继承时，基类的public成员和protected成员被继承到派生类中都作为派生类的protected成员，派生类的其它成员可以直接访问它们，但是类的外部使用者不能通过派生类的对象访问它们。基类的private成员在私有派生类中是不可直接访问的，所以无论是派生类成员还是通过派生类的对象，都无法直接访问基类中的private成员。保护继承的访问规则总结如下：

| 基类成员 | private成员 | public成员 | protected成员 |
| -------- | ----------- | ---------- | ------------- |
| 内部访问 | 不可访问    | 可访问     | 可访问        |
| 对象访问 | 不可访问    | 不可访问   | 不可访问      |

## <font color=yellow>哈希冲突以及哈希冲突的解决方法</font>

## <font color=yellow>构造函数和默认构造函数</font>

**<font color=red>默认构造函数在被需要的时候被编译器产生出来。</font>**

编译器为程序构建默认构造函数是因为编译器需要它，而不是因为程序需要它。如果一个类没有显示定义任何构造函数，或者定义的构造函数没有任何参数，或者定义的构造函数的所有参数都有默认值，此时创建类对象的时候不需要给出成员变量的值，编译器会使用默认构造函数创建类对象。

```c++
#include <iostream>
using namespace std;

class Foo{
public:
    // Foo() {cout << "default constructor" << endl;}	// 默认构造函数
    Foo(int v = 1) : val(v), next(nullptr) {cout << "default constructor" << endl;}	// 默认构造函数
    int val;
    Foo *next;
};

void fun() {
    Foo foo;
    if (foo.val || foo.next) {
        cout << "hello" << endl;
    }
}

int main() {
    fun();
    return 0;
}
```

* 默认构造函数是被编译器需要的，那么编译器什么时候需要它呢？在下列四种情况下编译器会生成默认构造函数：
  1. `class` 内包含有 `default constructor` 的 `member object`，合成 `default constructor` 为了调用 `member object `的` default constructor`。
  2. `class` 继承自含有 `default constructor `的基类，合成 `default constructor `为了调用基类的 `default constructor `
  3. 带有虚函数的 `class` ，合成 `default constructor `主要为了初始化`vptr`（虚函数表指针）。
  4. `class `有一个及以上的虚基类，合成 `default constructor `主要为了初始化虚基类指针。
* 当满足上面的条件时，编译器会对`contructor`进行拓展，拓展规则如下：
  1. 当 `class` 没有定义 `constructor` ，编译器会合成 `default constructor` ，并加入编译器需要的操作，可能包括调用 `member object` 的 `default constructor` ，调用基类的 `default constructor` ，初始化虚函数表指针及虚基类指针。
  2. 当 `class` 已经定义了一个或多个 `constructor` 时，编译器不会再去合成 `default constructor` ，但会扩展所有 `constructor`加入编译器需要的操作。

当一个类已经定义了构造函数时，编译器不会合成默认构造函数：

```c++
#include <iostream>
using namespace std;

class Foo{
public:
    // Foo() {cout << "default constructor" << endl;}
    Foo(int v) : val(v), next(nullptr) {cout << "constructor" << endl;}
    int val;
    Foo *next;
};

void fun() {
    Foo foo;
    if (foo.val || foo.next) {
        cout << "hello" << endl;
    }
}

int main() {
    fun();
    return 0;
}
```

上面代码编译报错`no matching function for call to 'Foo::Foo()'`，因为已经有了自己定义的构造函数，所有不会合成默认构造函数。

## <font color=yellow>左值和右值</font>

* 左值是有标识符，可以取地址的表达式，可以在作用域里长期存在。

* 纯右值是没有标识符，不可以取地址的表达式，一般也称为临时对象。**一个临时对象会在包含这个临时对象的完整表达式估值结束完成后、按生成顺序的逆序被销毁，除非有生命周期延长发生。**

  ```c++
  #include <stdio.h>
  
  class shape {
  public:
    virtual ~shape() {}
  };
  
  class circle : public shape {
  public:
    circle() { puts("circle()"); }
    ~circle() { puts("~circle()"); }
  };
  
  class triangle : public shape {
  public:
    triangle() { puts("triangle()"); }
    ~triangle() { puts("~triangle()"); }
  };
  
  class result {
  public:
    result() { puts("result()"); }
    ~result() { puts("~result()"); }
  };
  
  result
  process_shape(const shape& shape1,
                const shape& shape2)
  {
    puts("process_shape()");
    return result();
  }
  
  int main()
  {
    puts("main()");
    process_shape(circle(), triangle());
    puts("something else");
  }
  ```

  对于调用`process_shape(circle(), triangle());`来说，运行结果如下：

  ```
  main()
  circle()
  triangle()
  process_shape()
  result()
  ~result()
  ~triangle()
  ~circle()
  something else
  ```

  调用函数，首先传递参数时构造临时对象`circle()`和`triangle()`，所以会调用它们的构造函数，然后打印`process_shape()`，返回时，构造了临时对象`result`，所以会调用`result`的构造函数和析构函数，然后`circle`对象和`triangle`对象生命周期结束需要被销毁，调用它们的析构函数，最后打印`something else`。

## STL空间配置器allocator是怎么自动分配内存的



## 智能指针中的RAII

**Resource Acquisition Is Initialization（资源获取就是初始化）是C++语言的一种管理资源、避免内存泄漏的方法。**利用的是C++构造的对象最终会被销毁的原则。**RAII的做法就是使用一个对象来管理需要动态分配的内存，在其构造时获取对应的资源，在对象生命期内控制对资源的访问，使之始终保持有效，在最后析构的时候释放构造时获取的资源。**在构造时获取资源，在析构时释放资源，避免了手动分配释放资源的麻烦，更重要的是即使程序在运行过程中异常终止，也能通过对象的析构函数自动释放分配的内存（如果用new、delete手动分配内存，那么由于异常会导致后面的内存释放语句没有被调用，导致了内存泄漏），避免了内存泄漏。RAII的常见应用就是智能指针。

## new和malloc的区别

1. malloc返回的是void指针，没有类型，需要通过强制类型转换将其转换为所需要的类型指针，new返回的是某种数据类型的指针
2. new是操作符，可以重载，需要编译器支持，只能在c++中使用，malloc是函数，需要库函数支持
3. new可以调用类的构造函数，malloc仅仅分配内存，不执行构造函数，只能用于内部类型的动态内存分配，不能用于动态对象的内存分配
4. new操作符申请内存时无需指定内存块的大小，编译器会根据对象类型自动分配内存大小，而malloc需要显式地指出要分配内存的大小
5. new内存分配失败时，会抛出bad_alloc异常，malloc分配内存失败会返回NULL

## 内存溢出是什么？什么时候会产生内存溢出？

内存溢出是指程序在申请内存时没有足够的内存空间可以使用，原因可能如下：

* 需要往内存中加载的数据过大
* 存在死循环
* 递归调用太深，导致堆栈溢出
* 大量内存泄漏最终导致内存溢出

### 补充：内存溢出和内存泄漏

内存泄漏是指动态分配的内存没有释放，导致占用了有效内存。内存溢出指申请内存时内存不足。

## C++中的四种cast转换

c++中的四种cast转换是：`static_cast, dynamic_cast,const_cast,reinterpret_cast`

顶层const：表示指针本身是一个常量

底层const：表示指针或引用所指的对象是一个常量

1. const_cast

    将const变量转为非const，只能改变对象的底层const

    <font color=red>顶层const和底层const</font>

    * 顶层const指对象本身时const类型，底层const指的是对象的指针或者引用为const，引用只能是底层const修饰，指针既可以是顶层const又可以是底层const。

2. static_cast

    static_cast和C语言风格的强制类型转换基本一样，由于没有运行时类型检查来保证转换的安全性，所以这类强制类型转换有安全隐患。**用于非多态类型的转换。**

    * 用于类层次结构中基类和派生类之间指针或引用的转换。

        进行向上转换（把派生类的指针或者引用转换为基类表示）是安全的。

        进行向下转换（把基类的指针或者引用转换为派生类表示）时，由于没有运行时类型检查，所以时不安全的。

    * 用于基本数据类型之间的转换，例如将int转换为double，int转换为enum

    * 把任何类型的指针转为void指针

3. dynamic_cast

    **用于多态类型的转换。**

    **转换的目标类型必须是类的指针或引用。用于将基类的指针或引用安全的转换为派生类的指针或引用。**

    * dynamic_cast在运行期强制类型转换，运行时要进行类型检查，较安全
    * 不能用于内置的基本数据类型的强制转换

    涉及到类，**使用dynamic_cast进行转换的，基类中一定要有虚函数**，否则编译不通过。

    对指针进行dynamic_cast，失败返回NULL，成功返回cast之后的对象的指针

    对引用进行dynamic_cast，失败抛出一个bad_cast，成功返回正常cast之后的对象引用

    对于向上转换（将派生类的指针或者引用转换为其基类类型）都是安全的。

    对于向下转换有两种情况：

    * 第一类，基类指针所指对象是派生类类型，这种转换是安全的
    * 第二类，基类指针所指对象时基类类型，dynamic_cast在运行时做检查，转换失败，返回结果NULL。

    ```c++
    class Base{
    public:
        virtual dummy(){}
    };
    class Derived:public Base{};
    
    Base* b1 = new Derived;
    Base* b2 = new Base;
    
    Derived* d1 = dynamic_cast<Derived>(b1);//成功
    Deriver* d2 = dynamic_cast<Derived>(b2);//失败，返回NULL
    ```

4. reinterpret_cast

    在指针之间转换，将一个类型的指针转换为另一个类型的指针。<font color=red>顾名思义，就是把内存里面的值重新解释，例如，在32位系统中，可以把指针解释为一个int整数</font>

## 重载和重写的区别

### 重载

指同一作用域内被声明的几个具有不同参数列表（参数的类型、个数、顺序不同）的同名函数，根据参数列表确认调用哪个函数，重载不关心函数的返回类型。

```c++
#include<bits/stdc++.h>

using namespace std;

class A
{
	void fun() {};
	void fun(int i) {};
	void fun(int i, int j) {};
    int fun() {};	// 只有返回值不一样不是重载，编译器会报错
};

```

### 重写（override）

是指派生类中存在重新定义的函数。**其函数名，参数列表，返回值类型，所有都必须同基类中被重写的函数一致。**只有函数体不同（花括号内），派生类调用时会调用派生类的重写函数，不会调用被重写函数。**重写的基类中被重写的函数必须有virtual修饰。**

```c++
#include<bits/stdc++.h>

using namespace std;

class A
{
public:
	virtual	void fun()
	{
		cout << "A";
	}
};
class B :public A
{
public:
	virtual void fun()	// 重写了基类的fun()
	{
		cout << "B";
	}
};
int main(void)
{
	A* a = new B();
	a->fun();//输出B
}

```

### 重写和重载的区别

1. 范围区别：重写和被重写的函数在不同的类中，重载和被重载的函数在同一作用域中。
2. 参数区别：重写与被重写的函数参数列表一定相同，重载和被重载的函数参数列表一定不同。
3. virtual的区别：重写的基类函数必须要有virtual修饰，重载函数和被重载函数可以被virtual修饰，也可以没有。

## 函数对象

> 函数对象：定义了操作符()的对象。该对象可以像函数一样调用，因此取名函数对象。<font color=red>可用于替换容器的默认排序方式，或者当做algorithm函数的谓词函数。

```c++
class A 
{  
public:  
    int operator() ( int val )  
    {  
        return val > 0 ? val : -val;
    }  
};  
```

## extern关键字

1. **引用同一个文件中声明在后面的变量**

   ```c++
   #include<stdio.h>
    
   int func();
    
   int main()
   {
       func(); //1
       printf("%d",num); //2
       return 0;
   }
   
   int num = 3;
   
   int func()
   {
       printf("%d\n",num);
   }
   ```

2. **引用另一个文件中的变量或者函数**

   `main.cpp`

   ```c++
   #include <iostream>
   using namespace std;
   
   int main() {
       extern int add(int, int);
       extern void func();
       extern int num;
       cout << add(4, 5) << endl;
       cout << num << endl;
       func();
       return 0;
   }
   
   int num = 6;
   
   ```

   `add.cpp`

   ```c++
   #include <iostream>
   using namespace std;
   int add(int a, int b) {
       return a + b;
   }
   
   void func() {
       cout << "hello" << endl;
   }
   ```



# 现代C++实战30讲

## <font color=yellow>堆、栈、RAII：C++如何管理资源</font>

* 不管是否发生异常，类的析构函数都会得到执行
* 当对象很大，或者对象的大小在编译时不能确定，这些情况下对象不能存储在堆上。

系统资源有限，使用系统资源时必须遵循一个步骤：1）申请资源；2）使用资源；3）释放资源

```c++
#include <iostream> 

using namespace std; 

int main() 

{ 
    int *testArray = new int [10]; 
    // Here, you can use the array 
    delete [] testArray; 
    testArray = NULL ; 
    return 0; 
}
```

如果程序很复杂的时候，需要为所有的new分配的内存delete，导致效率低下，代码臃肿。**更重要的是某一个操作发生了异常而导致释放资源的语句没有被调用，就会导致内存泄露，这是就可以使用RAII（Resource Acquisition Is Initialization）机制**

由于系统的资源不具有自动释放的功能，而C++中的类具有自动调用析构函数的功能。**如果把资源用类进行封装起来，对资源操作都封装在类的内部，在析构函数中进行释放资源。当定义的局部变量的声明结束时，它的析构函数就会自动的被调用。因此，不需要程序员显式地调用释放资源的操作。**

## <font color=yellow>自己动手，实现C++的智能指针</font>

根据C++规则，如果提供了移动构造函数而没有提供拷贝构造函数，那么后者将被自动禁用。

```c++
smart_ptr(smart_ptr& other);	// 拷贝构造函数的声明
smart_prt(smart_ptr&& other);	// 移动构造函数的声明
```

# 并发编程

