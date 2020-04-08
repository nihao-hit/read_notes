# 深度探索C++对象模型（第三版）

[TOC]

## 第1章 关于对象（Object Lessons）

1. C++对象模型

   + 在C++中，有两种类数据成员：static和nonstatic，以及三种类成员函数：static、nonstatic和virtual。对下面的class Point声明：

   + ```c++
     :class Point {
     :public:
         Point(float xval);
         virtual ~Point();
         
         float x() const;
         static int PointCount();
     protected:
         virtual ostream&amp; print(ostream&amp; os) const;
         float _x;
         static int _point_count;
     }
     ```

   + C++对象模型如下：
   
     + nonstatic数据成员放置于**每个对象**中，static数据成员放置于**所有对象**外。
     + static、nonstatic成员函数都放置于**所有对象**外。
     + virtual成员函数则通过两个机制支持：
       + **每一个类**拥有一个存放着指向所有虚函数的指针的表（vtbl），vtbl的第一个slot存放着**每一个类**所关联的type_info对象，用以支持RTTI。
       + **每一个对象**被编译器添加了一个指针（vptr），指向vtbl。vptr的初始化与重置都由**每一个类**的构造函数、析构函数和拷贝赋值运算符自动完成。
   
   + ![图](./images/C++对象模型.png)
   
   + C++中凡处于同一个access section的数据，必定保证以其声明次序出现在内存布局中；然而被放置在多个access sections的数据，排列次序就不一定了。例如public与protected出现在内存布局的次序并无规定。
   
2. C++对象模型如何影响程序？

   + C++编译器对源代码做了如下修改：

     + RVO返回值优化
     + 将new关键字扩展为new()与对象构造函数调用。
     + 将delete关键字扩展为对象析构函数调用与delete()。
     + 使用虚机制扩展多态函数调用。

   + ```cpp
     //class X定义了拷贝构造、虚析构以及虚函数foo()
     //源代码							  //编译器修改后的代码
     X foobar(){							void foobar(X& _result){
         X xx;								//构造_result用来取代局部变量xx
         									_result.X::X();
         X* px = new X;						//扩展X* px = new X;
         									px = _new(sizeof(X));
         									if(px != 0)
                                                 px->X::X();
         //foo()是一个虚函数					//扩展xx.foo()但不使用虚机制
         xx.foo();							foo(&_result);
         px->foo();							//使用虚机制扩展px->foo();
         									(*px->vtbl[2])(px);	
         delete px;							//扩展delete px;
         									if(px != 0){
                                                 (*px->vtbl[1])(px);
                                                 _delete(px);
                                             }
         return xx;							//RVO;
         									return;
     }
     ```

3. 需要多少内存才能够表现一个C++对象？一般而言要有：
   + 其nonstatic数据成员的总和大小。
   + 加上任何由于align的需求而padding（填补）上去的空间。
   + 加上为了支持virtual而由内部产生的任何额外负担。（vtbl，还有什么？）
4. 继承体系中vptr如何放置？
   + 如图所示：单一继承下（多重继承和虚拟继承情况不同），派生对象Bear()与基对象ZooAnimal()共享基对象的vptr，这份vptr的内容在指针或引用发生类型转换时不需要修改，但是编译器如何为一个继承体系下的基对象与派生对象分别初始化vptr？
   + ![图](./images/继承体系内存布局.png)
   + 指向不同类型之指针的差异，既不在其指针表示法不同，也不在其内容（代表一个地址）不同，而是在其所寻址出来的对象类型不同，指针的类型会教导编译器如何解释某个特定地址的内存内容与大小（对象切割slice）。

## 第2章 构造函数语义学（The Semantics of Constructor）

1. 编译器什么时候会为类声明、定义**默认构造函数**？

   + **声明（declare）**：如果用户程序没有声明任何**构造函数**，那么编译器会声明一个implicite**默认构造函数**，这样的声明是trivial（无用的）。

   + **定义（define）**：只有当编译器需要声明的implicite默认构造函数执行某些编译器所需的行动，这时编译器会在需要调用构造函数的地方开始**定义**默认构造函数。这样的定义是nontrivial（有用的）。

   + 注意：**定义**只发生在编译器需要它的时候，而不是用户程序需要的时候，而且被合成出来的默认构造函数只执行编译器所需的行动。

     + 对下面的类，用户程序可能需要一个默认构造函数来将pnext指针初始化，但是这不是编译器的责任，因此编译器在此处并不会**定义**默认构造函数；就算编译器**定义**了默认构造函数，也不会执行pnext的初始化。

     + ```c++
       class Foo {
       public:
           int val;
           Foo* pnext;
       }
       ```

