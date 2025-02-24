+++
title = 'Python 教程 - 基础篇'
date = 2025-02-17T12:53:20Z
tags = ['python']
categories = ['Backend', 'Python']
+++

写个很简单的 python 教程给 0 基础的人入门用。

<!--more-->

Python 在所有编程语言中算是很简单的一门了，个人是很推荐新手来学的，而且应用非常广泛，脚本语言，后端开发，自动化开发啥的都能用的到。

学习前先把运行环境给搭建起来，这里推荐用 pycharm 的 notebook 来从 0 熟悉 python 语法。pycharm 安装什么的就不说了，网上一搜一大堆，安装好了打开选择新建 notebook，然后就可以在 noteook 的代码块中写代码并点击左边的那个绿色按钮来运行，非常之方便。

一般 python 代码由这些部分组成：
1. 变量定义：把字符串或数字赋值给一个有名字的代称供程序后面来使用
2. 流程控制：if 判断和循环
3. 函数：将一组操作封装在一起作为一个可重复利用的代码块
4. 错误和异常管理：但程序运行报错时如何处理
5. 包管理：代码非常多时如何模块化管理
6. 面向对象：类和对象，可以减少重复代码并且能使代码更加直观
7. 其他：装饰器，lambda 函数啥啥啥的

接下来的学习流程通过例子来学习，建议在 notebook 里敲这些代码，然后运行看看输出是什么。
先用一个简单的例子来说明前 3 项，我们来开发一个程序，用来计算某个人的 BMI（Body Mass Index，身体质量指数），并根据 BMI 值给出健康建议。
```python
# 定义一些变量来使用
PI = 3.14159  # 浮点数，表示圆周率
name = "小明"  # 字符串，表示用户姓名
height = 1.75  # 浮点数，表示身高（单位：米）
weight = 68  # 整数，表示体重（单位：千克）

bmi = weight / (height ** 2)  # 计算 BMI 值，变量定义了后就可通过它的名字来用

# 流程控制
if bmi < 18.5:
    category = "偏瘦"
elif 18.5 <= bmi < 24.9:
    category = "正常"
elif 24.9 <= bmi < 29.9:
    category = "超重"
else:
    category = "肥胖"

# 函数定义，这个函数要接受3个参数，并将最后的结果返回
def show_bmi_info(name, bmi, category):
    # 生成 BMI 计算结果
    return f"{name} 的 BMI 是 {bmi:.2f}，属于 {category}。"

# 调用函数，传入3个参数
result = show_bmi_info(name, bmi, category)
# 用 print 输出
print(result)
```
运行程序，结果应该是 
```python
小明 的 BMI 是 22.20，属于 正常。
```

接下来是对程序的解释：
- 以 `#` 符号开头的是代码的注释，目的是方便人们理解代码，对代码的运行没有影响，可以放心弄。
- python 的代码执行属于解释型语言，所以解释型语言就是和同声传译那样一行一行的向下执行，执行完一行代码再执行下一行代码，和解释型语言相对应的是编译型语言，如 java。
- python 代码对于缩进的要求很严格，按照冒号加缩进来区分为一组代码块。
- 函数的定义用 `def` 关键字来定义，然后依次是函数名，函数要接收的参数，函数内部的操作，最后是将结果用 `return` 关键字返回（如果没有结果时也可以不用返回）。
- `f"{name} 的 BMI 是 {bmi:.2f}，属于 {category}。"` 这个是 python 用于把变量组合在字符串里的语法，有时候字符串的一部分是不固定的，比如 "姓名：{实际的名字}"，明显这里的实际的名字是一个变量。
- 使用函数的时候直接通过函数名再把需要的参数传入即可，这里用 result 变量来接受了函数的返回值。
- 最后的 `print(result)`, 这里的 `print` 是 `python` 自带的函数，就是 `python` 已经事先定义好了我们直接用就可以了，因为输出内容这个功能是个非常常用的功能，从这里也能看出来定义函数的好处，就是每次使用时直接拿来用就可以了，不需要再把函数里定义的那些操作再写一遍。

