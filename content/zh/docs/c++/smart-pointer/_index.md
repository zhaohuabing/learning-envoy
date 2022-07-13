---
title: "Smart Pointers"
linkTitle: "Smart Pointers"
weight: 2
description: >
  Smart pointers 可以自动跟踪一个对象的引用次数并管理对象的内存。
---

Smart pointers 可以自动跟踪 object 的引用次数，并在指针退出其作用域时自动减少引用次数，当引用次数为零时自动删除其对应的内存空间。 Smart Pointer 提供了类似 Java, Golang 语言的内存回收机制，简化了 C++ 的内存管理。