2. 什么情况下编译器需要对象的**默认构造函数**完成某些必需的操作？

   1. 编译器需要在对象的**默认构造函数**中插入代码，完成对成员对象、继承对象的**默认构造函数**的调用。

      + 对象含有成员对象，且成员对象有定义**默认构造函数**；
      + 对象继承自基对象，基对象有定义**默认构造函数**；

   2. 编译器需要在对象的**默认构造函数**中插入代码，用来支持虚机制。

      + 对象有虚函数（声明或继承）：编译期间会产生两个扩张操作：i 编译器产生一个vtbl，存放类的虚函数地址和type_info对象地址；ii 类的每个对象会多出一个vptr指针指向vtbl。

      + 对象使用了虚继承：如何使虚基类在其每一个派生类对象内存模型中的位置，能够于运行期准备妥当。（多态？）

        + ```c++
          class X {
          public:
              int i;
          };
          class A : public virtual X {};
          class B : public virtual X {};
          class C : public A, public B {};
          //由于pa的真正类型在运行时才确定，而虚基类对象在派生类对象中的偏移地址不确定。因此在编译期无法扩展pa->i为i在pa对象内存模型中的实际偏移地址。因此编译器需要在默认构造函数中插入这样的信息便于运行时确定。
          void foo(A* pa){
              pa->i = 1024;
          }
          ```

3. C++新手一般有两个常见的误解：

   + 任何类如果没有定义默认构造函数，就会被编译器定义出一个来。
   + 编译器定义出来的默认构造函数会明确设定类中每一个数据成员的初始化值。

4. 在C++各个不同的编译单元中，编译器如何避免**定义**出多个**默认构造函数**？
   
+ 解决办法是把**定义**的**默认构造函数**、**拷贝构造函数**、**析构函数**、**拷贝赋值运算符**都以inline方式完成。一个inline函数有静态链接期（static linkage），不会被编译单元以外者看到。如果函数太复杂，不适合做成inline，就会**定义**成一个explicit non-inline static实体？
  
5. 编译器什么时候会为类声明、定义**拷贝构造函数**？

   + **声明（declare）**：如果用户程序没有声明**拷贝构造函数**，那么编译器会声明一个implicite**拷贝构造函数**，这样的声明是trivial（无用的）。
   + **定义（define）**：只有当编译器需要声明的implicite拷贝构造函数执行某些编译器所需的行动，这时编译器会在需要调用拷贝构造函数的地方开始**定义**拷贝构造函数。这样的定义是nontrivial（有用的）。

6. 什么情况下编译器需要对象的**拷贝构造函数**完成某些必需的操作？

   + 拷贝语义：区分数据成员是内建基本数据类型还是对象。

     + memberwise copy semantics：按成员拷贝数据，对于成员对象，递归调用拷贝构造函数施行memberwise copy semantics。（对象）
     + bitwise copy semantics：按位拷贝数据（内建基本数据类型）

   + 编译器需要在对象的**拷贝构造函数**中插入代码，完成对成员对象、继承对象的**拷贝构造函数**的调用。

     + 对象含有成员对象，且成员对象有定义**拷贝构造函数**；
     + 对象继承自基对象，基对象有定义**拷贝构造函数**；

   + 编译器需要在对象的**拷贝构造函数**中插入代码，用来支持虚机制。

     + 对象有虚函数（声明或继承）：拷贝构造时，编译器可能（使用不同类的对象拷贝构造时）需要调整vptr指向地址。

       + ```c++
         class X {
         public:
             virtual void foo();
         };
         class Y : public X {};
         X* px = new Y;		Y y;
         X x = X* px;		X x = y;
         ```

     + 对象使用了虚继承：拷贝构造时，编译器可能（使用不同类的对象拷贝构造时）需要调整虚基类对象在派生类对象内存模型中的偏移地址信息。

       + ```c++
         class X {
         public:
             int i;
         };
         class A : public virtual X {};
         class B : public virtual X {};
         class C : public A, public B {};
         A a;				B b;
         X x = a;			X x = b;
         ```

