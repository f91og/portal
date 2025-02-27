+++
title = 'Python 教程 - 基础篇'
date = 2025-02-17T12:53:20Z
tags = ['python']
categories = ['Backend', 'Python']
+++

尝试用尽可能简单的语言，写个尽可能简短的 Python 教程给零基础的人入门用。

<!--more-->

Python 在所有编程语言中算是很简单的一门了，个人是很推荐新手来学的，而且应用非常广泛，脚本语言，后端开发，自动化开发啥的都能用的到。

学习前先把运行环境给搭建起来，这里可以用 pycharm 这样的 IDE 来运行 python 程序，不过更推荐的是从命令运行 python 代码，这样有助于了解代码是怎么跑起来的。

关于什么是命令行，简单而言就是我们不用图形化程序（pycharm 这样的）的帮助，将一条一条指令发给计算机让它执行。可以看这个教程 [命令行命令一览——CLI 教程](https://www.freecodecamp.org/chinese/news/command-line-commands-cli-tutorial/)，也可以跟着以下的操作来快速熟悉命令行中 python 的使用，这里以 windows 为例（相信用 mac 为主力的程序员已经基本都不是小白了）：

1.下载 python 安装程序，现在的 windows 电脑一般都是 intel 64 位的，所以直接下载 64 位的 python 安装包就可以了，[下载地址](https://www.python.org/ftp/python/3.13.2/python-3.13.2-amd64.exe)：，安装的时候记得要勾选”“Add Python to PATH”“

2.安装完成后打开命令行工具
![](https://www.freecodecamp.org/news/content/images/2022/08/ss5-1.png)

3.打开命令行后输入命令:
```shell
python --version 
```
如果输出了 python 的版本则证明安装成功，这里我们通过命令行向计算机发送了一个非常简单的指令，用于检查 python 的版本，如果 python 没有安装成功的话计算机是找不到 `python3` 这个执行的，以后我们的程序的执行都要通过这个 `python3` 命令来执行。

4.在命令行显示的目录下面新建一个文件 `demo.py`，注意 python 代码文件的后缀名一定要是 `.py`
![](https://img2018.cnblogs.com/i-beta/774327/202002/774327-20200201080302766-2109351422.png)
举一个例子，上面图中的 `D:\pycharm\work\` 就是命令行当前的工作目录，要在这个目录下新建代码文件，要不然 python 找不到文件。

5.在 demo.py 中输入以下代码
```python
print("hello, python")
```
保存，并通过 python 命令来执行
```python
python demo.py
```
结果会在命令行输出 "hello, python"，这种通过在命令行发一条指令给计算机来执行，然后显示结果的交互就是命令行和计算机交互的基本方式，也是使用计算机最开始的交互方式，之不过后来为了方便人们的使用开发了各种图形化界面程序来人们通过点击拖拽的方式来使用计算机，而底层给计算机发指令的事情图形化程序已经代理我们做了。

上面是非常简单的代码，效果是输出一行文本，而一般 python 代码由这些部分组成：
1. 变量定义：把字符串或数字赋值给一个有名字的代称供程序后面来使用
2. 流程控制：if 判断和循环
3. 函数：将一组操作封装在一起作为一个可重复利用的代码块
4. 错误和异常管理：但程序运行报错时如何处理
5. 包管理：代码非常多时如何模块化管理
6. 面向对象：类和对象，可以减少重复代码并且能使代码更加直观
7. 其他：装饰器，lambda 函数啥啥啥的

接下来的学习流程通过例子来学习，可以在 `demo.py` 文件里自己跟着敲代码，然后运行看看输出是什么。

先用一个简单的例子来说明变量定义，流程控制和函数。我们来开发一个程序，用来计算某个人的 BMI（Body Mass Index，身体质量指数）。
```python
# 定义一些变量来使用
PI = 3.14159  # 浮点数，表示圆周率
name = "小明"  # 字符串，表示用户姓名
height = 1.75  # 浮点数，表示身高（单位：米）
weight = 68  # 整数，表示体重（单位：千克）

bmi = weight / (height * height)  # 计算 BMI 值，变量定义了后就可通过它的名字来用

# 流程控制
if bmi < 18.5:
    category = "偏瘦"
elif 18.5 <= bmi < 24.9:
    category = "正常"
elif 24.9 <= bmi < 29.9:
    category = "超重"
else:
    category = "肥胖"

# 函数定义，这个函数要接受3个参数，并将最后的结果返回，通过冒号和缩进来组成一个代码块
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

对程序的解释：
- 以 `#` 符号开头的是代码的注释，用于理解代码，对代码的运行没有影响。
- 代码是一行一行的向下执行，如果我们要将一组代码定义成一个代码块的话可以通过冒号 `:` 和缩进来，比如上面的函数定义部分，注意 python 对代码的缩进要求非常严格，没事不要乱缩进代码，否则会报错。
- 函数用关键字 `def` 来定义，然后依次是函数名，函数要接收的参数，函数内部的操作，最后用关键字 `return` 将结果返回，函数也被称为方法，其实是一样的，只不过叫法不一样。
- `f"{name} 的 BMI 是 {bmi:.2f}，属于 {category}。"` 这个是 python3（python3 以下的版本没有） 用于把变量组合在字符串里的语法，有时候字符串的一部分是不固定的，比如 "姓名：{实际的名字}"，明显这里的实际的名字是一个变量。而 `{bmi:.2f}` 中的 `.2f` 用于控制 bmi 这个浮点数输出取多少小数位，如果要输出 3 个数位的位的话则是 `.3f`。
- 使用函数时通过函数名，把需要的参数传入即可，这里用 result 变量来接受了函数的返回值。
- `print(result)`, 这里的 `print` 是 `python` 自带的函数，就是 `python` 已经事先定义好了直接拿来用就可以，因为输出内容这个功能是个非常常用的功能，从这里也能看出来定义函数的好处，就是每次使用时直接拿来用就可以了，不需要再把函数里的那些操作再写一遍。

再来一个例子来演示 python 中的错误和异常处理，包管理，我们用 python 的文件操作来为例子。
```python
import os  # 导入 os 模块用于文件操作

# 写入文件
def write_file(file_path, content):
    # 将可能会发生错误的代码块放在try里
    try:
	# 打开文件的固定写法，with open(file) as f，将文件打开并用变量f来接收，之后可以用文件对象里定义好的方法来操作文件
        with open(file_path, "w", encoding="utf-8") as f:
            f.write(content)
        print(f"成功写入 {file_path}")
    # 在excpet中处理可能的错误，要不然程序出错了会直接退出，用这个程序的人不知道发生了什么
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
- `import os `，使用 `import` 关键字来导入 os 模块，python 已经将文件的一些通用操作的代码放在了 os 模块里，要用里面的函数直接 import 这个模块就可以了。
- `try except`，文件操作是个会出错的操作，比如文件不存在或者文件损坏了，对于这样可能出错的操作需要将其作放在 try 里面，出错了则在 except 代码中处理，否则程序莫名其妙退出用程序的人不知道发生了什么。
- `with open(file_path, "w", encoding="utf-8") as f:`  python 中打开文件的固定写法。
- `os.remove(file_path)`， 既然导入了 os 模块，就可以直接在使用它的 remove 方法 (函数) 来删除文件。

最后来一个例子来演示 python 中面向对象的使用，首先简单解释下面向对象的概念：
- 程序只是一组数据和对这些数据的操作，我们希望用这些去模拟现实世界以更容易的编程。
- 将一组数据和数据的操作封装在一起，从而实现代码的封装和对现实世界的模拟，比如 “学生“ 这个概念可以由 ”名字，年龄，年级“ 这些数据来描述。
- ”类和对象“ 的概念就是通过将上面的 ”学生“ 定义为一个类别，再用这个类别来定义具体实例，即对象，比如有了学生这个概念后，可以定义一个具体的学生 “小王 (名字: 小王, 年龄: 15,  年级：高一)”。
- “面向对象” 的意思就是在程序中通过定义各种类别，再从这些类别中定义具体的对象，让这对象在程序中 “各种表演”，来实现各种功能。

来个简单的例子说明：
```python
# 1. 定义类 Student
class Student:
	## 2个下划线开始的函数是python中的特殊函数，__init__用于初始化
    def __init__(self, name, age, grade):
        self.name = name
        self.age = age
        self.grade = grade

    def introduce(self):
        return f"我是 {self.name}，{self.age} 岁，就读于 {self.grade} 年级。"

# 2. 从Student类派生子概念（子类）UniversityStudent
class UniversityStudent(Student):
    def __init__(self, name, age, grade, major):
        # super()是python中特殊的一个方法，通过这个调用父类Student中的初始化方法
        super().__init__(name, age, grade)  
        self.major = major

    def introduce(self):
        return f"我是 {self.name}，{self.age} 岁，大学 {self.grade} 年级，主修 {self.major}。"

    def study(self):
        return f"{self.name} 正在学习 {self.major} 相关课程。"

# 3. 创建对象
student1 = Student("小明", 15, 9)
uni_student = UniversityStudent("小红", 20, 2, "计算机科学")

# 4. 调用方法，并用 print 输出
print(student1.introduce())  # 调用父类Student的introduce方法
print(uni_student.introduce())  # 调用子类UniversityStudent的introduce方法
print(uni_student.study())  # 调用子类UniversityStudent独有的study方法
```
代码的解释：
- 利用 class 关键字来定义类，类就是类别的意思，简单而言就是程序对现实世界的抽象，比如这里定义了一个 Student 类
- 类里有一些属性，比如 Student 里的 name, age, grade，在 `__init__` 函数中（这个函数名时固定的，python 中特殊的函数）提供对这些属性的初始化操作。
- 类里除了属性外，也可以定义类自己的一些函数（也被称为方法），比如这里 introduce 函数用于自我介绍
- 定义了类之后，可以派生出子类，比如从 Student 派生出 UniversityStudent，实现了代码复用，大学生里面添加了 major(专业) 属性和 study 方法。
- 定义类后不能直接使用，需要将它实例化为一个具体的对象才能用，`student1 = Student("小明", 15, 9)`。 这个很好理解，因为类描述了一类事物的属性，比如学生，是一个概念性质的东西，现实世界还是以“学生小明”这样的具体事物来运作。
- `def introduce(self)` 这里的 self 的作用是将这个方法挂载在类的实例，也就是对象上，类的对象来使用这个方法，`student1.introduce()`，但是有些方法我们希望可以直接通过类名来用，这个使用定义方法的时候就不需要 `self`，这个以后再说。

入门的话差不多这么多就行，学习的时候不用扣一些细节，主要掌握大致的概念和整体的结构，然后自己跟着敲代码加深下理解。
