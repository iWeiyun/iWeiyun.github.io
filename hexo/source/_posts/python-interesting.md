---
title: 有趣的python
date: 2018-11-12 22:40:37
author: matthew
tags: python
---





### 前言

python是一门有趣的语言，有些特性在其它语言基本没有。比如代码格式会影响到代码，成员变量的权限是根据名称来决定，类型是全小写等。

但又有很多内容与其它语言相似。比如Generator、async/await，和JS里的相应概念很是相似，类的动态性也不少与js差不多。python类的一些高级特性，也有一些能在OC里找到影子。

python有些写法感觉会比较繁琐，如 && 符号，明明两个字符就能表示“并且”了，但python非要用and这样的单词。

但python又有些写法表达力很强，如lst[1:3]，即可取出子列表，以及列表生成式的表达能力，与haskell里的List Comprehension差不多了。

下面就列出python里一些有趣的点

​	


### 内容

---

#### 极强表达力的列表生成式

```python
r = [x*x for x in range(1, 11)]
print(r)	# 输出：[1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
```



> *在Haskell中称为[List comprehension](https://wiki.haskell.org/List_comprehension)的，写法与其类似：*

```haskell
[x*x | x <- [1..10]]
```



---

#### 类型是小写的

```python
# str即是表示字符串类型
def myFunc(s: str):
    # some code
```

> *参数类型是可选的，一般不需要加*



---

#### 变量不需要指定类型，而且在运行时可以修改类型

```python
v = 1
v = "2"
```

---


#### 与、或操作符，是用and、or来表示的

```python
b1 = True
b2 = False
o1 = b1 and b2	# False
o2 = b1 or b2	# True
```



---

#### 列表可以从后面来访问

```python
lst = [1, 2, 3]
print(lst[-1])	# 输出3, 也要注意不能越界
```

> lst[0:2], 这样来写就可以取出[1, 2]

---

#### 可以执行字符串

```python
exec("print('Hello World')")	# 输出：Hello World
```

> *类似js中的 `eval`*  

---

#### 支持lambda表达式

```python
lambda r, v : r + v
```

----

#### 元组定义时可以不用括号

```python
tuple = "a", "b", "c", "d"
print(tuple[1])	# 输出：b
```

> *元组其实就是不可变的列表*

---

#### 简单的交换变量的方式

```python
x, y = y, x
```


> *类似swift的写法，本质都是利用元组来交换：*

```swift
(x, y) = (y, x)
```

---

#### for-else结构

```python
lst = [1, 3, 5, 7, 9, 13, 19]
for i in lst:
    if i % 2 == 0:
        print("找到了偶数")
        break
else:
    print("没有找到偶数")		# 输出：没有找到偶数
```

----

#### 类的方法，定义时需要显示指定self

```python
class Student(object):
    def set_score(self, score):
        pass

s = Student()
s.set_score(85)	# 调用时不需要指定
```

>  *在其它语言里面基本都是编译器自动帮指定，不需要显示的写出来的。*

---

#### 成员变量要在\__init__里面指定，在方法外定义的是类属性

```python
class Student(object):
    count = 0
    def __init__(self, name, score):
        self.name = name
        self.score = score
        Student.count += 1

m = Student("Matthew", 80)
print(Student.count)	# 1
print(m.name)	# Matthew
```

---

#### 私有变量通过名字来限定

通过在名字前加双下划线，来表示是私有变量。

```python
class Student(object):
    def __init__(self, name, score):
        self.__name = name
        self.__score = score

s = Student("Matthew", 60)
print(s.__name)	# 'Student' object has no attribute '__name'
```

----

#### 类可以动态的增、删成员变量

```python
class Student(object):
    pass

o = Student()
o.name = "Matthew"
print(o.name)	# 输出: Matthew

del o.name
print(o.name)	# 'Student' object has no attribute 'name'
```

> *js也有这个能力*

---

#### 调用不存在的属性，可以被开发者接管

```python
class Student(object):
    def __getattr__(self, attr):
        if attr == 'name':
            return "Good"

o = Student()
print(o.name)	# 输出：Good
```

> *跟OC的转发找不到的方法非常像。包括  `__str__` 方法，也非常类似OC中的 `description` ，来实现自定义打印内容*

----

#### 可以通过代码来设置断点

```python
pdb.set_trace()
```

> *类似js的 `debugger` 语句*

---

​	



### 参考资料

[廖雪峰的官方网站](https://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000)