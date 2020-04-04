# C++：右值引用与转移语义

1. 左值与右值的区别：左值是命名对象，右值是临时对象。

2. 引入右值引用特性的目的

   + 支持转移语义：转移语义可以将资源 ( 堆，系统对象等 ) 从一个对象转移到另一个对象，这样能够减少不必要的临时对象的创建、拷贝以及销毁，能够大幅度提高C++应用程序的性能。

     + 支持转移语义的对象必须实现移动构造函数（移动赋值运算符）。

     + 编译器只有对右值引用才能调用移动构造函数（移动赋值运算符），因此std::move()支持将左值引用转换为右值引用。

       + ```c++
         void ProcessValue(int& i) { 
          std::cout << "LValue processed: " << i << std::endl; 
         } 
          
         void ProcessValue(int&& i) { 
          std::cout << "RValue processed: " << i << std::endl; 
         } 
          
         int main() { 
          int a = 0; 
          ProcessValue(a); 
          ProcessValue(std::move(a)); 
         }
         
         //运行结果
         LValue processed: 0 
         RValue processed: 0
         ```

   + 支持完美转发（perfect forwarding）：适用于需要将一组参数原封不动（左值/右值、const/non-const）的传递给另一个函数的场景，能够更简洁明确地定义泛型函数。C++11 中定义的 T&& 的推导规则为：右值实参为右值引用，左值实参仍然为左值引用。

     + //TODO

参考文献

[https://www.ibm.com/developerworks/cn/aix/library/1307_lisl_c11/index.html](https://www.ibm.com/developerworks/cn/aix/library/1307_lisl_c11/index.html)