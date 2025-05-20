## 类

### 继承和抽象类
#### 继承
```python
class Vector():
	def ...

class Vec2(Vector):#继承Vector
	def ...
```
#### 抽象类
```python
from abc import ABC,abstractmethod
class Vector(ABC): #继承抽象基类
	@abstractmethod #定义抽象方法
	def ...
```
**注意：**
ABC 类是 abc 模块中定义的抽象基类标识，继承它的类才能通过 @abstractmethod 装饰器实现抽象方法的强制约束。如果改为 Vector()，则 Vector 变为普通类，其中的 @abstractmethod 装饰器将失效。此时：

* 子类可以不实现​ pay 方法；
* 即使子类未实现抽象方法，也不会在实例化时报错。

#### 动态实例化

```python
class Parent:
    def create_new(self):
        return self.__class__()  # 动态实例化
```

## 基本函数使用

### 数据比较

#### `isclose`

`isclose()`用于**判断两个浮点数是否在指定的误差范围内“接近相等”**。由于浮点数在计算机中的存储和运算可能存在精度误差，直接使用 `a == b` 比较可能不可靠，`isclose()` 通过设置相对或绝对容差（允许的误差范围）来解决这一问题。

```python
import math
f1 = 1.2233222
f2 = 1.2233223
isclose(f1,f2)
```

### 列表、元组、集合的处理

#### `set`

#### 使用set创建不重复元素的集合

```python
basket = ['apple', 'orange', 'apple', 'pear', 'orange', 'banana']
print(set(basket))
#{'orange', 'banana', 'pear', 'apple'}

a = set('abracadabra')
print(a)
#{'a', 'r', 'b', 'c', 'd'}
```

#### `list`

`list()`函数可以将任何可迭代数据转换为列表类型，并返回转换后的列表。当参数为空时，list函数可以创建一个空列表。

1. 创建一个空列表（无参调用list函数）
```
>>> test = list()
>>> test
>>> []
```
2. 将字符串转换为列表
```
>>> >>> test = list('cat')
>>> test
>>> ['c', 'a', 't']
```
3. 将元组转换为列表
```
>>> >>> a_tuple = ('I love Python.', 'I also love HTML.')
>>> test = list(a_tuple)
>>> test
>>> ['I love Python.', 'I also love HTML.']
```
4. 将字典转换为列表
```python
>>> a_dict = {'China':'Beijing', 'Russia':'Moscow'}
>>> test = list(a_dict)
>>> test
>>> ['China', 'Russia']
>>> ⚠️注意：将字典转换为列表时，会将字典的值舍去，而仅仅将字典的键转换为列表。如果想将字典的值全部转换为列表，可以考虑使用字典方法dict.values()
```
5. 将集合转换为列表
```
>>> a_set = {1, 4, 'sdf'}
>>> test = list(a_set)
>>> test
>>> [1, 'sdf', 4]
```
6. 将其他可迭代序列转化为列表
下面的代码将range类型和map类型的可迭代序列转换为列表：
```
>>> test1 = list(range(10))
>>> test2 = list(map(int, [23.2, 33.1]))
>>> test1
>>> [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
>>> test2
>>> [23, 33]
```

### `zip`

**特性**：将对象中对应的元素打包成一个个元组，然后返回由这些元组组成的列表。如果多个给定两个不同的长度的列表会截断较长的那个。

```python
>>> a = [1,2,3]
>>> b = [4,5,6]
>>> c = [4,5,6,7,8]
>>> zipped = zip(a,b)     # 打包为元组的列表
[(1, 4), (2, 5), (3, 6)]
```



### `tuple`

### `map`

map函数会根据提供的函数对指定序列做映射，其的第一个参数 function 以参数序列中的每一个元素调用 function 函数，返回包含每次 function 函数返回值的新列表。

#### 语法

map() 函数语法：

```
map(function, iterable, ...)
```

#### 返回值

Python 2.x 返回列表。

Python 3.x 返回迭代器。

**实例** ：

```python
def square(x) :            # 计算平方数
	return x ** 2
map(square, [1,2,3,4,5])   # 计算列表各个元素的平方
#[1, 4, 9, 16, 25]
map(lambda x: x ** 2, [1, 2, 3, 4, 5])  # 使用 lambda 匿名函数
#[1, 4, 9, 16, 25]

# 提供了两个列表，对相同位置的列表数据进行相加
map(lambda x, y: x + y, [1, 3, 5, 7, 9], [2, 4, 6, 8, 10])
#[3, 7, 11, 15, 19]
```

### `random`

```python
import random
random_number = random.random() #返回一个0.0 - 1.0之间的小数
random.randint(a,b) #返回一个a 和 b 之间的整数（包括 a 和 b）。
```



### `NumPy`库

```python
import numpy as np 
```

#### 计算点积`dot()`

```
import numpy as np

# 一维数组点积
a = np.array([1, 2, 3])
b = np.array([4, 5, 6])
print(np.dot(a, b)) # 输出: 32

# 二维数组（矩阵）点积
A = np.array([[1, 2], [3, 4]])
B = np.array([[5, 6], [7, 8]])
print(np.dot(A, B))
# 输出:
# [[19 22]
# [43 50]]
```

### Hypothesis

进行基于属性测试。
