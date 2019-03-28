---
layout:     post
title:      effectiveC++
subtitle:   
date:       2019-03-17
author:     jabin
header-img: 
catalog: true
tags:
    - c++
---

# 一、 正确理解c++
## 1. 组成c++的几大块
1. c： 基础
2. Object-Oriented c++：面向对象
3. Template c++: 模板、泛型编程
4. STL: 一套模板类。 容器(containers)、迭代器(iterators)、算法(algorithms)
## 2. 尽可能用const
```
//1. 修饰普通变量
cosnt a = 8;
//2. 修饰指针变量
const int *p = 8;  //*前面， 指针指向的内容不可更改
int* const p = 9; //*后面， 指针本身不可更改
const int* const p=9; //指针和指针本身都不可更改
//3. const 参数传递
void cpf(const int a)
{
    a++;  //报错， a不可更改
} 
//4. 修饰类成员函数
```
## 3. 使用对象/变量前 都显示初始化
1. 内置类型手工初始化。 ```int a = 0;```
2. 构造函数使用成员初值列， 而不是赋值

# 二、设计与声明
## 1. 接口/函数的设计要容易被正确使用
1. 参数的定义、顺序
2. 参数判断、防错

