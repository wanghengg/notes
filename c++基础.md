# C++那些事儿

## inline关键字

`inline`要起作用的话应该和函数的定义放在一起，`inline`是一种用于实现的关键字，而不是用于函数的声明。

虚函数可以是内联函数，但是当虚函数表现为多态性的时候不能内联。**内联是在编译时建议编译器内联，而多态函数的多态性在运行期，编译器在编译时无法知道运行期间该调用那个代码，所以当虚函数表现为多态性时不可以内联。**

## sizeof关键字

* 空类的大小为1
* 一个类中，虚函数本身、成员函数（包括静态和非静态）和静态数据成员都不占用类对象的存储空间。
* 对于包含虚函数的类，不管有多少个虚函数，只有一个虚指针`vptr`的大小（对于32为操作系统为4字节，对64为操作系统为8字节）
* 普通继承，派生类继承了所有基类的函数与成员，要按照字节对齐来计算大小。
* 虚函数继承，不管是单继承还是多继承，都是继承了基类的`vptr`

## vtable和vptr

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

# 现代C++实战30讲

## 堆、栈、RAII：C++如何管理资源

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

## 自己动手，实现C++的智能指针

根据C++规则，如果提供了移动构造函数而没有提供拷贝构造函数，那么后者将被自动禁用。

```c++
smart_ptr(smart_ptr& other);	// 拷贝构造函数的声明
smart_prt(smart_ptr&& other);	// 移动构造函数的声明
```

# 基础知识

## 左值和右值

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