再来一个完整的例子来演示 python 中的错误和异常处理，包管理，这里以 python 的文件操作为例子，让 python 输出
```python
import os  # 导入 os 模块用于文件操作

# 写入文件
def write_file(file_path, content):
    # 将可能会发生错误的代码块放在try里
    try:
        with open(file_path, "w", encoding="utf-8") as f:
            f.write(content)
        print(f"成功写入 {file_path}")
    # 然后在excpet中处理这些错误
    except Exception as e:
        print(f"写入错误：{e}")

# 读取文件
def read_file(file_path):
    try:
        with open(file_path, "r", encoding="utf-8") as f:
            print(f.read())
    except Exception as e:
        print(f"读取错误：{e}")

# 删除文件
def delete_file(file_path):
    try:
        os.remove(file_path)
        print(f"已删除 {file_path}")
    except Exception as e:
        print(f"删除错误：{e}")

# 示例使用
file_name = "test.txt"
write_file(file_name, "Hello, Python!")  # 写入
read_file(file_name)  # 读取
delete_file(file_name)  # 删除
```
代码的解释：
- `import os `，使用 import 关键字来倒入 os 模块，python 已经将文件的一些通用操作的代码放在了 os 模块里（其实就是一个文件夹），想要用里面的函数的话 import 这个某块就可以了。
- `try except`，文件操作是个会出错的操作，比如文件不存在或者文件损坏了，对于这样的需要将这些操作放在 try 里面，然后万一出错了的话在 except 中处理。
- `with open(file_path, "w", encoding="utf-8") as f:`  python 中打开文件的固定操作
- `os.remove(file_path)` 因为我们导入了 os 模块，所以可以直接在代码里用它的 remove 方法来删除文件

最后来一个例子来演示 python 中面向对象的使用：
```python
# 1. 定义父类 Student
class Student:
    def __init__(self, name, age, grade):
        self.name = name
        self.age = age
        self.grade = grade  # 年级

    def introduce(self):
        return f"我是 {self.name}，{self.age} 岁，就读于 {self.grade} 年级。"

# 2. 定义子类 UniversityStudent，继承 Student
class UniversityStudent(Student):
    def __init__(self, name, age, grade, major):
        super().__init__(name, age, grade)  # super()可以拿到父类中的方法从而调用父类的初始化方法
        self.major = major  # 专业

    def introduce(self):
        return f"我是 {self.name}，{self.age} 岁，大学 {self.grade} 年级，主修 {self.major}。"

    def study(self):
        return f"{self.name} 正在学习 {self.major} 相关课程。"

# 3. 创建对象
student1 = Student("小明", 15, 9)
uni_student = UniversityStudent("小红", 20, 2, "计算机科学")

# 4. 调用方法，并用 print 输出
print(student1.introduce())  # 调用父类方法
print(uni_student.introduce())  # 调用子类重写的方法
print(uni_student.study())  # 调用子类独有的方法
```
代码的解释：
- 利用 class 关键字来定义类，类就是类别的意思，简单而言就是程序对现实世界的抽象，比如这里定义了一个 Student 类
- 类里有一些属性，比如 Student 里的 name, age, grade，在 `__init__` 函数中（这个函数名时固定的，python 中特殊的函数）提供对这些属性的初始化操作。
- 类里除了属性外，也可以定义类自己的一些函数（也被称为方法），比如这里 introduce 函数用于自我介绍
- 定义了类之后，可以派生出子类，比如从 Student 派生出 UniversityStudent，实现了代码复用，大学生里面只多处专业属性和 study 方法。
- 定义类后不能直接使用，需要用它来实例化一个对象才能用，`student1 = Student("小明", 15, 9)`, 这个很好理解，因为类描述了一类事物的属性，比如学生，是一个概念性质的东西，现实时间还是以“学生小明”这样的具体事物来运作。
- `def introduce(self)` 这里的 self 的作用是将这个方法挂载在类的实例，也就是对象上，一般当然是具体的对象来使用方法，但是有些方法我们希望可以直接通过类名来用，这个以后再说。