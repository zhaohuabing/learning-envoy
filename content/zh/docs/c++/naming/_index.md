---
title: "命名规范"
linkTitle: "命名规范"
weight: 1
description: >
  C++ 命名规范
---

Envoy 采用 [Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html)，除此以外，还有一些自定义的[编码规范](https://github.com/envoyproxy/envoy/blob/main/STYLE.md)。

## 文件名
小写字母，下划线(_)分割单词。如：my_useful_class.cc

## 类型名/方法名
驼峰形式，首字母大写，如：MyExcitingClass  AddTableEntry()

## 变量名

变量采用小写字母，下划线(_)分割单词的命名方式，如： a_local_variable

class 变量会在最后额外加一个下划线，如：a_class_data_member_
