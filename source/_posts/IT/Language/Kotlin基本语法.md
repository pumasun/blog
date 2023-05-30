---
title: Kotlin基本语法
date: 2023-02-9 07:57:03
tags: ['IT','Kotlin','Programming Language', 'Android']
category: ['IT', 'Android']
---

## [官方文档](https://kotlinlang.org/docs/basic-syntax.html)
## 包的定义与import

包的声明和导入应该在源文件的顶部。
``` Kotlin
package my.demo

import kotlin.text.*

// ...
```
不需要目录和包匹配:源文件可以放置在文件系统中任意位置。

## 程序的入口
Kotlin应用的入口是main函数
``` Kotlin
fun main() {
    println("Hello world!")
}
```

另一个形式的main函数可以接收多个String类型参数
``` Kotlin
fun main(args: Array<String>) {
    println(args.contentToString())
}
```


## 向标准输出打印
``` Kotlin
print函数向标准输出打印其参数
print("Hello ")
print("world!")
println函数打印其参数并添加换行符，因此其他的打印信息将出现下一行。
println("Hello world!")
println(42)
```


## 函数
拥有两个Int型参数和一个Int型返回值的函数
``` Kotlin
fun sum(a: Int, b: Int): Int {
    return a + b
}
```

函数体可以是一个表达式。它的返回类型将根据表达式的值推导。
``` Kotlin
fun sum(a: Int, b: Int) = a + b
```

一个返回值无意义的函数
``` Kotlin
fun printSum(a: Int, b: Int): Unit {
    println("sum of $a and $b is ${a + b}")
}
```

Unit返回值类型可以省略
``` Kotlin
fun printSum(a: Int, b: Int) {
    println("sum of $a and $b is ${a + b}")
}
```


## 变量
只读本地变量使用val声明，他们仅可以被赋值一次。
``` Kotlin
val a: Int = 1  // immediate assignment
val b = 2   // `Int` type is inferred
val c: Int  // Type required when no initializer is provided
c = 3       // deferred assignment
```

使用关键字var的变量可以重新赋值。
``` Kotlin
var x = 5 // `Int` type is inferred
x += 1
```

可以在顶层声明变量
``` Kotlin
val PI = 3.14
var x = 0

fun incrementX() { 
    x += 1 
}
```


## 创建类和实例
使用class关键字创建类
``` Kotlin
class Shape
```

类的属性可以在其声明或者类体内罗列
``` Kotlin
class Rectangle(var height: Double, var length: Double) {
    var perimeter = (height + length) * 2
}
```

类定义中列出的默认带参数构造函数默认是可用的
``` Kotlin
val rectangle = Rectangle(5.0, 2.0)
println("The perimeter is ${rectangle.perimeter}")
```

类的继承使用冒号(:)声明。类默认是final的；将类标记为open使之可被继承。
``` Kotlin
open class Shape
class Rectangle(var height: Double, var length: Double): Shape() {
    var perimeter = (height + length) * 2
}
```


## 注释
就像大多数现代编程语言一样，Kotlin支持单行(或行末)注释和多行(块)注释。
``` Kotlin
// This is an end-of-line comment

/* This is a block comment
   on multiple lines. */
```

块注释是可以嵌套的
``` Kotlin
/* The comment starts here
/* contains a nested comment *⁠/
and ends here. */
```


## 字符串模板
``` Kotlin
var a = 1
// simple name in template:
val s1 = "a is $a" 

a = 2
// arbitrary expression in template:
val s2 = "${s1.replace("is", "was")}, but now is $a"
```


## 条件表达式
``` Kotlin
fun maxOf(a: Int, b: Int): Int {
    if (a > b) {
        return a
    } else {
        return b
    }
}
```

在Kotlin中，if也可以作为表达式
``` Kotlin
fun maxOf(a: Int, b: Int) = if (a > b) a else b
```


## For循环
``` Kotlin
val items = listOf("apple", "banana", "kiwifruit")
for (item in items) {
    println(item)
}
```

或
``` Kotlin
val items = listOf("apple", "banana", "kiwifruit")
for (index in items.indices) {
    println("item at $index is ${items[index]}")
}
```


## While循环
``` Kotlin
val items = listOf("apple", "banana", "kiwifruit")
var index = 0
while (index < items.size) {
    println("item at $index is ${items[index]}")
    index++
}
```

## When表达式
``` Kotlin
fun describe(obj: Any): String =
    when (obj) {
        1          -> "One"
        "Hello"    -> "Greeting"
        is Long    -> "Long"
        !is String -> "Not a string"
        else       -> "Unknown"
    }
```


## 范围
使用in判断一个数字是否在指定的范围内
``` Kotlin
val x = 10
val y = 9
if (x in 1..y+1) {
    println("fits in range")
}
```

判断一个数字是否不在范围内
``` Kotlin
val list = listOf("a", "b", "c")

if (-1 !in 0..list.lastIndex) {
    println("-1 is out of range")
}
if (list.size !in list.indices) {
    println("list size is out of valid list indices range, too")
}
```

遍历范围
``` Kotlin
for (x in 1..5) {
    print(x)
}
```

遍历一个序列
``` Kotlin
for (x in 1..10 step 2) {
    print(x)
}
println()
for (x in 9 downTo 0 step 3) {
    print(x)
}
```


## 集合 
遍历集合
``` Kotlin
for (item in items) {
    println(item)
}
```

使用in判断集合是否包含指定的条目
``` Kotlin
when {
    "orange" in items -> println("juicy")
    "apple" in items -> println("apple is fine too")
}
```

使用lamda表达式过滤或映射集合
``` Kotlin
val fruits = listOf("banana", "avocado", "apple", "kiwifruit")
fruits
    .filter { it.startsWith("a") }
    .sortedBy { it }
    .map { it.uppercase() }
    .forEach { println(it) }
```


## 可为Null的值和Null检查
一个引用的值可能为null时，必须显示地将其声明为可为null的。可为null的类型以?结尾。
如果str不包含一个整数，则返回null
``` Kotlin
fun parseInt(str: String): Int? {
    // ...
}
```

使用函数返回可为null的值
``` Kotlin
fun printProduct(arg1: String, arg2: String) {
    val x = parseInt(arg1)
    val y = parseInt(arg2)

    // Using `x * y` yields error because they may hold nulls.
    if (x != null && y != null) {
        // x and y are automatically cast to non-nullable after null check
        println(x * y)
    }
    else {
        println("'$arg1' or '$arg2' is not a number")
    }    
}
```

或
``` Kotlin
// ...
if (x == null) {
    println("Wrong number format in arg1: '$arg1'")
    return
}
if (y == null) {
    println("Wrong number format in arg2: '$arg2'")
    return
}

// x and y are automatically cast to non-nullable after null check
println(x * y)
```


## 类型检测和自动转换 
is操作符检查表达式的值是否为指定的类型。如果检查一个不可修改的本地变量或属性是否为指定类型，则没有必要显示地进行转换
``` Kotlin
fun getStringLength(obj: Any): Int? {
    if (obj is String) {
        // `obj` is automatically cast to `String` in this branch
        return obj.length
    }

    // `obj` is still of type `Any` outside of the type-checked branch
    return null
}
```

或
``` Kotlin
fun getStringLength(obj: Any): Int? {
    if (obj !is String) return null

    // `obj` is automatically cast to `String` in this branch
    return obj.length
}
```
亦或
``` Kotlin
fun getStringLength(obj: Any): Int? {
    // `obj` is automatically cast to `String` on the right-hand side of `&&`
    if (obj is String && obj.length > 0) {
        return obj.length
    }

    return null
}
```
