---
title: "Smart Pointers"
linkTitle: "Smart Pointers"
weight: 1
description: >
  Smart pointers 可以自动跟踪一个对象的引用次数并管理对象的内存。
---

Smart pointers 可以自动跟踪 object 的引用次数，并在指针退出其作用域时自动减少引用次数，当引用次数为零时自动删除其对应的内存空间。 Smart Pointer 提供了类似 Java, Golang 语言的内存回收机制，简化了 C++ 的内存管理。

## Raw Pointer 的问题

使用 Raw Pointer 来指向通过 new 关键字分配的内存后，必须在代码中使用 delete 关键字来删除内存。如果存在一些异常分支未能删除内存，会导致内存泄露。

例如在下面的程序中，当 x== 45 时，程序直接返回，导致内存泄露。

```C++
void my_func()
{
    int* valuePtr = new int(15);
    int x = 45;
    // ...
    if (x == 45)
        return;   // here we have a memory leak, valuePtr is not deleted
    // ...
    delete valuePtr;
}
 
int main()
{
}
```

## Smart Pointer 的原理

Smart Pointer 是 C++ 提供的自动回收内存的机制。其原理是利用 destructor 来清理指针指向的内存。由于当一个对象退出其作用域时，其 destructor 一定会被调用到，因此该机制保证了分配的内存一定会得到清理。下面代码中的 SmartPtr 类演示了 Smart Pointer 的实现原理。

```C++
#include <iostream>
using namespace std;
 
class SmartPtr {
    int* ptr; // Actual pointer
public:
    // Constructor: Refer https:// www.geeksforgeeks.org/g-fact-93/
    // for use of explicit keyword
    explicit SmartPtr(int* p = NULL) { ptr = p; }
 
    // Destructor
    ~SmartPtr() { 
      delete (ptr); 
      cout << "allocated memory has been deleted"
      }
 
    // Overloading dereferencing operator
    int& operator*() { return *ptr; }
};
 
int main()
{
    SmartPtr ptr(new int());
    *ptr = 20;
    cout << *ptr;
 
    // We don't need to call delete ptr: when the object
    // ptr goes out of scope, the destructor for it is automatically
    // called and destructor does delete ptr.
 
    return 0;
}

 [运行上面的示例代码](http://cpp.sh/8zap3)

```

## 参考阅读

* https://www.geeksforgeeks.org/smart-pointers-cpp/