7. 编译器会一一操作initialization list，以适当次序（成员在类中的声明顺序）在constructor之内安插初始化操作，并且在任何explicit use code之前。

## 第3章 Data语义学（The Semantics of Data）

1. 影响C++对象内存大小的因素

   + 所有的nonstatic数据成员；
   + 由编译器添加的数据成员，用以支持某些语言特性（主要是各种virtual特性：vptr、虚基类偏移）；
   + 对齐；
   + 编译器对于特殊情况所提供的可能的优化处理：（不同编译器对特殊情况的处理并不相同）
     + 对于一个空类，编译器需要插入一个字节的占位符；而对于空的虚基类，某些编译器可能优化掉派生类对象中虚基类的占位符。
   
2. 虚继承对象内存模型示例

   + ```c++
     class X {};
     class Y : public virtual X {};
     class Z : public virtual X {};
     class A : public Y, public Z {};
     ```

   + 如下图所示：（编译器：gcc version 8.1.0 (x86_64-posix-sjlj-rev0, Built by MinGW-W64 project；系统：X86_64）

     + X为空类，编译器插入一字节的占位符，用于为不同对象在内存中配置独一无二的地址；
     + Y，Z各持有一个指向虚基类子对象起始地址的指针，这里编译器优化掉了X子对象的占位符。
     + A含有Y和Z子对象，且X子对象的占位符也被编译器优化掉了。
   
   + ![图](./images/虚继承对象内存模型示例.png)
   
   + 别的编译器可能不会优化空虚基类的占位符，编译器之间的潜在差异正说明了C++对象模型的演化。这个模型为一般情况提供了解决之道。当特殊情况逐渐被挖掘出来时，种种启发（尝试错误）法于是被引入，提供优化的处理。如果成功，启发法于是就提升为普遍的策略，并跨越各种编译器而合并。它被视为标准（虽然它并不被规范为标准），久而久之也就成了语言的一部分。vtbl是一个好例子，另一个例子是NRVO。

3. 数据成员的绑定时机

   +  编译器对成员函数函数体的解析会直到整个类的声明完成了才会开始，因此在一个inline成员函数函数体内的数据成员绑定操作，会在整个类声明完成之后才发生。

   + 但是成员函数参数列表中的名称还是会在第一次出现时被决议（resolved）完成。因此需要将类中声明的typedef语句放在类的起始处，这样可以保证成员函数参数列表中的typedef被决议为类中声明而不是外部声明。

     + ```c++
       typedef int len;
       class X {
       public:
           //typedef long len;//放在这里可以避免val的类型被决议为int。
           //这里val的类型被决议为int，而_val的类型是long.
       	void foo(len val){
               _val = val;
           }
       private:
           typedef long len;
           len _val;
       }
       ```

4. 从对象存取数据成员与从对象指针或引用存取数据成员有什么差异？

   + 对于static数据成员：这两种方式没有任何区别，因为static数据成员作为全局静态变量统一存放在.data或.bss段，通过名称修饰机制解决访问控制与名称冲突问题。

   + 对于nonstatic数据成员：
     + 对于独立类、单一继承、多重继承：这两种存取方式没有区别，数据成员在类中的偏移在编译期即可获知。
     + 而对于虚继承：如果数据成员继承自虚基类，那么从对象指针或引用存取数据成员，由于不能确定具体对象，因此解析只能延迟到运行时，经由一个额外的间接引导，才能够解决。

5. 继承机制下的数据成员内存模型
   + 单一继承且不含虚函数：C++保证出现在派生类中的基类子对象有其完整原样性
     + ![图](./images/单一继承且不含虚函数.png)
   + 单一继承且含虚函数：vptr可以在对象内存模型的开始处，也可以在对象内存模型的结束处。
     + ![图](./images/单一继承且含虚函数.png)
   + 多重继承：对一个多重派生对象，将其地址指定给最左端（也就是第一个）基类的指针，情况将和单一继承时相同，因为二者都指向相同的起始地址；至于第二个或后继的基类的地址指定操作，则需要将地址修改。
     + ![图](./images/多重继承继承关系.png)
     + ![图](./images/多重继承.png)
   + 虚拟继承：对于Vertex这样继承自虚基类的类，将其分割为两部分：一个不变局部和一个共享局部（虚基类部分），不变局部有固定的类偏移，共享局部则会因为继续派生Vertex导致其类偏移一直变化；一般的布局策略是先安排好不变局部，然后再建立起共享局部；
     + 对共享局部的访问有两种实现方法：
       + 其一是为派生类的每一个虚基类生成一个虚基类指针，指向虚基类起始地址。
       + 其二是在派生类的虚函数表起始地址之前存放所有虚基类的类偏移，这样vtbl[0,1,...]支持虚函数机制，vtbl[-1,-2,...]支持虚继承机制。
     + ![](./images/虚继承继承关系.png)
     + ![图](./images/虚基类指针策略实现的虚继承.png)
     + ![图](./images/虚基类偏移表策略实现的虚继承.png)
6. C++为了支持虚函数需要的额外开销
   + 为每个含有虚函数的类生成一个vtbl，用于存放它所声明的虚函数地址，以及type_info地址（用于RTTI）。
   + 为每个含有虚函数的类对象生成一个vptr指针，指向vtbl。
   + 在构造函数（包括拷贝构造）中插入相关代码，为类对象设置vptr值。这可能意味着在继承体系下的从基类到最底派生类的构造函数调用中，不断重置vptr值，其情况视编译器优化的积极性而定。
   + 在析构函数中插入相关代码，使它能够抹消vptr值。这可能意味着在继承体系下的从最底派生类到基类的析构函数调用中，不断抹消vptr值，其情况视编译器优化的积极性而定。

7. 指向数据成员的指针(原文讲的并不清晰，因此//TODO)

   + ```c++
     //取数据成员的地址将会得到类偏移
     &Point::z;
     //取绑定于具体对象的数据成员的地址将会得到成员的内存地址
     Point p;
     &p::z;
     ```

   + 可以通过指向数据成员的指针判断vptr在对象内存布局中的位置

     + ```c++
       //在我的平台上（编译器：gcc version 8.1.0 (x86_64-posix-sjlj-rev0, Built by MinGW-W64 project；系统：X86_64）
       //三个数据成员的类偏移分别为8、12、16，因此vptr位于对象开头。
       class Point {
       public:
           int x, y, z;
           virtual void foo();
       };
       int main(){
           printf("%p\n", &Point::x);//使用cout输出三条语句都返回1？
           printf("%p\n", &Point::y);
           printf("%p\n", &Point::z);
       }
       ```

   + 为了区分指向第一个数据成员的指针和没有指向任何数据成员的指针，所有的真正的成员类偏移都被加上了1。

     + ```c++
       //如何区分
       int Point::* p1 = 0, Point::* p2 = &Point::x;
       ```

## 第4章 Function语义学（The Semantics of Function）

1. 成员函数分类
   + nonstatic成员函数、virtual函数、static成员函数。
2. nonstatic成员函数调用方式：C++的设计准则之一就是：nonstatic成员函数至少必须和非成员函数有相同的效率。因此对于nonstatic成员函数，编译器通过如下步骤将其转化为一个非成员函数形式调用：
   + 改写函数的signature（函数签名），插入形参this指针。
   + 将每一个对nonstatic数据成员的存取操作改为经由this指针来存取。
   + 此时函数已经成为一个非成员函数，编译器通过名称修饰机制解决访问控制与名称冲突问题。

```c++
//nonstatic成员函数							 //转化为非成员函数，使用了NRVO机制
Point3d Point3d::normalize() const{			void normalize_7Point3dFv(
    											register const Point3d* const this, 	
    											Point3d& _result){
    
    register float mag = magnitude();			register float mag = this->magnitude();			
    Point3d normal;								_result.Point3d::Point3d();
    
    normal._x = _x/mag;							_result._x = this->_x/mag;
    normal._y = _y/mag;							_result._y = this->_y/mag;	
    normal._z = _z/mag;							_result._z = this->_z/mag;
    return normal;								return;
}											}
```

3. virtual成员函数调用方式：virtual函数通过对象调用时，在编译期使用与nonstatic成员函数一样的方式被决议；通过对象指针或引用表现多态时，通过vptr->vtbl在运行期被决议。

   + vptr也会使用名称修饰机制，因为在一个复杂的继承体系中，可能存在多个vptrs。（多重继承、虚拟继承）

   + ```c++
     ptr->normalize();		(*ptr->vptr[1])(ptr);
     ```

4. static成员函数调用方式：static成员函数与非成员函数调用没有什么区别，它的主要特性就是没有this指针，这导致了以下次要特性：

   + 它不能直接存取nonstatic成员（数据、函数）；
   + 它不能被声明为const、volatile或virtual；
   + 它不需要经由对象才能被调用——虽然大部分时候它还是这样被调用的。

   + > //取static成员函数的地址将会得到它的内存地址，而且其类型是一个指向非成员函数的指针。
     > void (~~class::~~ *ptr)() = &class::static_func()

5. 多重继承体系下的虚函数支持

   + 在多重继承下，派生类中含有与上一层基类个数相同的vtbls和vptrs，通过名称修饰机制解决访问控制与名称冲突问题。如本例中的vtbl_Derived、vptr_Derived和vtbl_Base2_Derived、vptr_Derived。

   + 当将一个Derived对象地址指定给一个Base1指针或Derived指针时，被处理的vtbl是主虚表vtbl_Derived；而当指定给一个Base2指针时，被处理的vtbl是次虚表vtbl_Base2_Derived。

   + 将Derived对象地址指定给Base2指针来实现多态时必须在运行期调整this指针，这样一个调整是通过桩代码来实现的，实现机制与延迟绑定中的GOT+PLT相似。

   + ![图](./images/多重继承4.png)

   + ![图](./images/多重继承内存模型4.png)

   + 上图中三个需要调整this指针的虚函数调用情况分别如下：

   + ```c++
     1.第一种情况：虚析构函数调用
     Base2* ptr = new Derived;//这个赋值操作会调整ptr指向Base2子对象起始地址，这个调整操作在编译期完成
     //调用Derived::~Derived(),ptr必须向后调整sizeof(Base1)个字节。
     //这个调整操作需要额外的偏移信息以及相关指令，这里不谈。
     delete ptr;
     
     2.第二种情况：使用Derived指针调用从Base2继承而来的虚函数
     Derived* ptr = new Derived;
     //调用Base2::mumble(),ptr必须向前调整sizeof(Base1)个字节
     ptr->mumble();//这里为什么是调用Base2::mumble()，是不是前述的赋值语句是这样的：Derived* ptr = new Base2;但这个向下转型是错误的。
     
     3.第三种情况：支持虚函数返回值不同的这样一个语言的扩充性质：可以返回基类对象，也可返回派生类对象。
     Base2* pb1 = new Derived;
     //调用Derived* Derived::clone(),返回值必须被调整，以指向Base2子对象
     Base2* pb2 = pb1->clone();
     ```

6. 虚继承体系下的虚函数支持

   + 派生类与虚基类之间的指针转换也需要调整this指针。作者建议：不要在一个虚基类中声明nonstatic数据成员，不然，你在凝视深渊，同时深渊也在凝视你。

   + ![图](./images/虚继承4.png)

   + ![图](./images/虚继承内存模型4.png)

7. 指向成员函数的指针

   + 取一个nonstatic数据成员的地址，得到的是该成员的类偏移（再加1），它是一个不完整的值，需要被绑定于某个具体对象的地址上，才能够被存取；

   + 取一个nonstatic，non-virtual成员函数的地址，得到的是该函数的内存地址，但是该值也是不完全的，需要被绑定于某具体对象的地址上，才能够被调用。

   + 取一个virtual成员函数的地址，由于多态机制，真正的内存地址只能在运行时被决议，因此得到的是vtbl中的slot偏移（大概同一个虚函数在继承体系下的基类和派生类的一个或多个vtbl中处于相同slot）。为了让指向成员函数的指针同时支持virtual函数与non-virtual函数，需要一些复杂的操作，这里不谈。

   + ```c++
     //声明一个指向成员函数的指针
     double (Point::* pmf)();
     //赋值
     pmf = &Point::x;
     //调用
     Point p;
     (p.*pmf)();
     ```

8. inline函数

   + 关键词inline只是一个请求，如果这个请求被接收，编译器就必须认为它可以用一个表达式合理地将这个函数扩展开来。如果不能，那么这个请求会被驳回。

   + 编译器通过测试扩展前后函数调用的各种操作的开销对比来判断是否能合理扩展。

   + 如果inline函数被展开，编译器可能需要为形式参数或局部变量生成大量临时对象。

   + 此外，inine中再有inline，可能导致一个表面上看起来平凡的inline却因其连锁复杂度而没办法扩展开来。这种情况可能发生于复杂继承体系下的构造函数，或是其他一些表面上并不正确的inline调用组成的串联——它们每一个都会执行一小组运算，然后对另一个对象发出请求。