+++
title = 'Golang 编程备注'
date = 2024-11-24T09:04:10Z
draft = false
tags = ['go']
categories = ['Golang']
+++

Go 是一门以简洁设计为原则的语言，虽说简洁但因其语言特性有些地方还是要去注意一下的。

<!--more-->

## 用字符串索引得到的不是字符类型而是 byte 类型

这个就很莫名，可能 python 代码敲多了想当然认为用字符串索引可以直接拿到字符。
```go
import "fmt"

func main() {
   s := "sdfsfs"
   fmt.Println(s[1]) // 结果是100， 不是字符d
   fmt.Println(s[1:2]) // 切片的形式拿到 "d"
   
   char := string(s[1]) // []byte -> string 转换的形式拿到 "d"
   fmt.Println(char) // "d"
}
```
如果想要拿到字符，需要用切片的方式或者将 byte 数组转化为字符串， go 中没有 char 类型。

## 切片和结构体在函数参数中的值传递和引用传递

在 Go 中，函数的参数是通过值传递的。这意味着当你传递一个切片或者结构题到函数时，实际上传递的是一个切片的副本。虽然切片本身是一个指针、长度和容量的结构体，但在函数内部对这个切片变量重新赋值（如 `nums1 = result`）不会影响函数外部的切片，如果要改变原参数的值，那么：
- 对结构体而言，使用指针来引用传递
- 对于切片而言，指针来引用传递或者循环直接修改切片内部的值，因为当将切片值传递到函数时，Go 会复制这个结构体，但底层数组是共享的，所以对切片底层数组的修改（通过 `slice[i] = value`）会影响外部的切片
```go
package main

import "fmt"

// 尝试通过重新赋值修改切片（不会影响外部变量）
func modifySliceByValue(slice []int) {
    newSlice := []int{10, 20, 30}
    slice = newSlice // 对切片重新赋值，不影响外部变量
    fmt.Println("Inside modifySliceByValue:", slice)
}

// 使用循环直接修改切片内部的值（会影响外部变量）
func modifySliceContents(slice []int) {
    for i := range slice {
        slice[i] = slice[i] * 2 // 修改切片内部的值
    }
}

// 使用指针修改切片（影响外部变量）
func modifySliceWithPointer(slice *[]int) {
    newSlice := []int{100, 200, 300}
    *slice = newSlice // 通过指针重新赋值，影响外部变量
}

func main() {
    nums := []int{1, 2, 3}

    fmt.Println("Before modifySliceByValue:", nums)
    modifySliceByValue(nums)
    fmt.Println("After modifySliceByValue:", nums) // nums 未被修改

    fmt.Println("Before modifySliceContents:", nums)
    modifySliceContents(nums)
    fmt.Println("After modifySliceContents:", nums) // nums 内容被修改

    fmt.Println("Before modifySliceWithPointer:", nums)
    modifySliceWithPointer(&nums)
    fmt.Println("After modifySliceWithPointer:", nums) // nums 被重新赋值
}
```

## 使用 make 时没有注意到 length 和 capacity 导致可能的错误

在 Go 中使用 make 函数可以初始化一个切片：
```go
nums := make([]int, length, capacity)
```
这里需要注意的是就算直接给 capacity 足够的长度，length 没有设置足够长的话，也会发生数组越界错误：
```go
res := make([]int, 0, 10)
res[9] // index of range，因为数组的长度是0
```
这里就有一个疑问是如果这样那要这个 capacity 有毛线用😅，反正还是根据数组长度来判断越不越界，应该和 go 的切片实现机制有关系，有时间可以研究研究下。

这里容易踩的另一个坑就是当指定了长度来初始化切片后，用 append 方法来向切片中添加元素是会直接从切片末尾来添加，而不是切片最开始的地方。
```go
result := make([]int, m+n)
result = append(result, 1)
```
这会创建一个长度为 `m+n` 的切片，同时会将这些位置初始化为默认值（整数默认是 `0`）。但是随后用 `append` 向 `result` 中追加元素时，实际上是往 `result` 的 **后面追加**，而不是覆盖掉前面的初始值，这导致 `result` 的长度会超过 `m+n`。

修正 `result` 的初始化方式，确保它的初始长度为 `0`（只分配容量），这样 `append` 操作不会导致超长。
```go
result := make([]int, 0, m+n) // 初始长度为 0，容量为 m+n
result = append(result, 1) // 从索引0开始给切片添加元素
```

## make 和 new 的区别

数据结构确定时用 make，否则如果只想要一个指针则用 new。
- **`make` 不返回指针**，是为需要底层数据结构的类型（slice、map、channel）专门设计的，它会初始化这些类型的底层结构（适用于 slice、map、channel）。
- `new` 是一个通用的工具，只做最基本的内存分配，并不关心具体类型的特殊需求，返回指针。
```go
package main

import "fmt"

func main() {
    // 使用 new 分配一个 int
    p := new(int)
    fmt.Println(*p) // 输出 0，零值
    *p = 42         // 修改通过指针指向的值
    fmt.Println(*p) // 输出 42

    // 使用 new 分配一个 map（不可直接使用）
    mp := new(map[string]int)
    // (*mp)["key"] = 42 // 会 panic，因为底层未初始化
    *mp = make(map[string]int) // 需要用 make 初始化底层 map
    (*mp)["key"] = 42 // 现在可以正常使用了
    fmt.Println(mp) // 输出 &map[]
    
    mp2 := make(map[string]int)
    mp2["key"] = 42 // 通过make来创建map可以直接使用
    fmt.Println(mp2) // map[key:42]
}
```

## 将方法挂在结构体 vs 定义在包里
在 Go 中一个方法可以被挂载载结构体上使用（这种被称为方法），也可以和结构体无关直接作为单独的函数定义在包中来通过包名来直接使用，有必要思考下这 2 种使用方式的不同以及适合的场景。

方法和函数的使用方式：
```go
// 方法
type MyStruct struct {
    Name string
}

func (m *MyStruct) Greet() string {
    return "Hello, " + m.Name
}

obj := MyStruct{Name: "Alice"}
fmt.Println(obj.Greet())  // 调用方法

// 函数
package mypackage

func Greet(name string) string {
    return "Hello, " + name
}

import "mypackage"
fmt.Println(mypackage.Greet("Alice"))  // 调用独立函数
```

**方法**：适合在结构体上定义与结构体行为相关的功能，尤其是当需要访问或修改结构体内部状态时。  
**独立函数**：适合处理与特定结构体无关的通用功能，便于复用，适合没有状态管理需求的功能。

## 通过导入 init 函数来初始化

## 如何实现AOP类似的功能
## 