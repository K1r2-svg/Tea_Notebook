# c++基础
### 数据类
#### 保留小数点
```C++
#include <iostream>
#include <iomanip>

int main()
{
	double a = 1.2345;
	std::cout << std::setprecision(3) << a << std::endl;
}
```

### #include的知识

#### `#include " "`和`#include <>`的区别

##### 使用 #include " "

* **查找路径**
  编译器会首先在当前源文件所在的目录及其子目录中查找指定的头文件。如果找不到，才会去系统默认的头文件路径中查找。

* **适用场景**
  通常用于自定义的头文件或项目内部的头文件。这样可以确保编译器优先查找本地项目中的文件，避免与系统库中的同名文件冲突。

##### 使用 `#include <>`

- **查找路径**

  编译器只会去系统默认的头文件路径中查找指定的头文件。这些路径通常是安装开发工具链时设置好的标准库路径。

- **适用场景**

  通常用于标准库或第三方库的头文件。这样可以确保编译器从系统的标准库路径中获取正确的头文件。

#### 具体例子
```C++
#include "GLFW/glfw3.h"
```
这行代码告诉编译器先在当前项目的目录中查找`GLFW/glfw3.h`，如果找不到再去找系统路径。

```C++
#include <GLFW/glfw3.h>
```
这行代码直接告诉编译器去系统路径中查找`GLFW/glfw3.h`。

### 内联函数

#### 简述

内联函数，提高程序性能，减少函数调用开销，直接将函数代码嵌入调用处，避免函数调用时压栈的操作，节省时间。
是否编译成内联函数，编译器决定，否则，编译后还是函数的调用。
```C++
inline int Cuboid_Vol(int x,int y,int z)
{ return x * y * z; }
int main()
{
	int vol = Cuboid_Vol(1,2,3);
}
```



### 三元表达式

#### 用法

```c++
int a;
int n;
a = n > 4 ? 4 : 10;
    
a = n > 5 ? (n > 4 ? 4 : 5) : 4;

a = n > 5 && n < 10 ? 6 : 5; 
```



### 数组

#### 声明数组
```C++
int a[5]; //在栈上创建数组
int *ptr = new int[5]; //在堆上创建数组
delete[] ptr //释放堆上的数组

*(ptr + 2) = 2 <--> ptr[2] = 2 //(ptr+2)==(char *)ptr + 2*sizeof(int)
```

#### 在类中声明数组

```C++
class A
{
public:
	static const int size = 5; //类中常数必须是静态的
	int a[size];
	int b[5];
	int* c = new int[5];
}
```
#### std::array
##### 用法
```C++
#include <array>
std::array<int,5> a;
int size = a.size();
```

#### 数组操作函数

一下函数区间是[a,b)

#### min_element

```c++
int a[4] = { 2,3,4,1 };
auto b = std::max_element(a, a+3);
std::cout << *b << std::endl;
```

#### max_element

```c++
int a[4] = { 2,3,4,1 };
auto b = std::max_element(a, a+3);
std::cout << *b << std::endl;
```

#### find

```C++
std::vector<int> v = { 5, 3, 9, 7, 2 ,1};
int target = 2;

auto it = std::find(v.begin(), v.end(), target);

if (it != v.end()) {
    std::cout << "找到 " << target << "，位置：" << std::distance(v.begin(), it);
}
else {
    std::cout << "未找到 " << target;
}
```

### 指针

#### 声明

```C++
int *a;//声明指针
int *a, *b; //声明一行指针
int *a, b;//这样只声明了a是指针
```

### 函数指针

#### 声明

```c++
void PrintValue(int value)
{
	std::cout << value << std::endl;
}
int main()
{
    void(*Print)(int) = PrintValue;
    using void(*Print)(int);
    Print a = PrintValue;
}
```
### lambda
#### 用法
```C++
#include <iostream>
#include <unordered_map>
#include <functional>
void ForEach(std::function<void()>& fun)
{
	fun();
}
int main()
{    
	int a = 0;
	auto b = 2;
	auto lam = [&]()
	{a = 2;std::cout << a << std::endl;}; //按引用捕获所有外部变量
	//修改外部的a
	auto lam = [=]() mutable 
	{a = 3 std::cout << a << std::endl;}; //按值捕获所有外部变量
	//修改内部的a
	auto lam = []() { std::cout << "hhh" << std::endl;};
	ForEach(lam);
}
```

### 智能指针

#### unique_ptr

不能将它复制给别人

```C++
class Entity{}
int main()
{
    {
    	std::unique_ptr<Entity> entity=std::make_unique<Entity>();//出作用域死亡
        entity->Print();
    }
}
```

#### shared_ptr

​	可以共享的指针，有一个计数器，计数器为0后销毁。创建时，先创建对象，再创建一个计数器。每赋值一个计数器+1。不能用两个 shared_ptr 对象托管同一个指针。如果有两个shared_ptr相互引用，那么这两shared_ptr指针的[引用计数](https://zhida.zhihu.com/search?content_id=169332346&content_type=Article&match_order=1&q=引用计数&zhida_source=entity)永远不会下降为0

```C++
class Entity{}
int main()
{
    {
        std::shared_ptr<Entity> e;
        {
            std::shared_ptr<Entity> entity=std::make_shared<Entity>();
            e=entity 
        }
        //当所有引用死了后才释放。
    }
        
}
```

#### weak_ptr

* weak_ptr和shared_ptr可以相互转化，shared_ptr可以直接赋值给weak_ptr，weak_ptr也可以通过调用lock函数来获得shared_ptr。

* weak_ptr指针通常不单独使用，只能和 shared_ptr 类型指针搭配使用。将一个weak_ptr绑定到一个shared_ptr不会改变shared_ptr的引用计数

* weak_ptr并没有重载operator->和operator *操作符，因此不可直接通过weak_ptr使用对象，典型的用法是调用其lock函数来获得shared_ptr示例，进而访问原始对象。

  ```C++
  class B;
  class A
  {
  public:
      weak_ptr<B> pb_weak;
      ~A()
      {
          cout << "A delete\n";
      }
  };
  class B
  {
  public:
      shared_ptr<A> pa_;
      ~B()
      {
          cout << "B delete\n";
      }
      void print() {
          cout << "This is B" << endl;
      }
  };
  
  void fun() {
      shared_ptr<B> pb(new B());
      cout << "pb.use_count " << pb.use_count() << endl;//1
      shared_ptr<A> pa(new A());
      cout << "pa.use_count " << pa.use_count() << endl;//1
  
      pb->pa_ = pa;
      cout << "pb.use_count " << pb.use_count() << endl;//1
      cout << "pa.use_count " << pa.use_count() << endl;//2
  
      pa->pb_weak = pb;
      cout << "pb.use_count " << pb.use_count() << endl;//1：弱引用不会增加所指资源的引用计数use_count()的值
      cout << "pa.use_count " << pa.use_count() << endl;//2
  
      shared_ptr<B> p = pa->pb_weak.lock();
      p->print();//不能通过weak_ptr直接访问对象的方法，须先转化为shared_ptr
      cout << "pb.use_count " << pb.use_count() << endl;//2
      cout << "pa.use_count " << pa.use_count() << endl;//2
  }//函数结束时,调用A和B的析构函数
  
  //资源B的引用计数一直就只有1，当pb析构时，B的计数减一，变为0，B得到释放，
  //B释放的同时也会使A的计数减一，同时pa自己析构时也会使资源A的计数减一，那么A的计数为0，A得到释放。
  int main()
  {
      fun();
  }
  ```

### 默认参数

#### 简述

```c++
void printArea(int value = 2) //当没传值的时候使用默认值。
{ }
```

#### 实例

```c++
void printArea(int value = 2,int a=2);
int main()
{ ... }
void printArea(int value,int a)
{ ... }
```



#### 注意

```c++
void printArea(int a1,int a2 = 1) //正确
void printArea(int value = 2,int a1) //错误
```



### auto

#### 简述

自动判断数据类型

#### 用法

```C++
auto a = 34;
```

#### 注意

不能滥用，在返回类型很长，不能判断返回类型的时候用。

### C++异常类

#### 

### 字符串

#### 用法
```C++
const char* str1 = "deadbeef";//声明常量字符串
char str2[] = "deadbeef"; //声明可修改字符串

std::string a="dead"; //使用string类声明
a += "beef"; //这里可以这样，+=被string类重载了，调用了std::string 的成员函数 operator+=

std::string a=std::string("dead") + "beef"; //string对象的实列附加"beef".

```


#### 错误用法

```C++
std::string a="dead" + "beef"; //这个是错误的，这里两个字符串其实是两个指针，不能将指针相加
```


#### string类型

```c++
std::string a = "beef"
std::wstring a = "beef"
std::u32string a = "beef"
```

#### std::string中的一些方法

##### .data

```C++
string s = "hhhhh";
s.data() //返回s中的字符串的地址。
```

##### 函数中引用
```C++
void PrintString(std::string str)//这样会将传入的str复制一遍，不会改变原来的值
//复制是很慢的，拖慢速度。
{
	std::cout << str << std::endl;
}
void PrintString(std::string& str)//这样会引用，会在函数内改变原来的值
//这样快
{
	std::cout << str << std::endl;
}
//推荐做法
void PrintString(const std::string& str)//这样说明是个常量，并且不可更改，还快
{
	std::cout << str << std::endl;
}

```


##### .find()

```C++
std::string str = "deadbeef";
str.find("beef") //返回beef的起始位置 返回std::string::npos表示没找到
```

##### 字符串拼接

```c++
std::string str= std::string("adf") + "gggg"//方法一
std::string str="gggg\n"
    "aaaa\n"
    "ggggg\n";
std::string a = //使用R
R"(Hello
hellpsdfs
hhh)";
```

##### compare比较

`compare` 方法通过返回值指示字符串的字典顺序，具体规则如下：

- **返回 0**：两字符串相等；
- **返回负数**：调用字符串小于参数字符串；
- ** 返回正数**：调用字符串大于参数字符

###### 主要重载形式及用途：

1. **比较整个字符串**
   语法：`int compare(const string& str) const`
   示例：

   cpp

   复制

   ```cpp
   std::string s1 = "apple", s2 = "banana";
   int result = s1.compare(s2); // 返回负数（"apple" < "banana"）
   ```

2. **比较子字符串**
   语法：`int compare(size_t pos1, size_t len1, const string& str)`
   示例：

   cpp

   复制

   ```cpp
   std::string s = "hello world";
   int res = s.compare(6, 5, "world"); // 比较 "world" 与参数字符串，返回 0[3,6](@ref)
   ```

3. **跨字符串部分比较**
   语法：`int compare(size_t pos1, size_t len1, const string& str, size_t pos2, size_t len2)`
   示例：

   cpp

   复制

   ```cpp
   std::string s1 = "programming", s2 = "grammar";
   int res = s1.compare(3, 5, s2, 0, 5); // 比较 "gramm"（s1[3:8]）与 "gramm"（s2[0:5]）
   ```

4. **与 C 风格字符串比较**
   语法：`int compare(const char* s) const`
   示例：

   cpp

   复制

   ```cpp
   std::string s = "C++";
   int res = s.compare("C"); // 返回正数（"C++" > "C"）[1,5](@ref)
   ```

##### append() 添加

##### assign() 赋值

```C++
int main()
{
	std::string s = "Hello";
	std::cout << s << std::endl;
	std::string s1;
	std::cout << s1.assign(s,1,3) << std::endl;
    // s1 : ell
}
```

##### compare() 比较

##### substr()

##### insert()

#### std::string的优化

##### 使用std::string_view

每次使用std::string都会使用malloc分配内存。使用substr等一些方法，也会调用malloc,拖慢速度，所以一些情况下使用std::string_view优化代码.
**实列：**
```C++
#include <iostream>
#include <future>
#include <mutex>


#define VIWE 0
static int count = 0;
void* operator new(size_t size) 
{
    std::cout << "Malloc size: " << size << std::endl;
    count++;
    return malloc(size);
}
#if VIWE
void PrintName(std::string_view& name)
{
    std::cout << name << std::endl;
}
#else
void PrintName(std::string& name)
{
    std::cout << name << std::endl;
}
#endif // VIWE
int main() {
    std::string name = "Liu JiaYi";
#if VIEW
    std::string_view lastName(name.c_str(),3);
    std::string_view firstName(name.c_str()+4, name.size()-1);
#else
    std::string lastName = name.substr(0, 3);
    std::string firstName = name.substr(4, name.size() - 1);
#endif
    PrintName(lastName);
    PrintName(firstName);
    std::cout << "Malloc count: " << count << std::endl;
    return 0;
}
```

#### stringstream

```C++
#include <sstream>
#include <string>

int main()
{
	std::string s = "Hello";
	std::stringstream s1;
	s1 << 1.234;
	s1 >> s;
	std::cout << s << std::endl;
}
```

#### .getline



### 引用

#### 特性

	不占用内存,相当于别名，不是真正变量。相当于const的指针，初始化后不可更改。



#### 用法

```C++
int a=5;
int& r=a;引用//
r=6 //--> a==6
//声明必须传值
int &ret=a;
//函数传值

void fun(int &b){
	b=8;
}
int main(){
	int a=1;
	fun(a);
	printf(a);
	//--> a==8s;
}
```
初始化后不能再赋值别的变量
```C++
int &a=b;
a=c;//这是错误的
```
这是不对的



#### 引用本质

```c++
//引用的本质 是常量指针
int a = 10;
int &ptr=a;  //--> int * const ptr = &a;
int &a <--> int * const a;
a=20; //内部自动转换为 *a=20;
```

### 关于static

####  只在当前文件中使用

```c++
static void function();
static int a;
//其他文件就无法访问
```

### extern 外部声明

```C++
exetern int a; //访问外部变量
exetern void function(); //访问外部变量
```



### 关于CONST

#### 实例

``` C++
int a=10;
int b=11;
const int* ptr1=5; //只能改变指针指向的值
ptr1 = &a //能这样，不能 *ptr1=2;
    
int const* ptr2=&a; //不能改变指针,但能改变指针的值
*ptr2 = 11; //但不能 ptr=&b;

const int const* ptr3=&a //既不能修改指针，也不能修改指向的值
```

### constexpr

#### 简述

**`constexpr`** 是C++11引入的关键字，旨在进一步强化常量的概念，尤其是针对那些需要在编译期完全确定的值。它不仅限于变量，还可用于函数和类的构造函数

constexpr适用于需要在编译期完全确定的场景，如数组维度、模板参数、编译期计算逻辑等，这有助于减少运行时开销，提升程序效率。

### 命名空间

#### 简述
	用来解决命名重复的冲突问题


#### 用法

创建
``` C++
namespace n1
{
	int a=1;	
}
int a=0;//全局
//这两个a互不干扰
```
使用作用域限定符 :: 进行访问 
```C++
namespace n1
{
	int a=1;	
}
int a=5;
int main()
{
	int b=::a;//::用来访问全局域
	int c=n1::a//用来访问n1中的a
}
```
#### using namespace全部展开 
使用using namespace 将命名空间名称全部展开
```C++
namespace n1
{
	int f=1;
}
using namespace n1;
int main()
{
	f++;
	int a=f;
}
```
#### using部分展开
使用using部分展开
```C++
namespace n1
{
	int f=1;
	int a=2;
}
using n1::f;
```
### 申请释放堆内存

#### new

使用`new` 申请堆内存,调用`new`后，内部会调用`malloc`。`new`后一定用`delete`删除

```C++
class A ...
    
int *a = new int[50];
A e = new A;
```

#### delete

使用`delete`释放堆内存，内部会调用`free()`	

```c++
class A ...
    
int *a = new int[50];
A e = new A;

delete[] a; //释放数组
delete e;   //释放对象
```

### 枚举enum

#### 简述
	每个枚举常量可以用一个标识符来表示，也可以为它们指定一个整数值，如果没有指定，那么默认从 0 开始递增,在类中属于类的命名空间。
#### 用法
```C++
enum Level
{
	n1,n2,n3
};
//声明
Level n=n1; //必须是枚举类中的常量。
```
#### 类中用法
```C++
class Log
{
public:
	enum Level
	{
		Err,War,Info
	};
}
int main(){		
	Log log;
//调用
	Log::Err;
	Log::War;
}
```

### 类

#### 简述
	类中变量函数默认private
#### 类与结构体的区别
	类和结构体在技术上没有什么不同，在类中默认是Private，在结构体中默认public，同样可以在结构体中写函数，在结构体中没有继承的关系。
#### 创建对象
```C++
class A
{
public:
	int x;
	A(){
		std::cout << "Hello
	}
}

int main()
{
	A sub;//在栈上创建
	A* sub=new A();//在堆上创建
}
```
#### 销毁对象

在堆上的对象使用delete删除。

```C++
class A
{
public:
	int x;
	A(){
		std::cout << "Hello
	}
}

int main()
{
	A sub1;//在栈上创建
	A* sub2=new A();//在堆上创建
	delete sub2;
}
```

#### 访问在栈上的对象

```C++
class A
{
public:
	int x;
	A(){
		std::cout << "Hello
	}
}

int main()
{
	A sub;//在栈上创建
	sub.x=2;
}
```

#### 访问在堆上的对象

```C++
class A
{
public:
	int x;
	A(){
		std::cout << "Hello
	}
}

int main()
{
	A* sub = new A();//在栈上创建
	sub->x=2;
}
```

#### 类中的static

* static方法不能访问非static变量。
* 类中的static成员变量，需要在类外定义。
```C++
class Entity
{
	static int x,y;
}
int Entity::x; //类中静态变量需要初始化，才能用
int Entity::y;
//调用
int main()
{
	Entity::x = 2;
}
```
* 改变类中的某个static变量，所有该类的实列中的那个static变量都会改变。
* class中的static变量，相当于是类中的共享数据，像末影箱。
* 不实例化对象，也可使用静态变量和方法。
#### 类中的常量
	在类中常量必须是静态的。
#### 单例创建

##### 使用静态创建
```C++
#include <iostream>

class Random
{
public:
    // 禁用拷贝构造函数
    Random(const Random&) = delete;

    // 公有成员变量
    float Rand = 0.5f;

    // 静态成员函数，用于获取单例实例
    static Random& Get()
    {
        return s_Instance;
    }

private:
    // 私有构造函数，确保只能通过 Get() 方法获取实例
    Random() {}

    // 私有成员变量
    float m_Rand = 0.5f;

    // 静态成员变量的声明
    static Random s_Instance;
};

// 静态成员变量的定义，为其分配内存空间
Random Random::s_Instance;

int main()
{
    // 通过静态成员函数获取单例实例
    Random& random = Random::Get();
    std::cout << "Rand: " << random.Rand << std::endl;
    return 0;
}
```

#### 构造函数

##### 简述 
	可以使用构造函数来初始化类，构造函数与类名相同，不实例化类无法使用。
##### 用法
```C++
class Log
{
public:
	int X,Y;

	Log(int x,int y)
	{
		X=x;
		Y=y;
	}
}
int main()
{
	Log log(1,2); //当声明时自动调用构造函数。
}
```
##### 初始化传值方法
```C
class Log
{
private:
	int m_X,m_Y;
public:
    Player(int x,int y)  
        : m_X(x),m_Y(y) {}
}
```
##### 禁止使用构造函数
方法一：
```C++
class Log
{
private:
	Log() {}
}
```
方法二：
```C++
class Log
{
public:
	Log() = delete;
}
```

#### 析构函数
##### 简述
	在对象销毁时即对象所在的栈或堆被销毁，或所在作用域结束时调用。
##### 用法
```C++
class A
{
	~A() //析构函数
	{
		std::cout << "Bay" << std::endl;
	}
}
```
#### 类中调用类外函数

```c++
//类中调用类外函数，需要在类的上方声明函数，或将函数写类的上方。若函数中引用了类，需要在函数上方定义类。
```

##### 实例

```C++
#include <iostream>

class Vector;
void PrintVector(const Vector& v);
class Vector
{
public:
	float X, Y, Z;
	Vector(float x, float y, float z)
		:X(x), Y(y), Z(z) 
	{
		PrintVector(*this);
	}
};
void PrintVector(const Vector& v)
{
	std::cout << v.X << ", " << v.Y << ", " << v.Z << std::endl;
}
```



#### 初始化列表

##### 简述

	调用类中方法时，用来初始类中变量。
##### 用法
```C++
class Entity
{
private:
	int m_X,m_Y;
	std::string m_name;
public:
	Entity()
		:m_Y(0),m_Y(2)
	{
		std::cout << x << ',' << y << std::endl;
	}
	
	Entity(const std::string& name)
	{
		m_name= name; //这样会创建两个std::string,消耗性能。
		std::cout << m__name << std::endl;
	}
	
	//正确做法
	Entity(const std::string& name)
		:m_name(name) //这里不会 
	{
		std::cout << name << std::endl;
	}
	
	
}
```
##### 对性能的优化
实例：
```C++
class Exc
{
public:
	Exc() 
	{
		std::cout << "hhh" << std::endl;
	}

	Exc(int x)
	{
		std::cout << "hhh " << x << std::endl;
	}
};
class Entity
{
private:
	Exc ex;
public:
	Entity()
		:ex(Exc(8)) //这里只会创建一次,还可以 ex(8)
	{
	}
};

class Entity2
{
private:
	Exc ex;
public:
	Entity2()
	{
		ex = EXc(8);//这里只会创建两次次
	}
};
```


#### 继承

##### 简述
	类的继承，已有的类称为基类，新建的类称为派生类。
	子类能继承父类的所用东西，但子类不能访问父类的私有变量和方法，只能去传入父类的地方，可以传入子类。
##### 用法
```C++
class A //基类
{
public:
	int a,b;
}
class B : pulibc A //继承A,派生类
{
	int c
}
```
##### 访问控制
|访问 |	public |protected|private|
|---|---|---|---|
|同一类 | T | T | T |
|派生类 | T | T | F |
|外部类 | T | F | F|
##### 继承类型
```
class A {}
class B : public A {}
```
* pulibc 
		基类的公有成员，成为子类的公有;基类的保护成员，成为子类的保护;基类的私有成员，子类不可访问，但可以通过基类公有或保护成员访问。
* protected
		基类的公有成员、保护成员，成为子类的保护的成员。
* private
		基类的公有成员、保护成员，成为子类类的私有成员

#### 虚函数
* 虚函数是实现多态性的基础。当一个类中声明了虚函数，编译器会为这个类创建一个虚函数表。
* 多态性允许通过基类的指针或引用调用派生类中重写的函数，根据对象的实际类型而不是指针或引用的类型来决定调用哪个函数。
* 
运用：
```C++
class Base {
public:
    virtual void print() { //告诉编译器生成虚表，声明虚函数关键字 virtual
        std::cout << "Base class" << std::endl;
    }
};
class Derived : public Base {
public:
    void print() override { //覆写函数关键字 override 
        std::cout << "Derived class" << std::endl;
    }
};
int main() {
    Derived d;
    // 向上转型，将派生类对象的引用转换为基类对象的引用
    Base& b_ref = d;
    // 向上转型，将派生类对象的指针转换为基类对象的指针
    Base* b_ptr = &d;
    b_ref.print(); // 调用 Derived 类的 print 方法
    b_ptr->print(); // 调用 Derived 类的 print 方法
    return 0;
}
```
##### 向上转型
* 向上转型是向上转型是指将派生类（子类）的指针或引用转换为基类（父类）的指针或引用的过程。

* 向上转型是隐式的

* 通过向上转型，我们可以调用派生类中重写的函数。

  

##### 向下转型

* 在继承体系里，基类通常是派生类的抽象概括，
* 目的是为了能访问派生类中特有的成员和功能。
* 使用dynamic_cast实现安全的向下转型

##### 非虚函数

但没有指出虚函数时，向上转型后，将调用基类的方法。
```C++
class Base {
public:
    void print() { //告诉编译器生成虚表，声明虚函数关键字 virtual
        std::cout << "Base class" << std::endl;
    }
};
class Derived : public Base {
public:
    void print(){ //覆写函数关键字 override 
        std::cout << "Derived class" << std::endl;
    }
};
int main() {
    Derived d;
    // 向上转型，将派生类对象的引用转换为基类对象的引用
    Base& b_ref = d;
    // 向上转型，将派生类对象的指针转换为基类对象的指针
    Base* b_ptr = &d;
    b_ref.print(); // 调用 Base 类的 print 方法
    b_ptr->print(); // 调用 Base 类的 print 方法
    return 0;
}
```
##### 纯虚拟表virtual
* 纯虚拟表是在基类中只声明而没有实现的虚函数，相当于接口，使用 “ =0 ”标记。
* 含有纯虚函数的类是抽象类，不能被实例化。
* 为纯虚拟表的子类时，必须实现这些纯虚函数，才能被实列化。
纯虚函数：
```C++
class A //纯虚拟表
{
public:
	virtual std::string GetClassName() = 0;
}

class B : public A
{
public:
	std::string GetClassName() override {return "B";} //这样B才能实例化
}
```
##### 虚析构函数

在使用向上转型的时候，会调用基类和派生类的构造函数，但销毁后不会调用派生类的析构函数。

所以要在基类的析构函数前加上virtual.

```C++
#include <iostream>
#include <vector>
#include <functional>
#include <algorithm>

class Parent
{
public:
	int b;
	Parent()
	{
		std::cout << "Parent start\n";
	}
	virtual ~Parent() 
	{ 
		std::cout << "Parent end\n";
	}
};

class chlid : public Parent
{
public:
	int a;
	chlid()
	{
		std::cout << "chlid start\n";
	}
	~chlid()
	{
		std::cout << "chlid end\n";
	}
};
int main()
{	
	Parent* b = new chlid();
	delete b;
}
```



#### 类中的const

##### 实例

```C++
class A
{
private:
	int m_X,m_Y;
    mutable int a;//mutable允许在常量方法中修改的变量。
public:
    int x,y;
	int GetX() const //意味着不能更改类中的变量
	{
		m_X = 5; //这是不行的，会报错，使用mutable声明变量可修改
        a = 5; //这样不会报错
		return m_X;
	}
	int GetY() 
	//意味着不能更改类中的变量,返回不可以修改的指针，和不可修改指针的值。
	{
		return m_Y;
	}
}
//在方法中调用
//const A& a 和 A& a的区别
//在const类中，只能调用const的方法，保证不修改数据。
void PrintString(const A& a)
{
    //带const的，不允许外部修改类中的变量和调用非const的function。
	a.GetX(); //正确的
    a.GetY(); //这是不允许的，GetY 不是const的function。
    a.x=8; //这是不允许的，不允许外部修改const类中的变量
}
int main()
{
    const A a; //同上函数中的特性。
}
```

#### 类中隐性转换

##### 实例

隐性转换只能转换一次，

```C++
#include <iostream>

class Entity
{
private:
	std::string m_Name;
	int m_Age;
public:
	Entity(const std::string& name)
		:m_Name(name) {}

	Entity(int age)
		:m_Age(age) {}
};

int main() 
{
	Entity e1 = std::string("hhhh");// std::string --> Entity
    Entity e1 = "hhhh"; //这是不行的 const char[] --> std::string --> Entity
	Entity e2 = 22;	   //int --> Entity
}
```

#### explicit关键字禁止隐性转换

有时禁止隐性转换后，可以改善性能和避免bug。

##### 实例

```c++
class Entity
{
private:
	std::string m_Name;
	int m_Age;
public:
	explicit Entity(const std::string& name) //意味着你必须明确调用这个构造函数来创建对象。
		:m_Name(name) {}

	explicit Entity(int age)
		:m_Age(age) {}
};
void printValue(const Entity& obj) {
    ... ...
}

int main() {
    // 正确：显式构造
    MyClass obj1(42);
    printValue(obj1);

    // 错误：隐式构造（不允许）
    // printValue(42);  // 这行会导致编译错误

    // 正确：显式类型转换
    printValue(static_cast<Entity>(42));
    Entity e("hhhh");
    Entity e(22);
}
```

#### 运算符与重载operator

##### 用法

使用 `operator` 重载运算符。

##### 实例1_重载 * + ==

```C++
#include <iostream>

class Vector
{
public:
	float X, Y, Z;
	Vector(float x, float y, float z)
		:X(x), Y(y), Z(z) {
	}
	Vector Add(const Vector& v) const
	{
		return Vector(v.X + X, v.Y + Y, v.Z + Z);
	}
	Vector operator+(const Vector& v) const //重载运算符+
	{
		return Add(v);
	}

	Vector mul(const Vector& v) const
	{
		return Vector(v.X * X, v.Y * Y, v.Z * Z);
	}

	Vector operator*(const Vector& v) const //重载运算符*
	{
		return mul(v);
	} 
	bool operator==(const Vector& v) //重载运算符==
	{
		return v.X == X && v.Y == Y && v.Z == Z;
	}

	void PrintVector()
	{
		std::cout << X << ", " << Y << ", " << Z << std::endl;
	}
};

std::ostream& operator<<(std::ostream& stream,const Vector& v) 
{
	stream << v.X << ", " << v.Y << ", " << v.Z;
	return stream;
}

int main() 
{
	Vector v1(1.0f, 3.0f, 3.0f);
	Vector v2(1.0f, 2.0f, 3.0f);
	Vector v3 = v1 + v2 * v2;
	if (v1 == v2) 
	{
		std::cout << "Yes" << std::endl;
	}
	else
	{
		std::cout << "No" << std::endl;
	}
	std::cout << v1 << std::endl;
}
```
##### 实例2_重载 [] <<
```C++
#include <iostream>

class String
{
private:
	char* m_Str;
	unsigned int m_Size;
public:
	String(const char* str) 
	{
		m_Size = strlen(str);
		m_Str = new char[m_Size + 1];
		memcpy(m_Str,str,m_Size + 1);
		m_Str[m_Size] = '\0';
	}
	String(const String& string)
		:m_Size(string.m_Size)
	{
		std::cout << "hhh" << std::endl ;
		m_Str = new char[m_Size + 1];
		memcpy(m_Str,string.m_Str,m_Size+1);
	}
	~String()
	{
		delete[] m_Str;
	}
	
	char& operator[](unsigned int index)
	{
		return m_Str[index];
	}
	friend std::ostream& operator<<(std::ostream& stream, const String& str)
};

void Print(const String& str) 
{
	std::cout << str << std::endl;
}
std::ostream& operator<<(std::ostream& stream,const String& str)
{
	stream << str.m_Str;
	return stream;
}

int main() 
{
	String a = "LIU";
	String str = a;
	str[2] = 'L';
	Print(a);
	Print(str);
}
```
##### 实例3_重载 ->

```C++
#include <iostream>

class Entity
{
public:
	void Print()
	{
		std::cout << "Hello！" << std::endl;
	}
};

class Set 
{
private:
	Entity* m_Entity;
public:
	Set(Entity* e)
		:m_Entity(e)
	{}
	Entity* GetEntity()
	{
		return m_Entity;
	}
	Entity* operator->()
	{
		return m_Entity;
	}
};

int main() 
{
	Set e = new Entity();
	e->Print(); //等价于 e.operator->()->Print()
}
```

### 线程

#### std::thread

适用于需要对线程进行精细控制的场景，例如需要手动管理线程的生命周期、控制线程的启动和停止时间，或者需要创建多个线程并进行复杂的线程间同步和通信。

```C++
#include <iostream>
#include <thread>
int flag = 0;
void DoWork()
{
	while(!flag) {
		std::cout << "working... ..." << std::endl;
	}
}
int main()
{    
	std::thread worker(DoWork);
	std::cin.get();
	flag = 1;
	worker.join(); //等待线程完成。
	std::cin.get();
}
```

#### std::async

适用于简单的异步任务，尤其是只需要获取任务结果而不需要过多关注线程管理的场景。它简化了异步编程的过程，让代码更加简洁。例如，在需要异步执行一些计算任务并获取结果时，使用 `std::async` 会更加方便。

##### 参数说明

- `policy`

  ：可选参数，指定任务的执行策略，有以下几种取值：

  - `std::launch::async`：异步执行任务，即会创建一个新的线程来立即执行函数 `f`。
  - `std::launch::deferred`：延迟执行任务，直到调用 `std::future` 对象的 `get()` 或 `wait()` 方法时才会执行函数 `f`，并且是在调用这些方法的线程中执行。
  - `std::launch::async | std::launch::deferred`：默认策略，具体执行方式由实现决定，可能是异步执行，也可能是延迟执行。

- **`f`**：要执行的可调用对象，如函数、函数指针、成员函数指针、lambda 表达式等。

- **`args`**：传递给可调用对象 `f` 的参数。

##### 注意

函数参数，不能是引用。它需要确保传递的参数在新线程执行时是有效的。如果直接传递引用，当调用 std::async 的线程结束时，引用的对象可能已经被销毁，这样在新线程中使用该引用就会导致未定义行为。

##### 用法

```C++
#include <iostream>
#include <future>
#include <mutex>


std::mutex mut;
int calculate(int* acc) {
    std::lock_guard<std::mutex> lock(mut); //进程锁
    for (int i = 0; i < 100;i++) {
        (*acc)++;
    }
    return *acc;
}
int main() {
    int a = 0;
    for (int i = 0; i < 2;i++) {
        std::future<int> vulae = std::async(std::launch::async,calculate,&a);
        std::cout << vulae.get() << std::endl;
    }
    
    return 0;
}
```

##### std::future

std::async 会返回一个 std::future 对象，通过调用 std::future 的 get() 方法可以方便地获取任务的返回值。如果任务抛出异常，get() 方法会重新抛出该异常。

#### std::mutex 互斥锁类

`std::mutex`是 C++ 标准库中用于实现线程同步的互斥锁类，它位于 `<mutex>` 头文件中。互斥锁是一种同步原语，用于保护共享资源，防止多个线程同时访问和修改这些资源，从而避免数据竞争和不一致的问题。

##### 互斥锁的锁定和解锁

使用`lock() `取互斥锁，`unlock()`释放互斥锁，如果互斥锁已经被其他线程持有，当前线程会被阻塞，直到互斥锁被释放。

```C++
#include <iostream>
#include <thread>
#include <mutex>
std::mutex mtx;
int sharedResource = 0;
void increment() {
    // 锁定互斥锁
    mtx.lock();
    for (int i = 0; i < 100000; ++i) {
        ++sharedResource;
    }
    // 释放互斥锁
    mtx.unlock();
}
int main() {
    std::thread t1(increment);
    std::thread t2(increment);
    t1.join();
    t2.join();
    std::cout << "The value of sharedResource is: " << sharedResource << std::endl;
    return 0;
}
```



##### 使用 std::lock_guard 自动管理互斥锁

手动调用 lock() 和 unlock() 容易出现忘记解锁的情况，导致死锁。std::lock_guard 是一个 RAII（资源获取即初始化）类模板，它可以在构造时自动锁定互斥锁，在析构时自动释放互斥锁，避免了手动管理锁的复杂性。其模板参数必须是互斥锁类型。

```C++
#include <iostream>
#include <thread>
#include <mutex>
std::mutex mtx;
int sharedResource = 0;
void increment() {
    // 使用 std::lock_guard 自动管理互斥锁
    std::lock_guard<std::mutex> lock(mtx); //出作用域后调用析构函数释放
    for (int i = 0; i < 100000; ++i) {
        ++sharedResource;
    }
}
int main() {
    std::thread t1(increment);
    std::thread t2(increment);
    t1.join();
    t2.join();
    std::cout << "The value of sharedResource is: " << sharedResource << std::endl;
    return 0;
}
```



### 程序计时

#### **`std::this_thread::sleep_for`**

```C++
#include <iostream>
#include <thread>  // 包含 sleep_for
#include <chrono>  // 包含时间单位

int main() {
    std::cout << "Start..." << std::endl;
    std::this_thread::sleep_for(std::chrono::seconds(1)); // 延时 1 秒
    std::cout << "1 second later..." << std::endl;
    return 0;
}
```

支持的时间单位：

* std::chrono::nanoseconds（纳秒）
* std::chrono::microseconds（微秒）
* std::chrono::milliseconds（毫秒）
* std::chrono::seconds（秒）
* std::chrono::minutes（分钟）
* std::chrono::hours（小时）

#### 计时

```c++
#include <iostream>
#include <unordered_map>
#include <functional>
#include <thread>

struct Timer
{
	std::chrono::time_point<std::chrono::steady_clock> start,end; 	  				     	//std::chrono::time_point 类型，用于存储时间点
	std::chrono::duration<float> duration;
    //用来表示以浮点数形式存储的秒级时间间隔
	Timer()
	{
		start = std::chrono::high_resolution_clock::now();// 获取开始时间点
	}
	~Timer()
	{
		end = std::chrono::high_resolution_clock::now();// 获取结束时间点
		duration = end - start;
		std::cout << duration.count() * 1000.0f << "ms\n"; //duration.count()返回秒数
	}
};

void DoWork()
{
	Timer timer;
	for (int i = 0;i < 100;i++) {
		std::cout << "working... ..." << std::endl;
	}
}
int main()
{    
	
	std::thread worker(DoWork);
	worker.join();
	std::cout << "OK!" << std::endl;
	std::cin.get();
}
```

#### 性能测试例子

```c++
#include <iostream>
#include <chrono>
#include <array>
class Timer
{
public:
    Timer()
    {
        startTimer = std::chrono::high_resolution_clock::now();
    }
    ~Timer()
    {
        Stop();
    }
    void Stop() 
    {
        auto endTimer = std::chrono::high_resolution_clock::now();
        auto start = std::chrono::time_point_cast<std::chrono::microseconds>(startTimer).time_since_epoch().count();
        auto end = std::chrono::time_point_cast<std::chrono::microseconds>(endTimer).time_since_epoch().count();
        auto duration = end - start;
        double ms = duration * 0.001;
        std::cout << ms << std::endl;
    }
private:
    std::chrono::time_point<std::chrono::steady_clock> startTimer;
};
int main() 
{
    {
        Timer timer;
        std::array<std::shared_ptr<int>, 1000> a;
        for (int i = 0; i < 1000;i++)
            a[i] = std::shared_ptr<int>(new int);
    }
    {
        Timer timer;
        std::array<std::shared_ptr<int>, 1000> a;
        for (int i = 0; i < 1000;i++)
            a[i] = std::make_shared<int>();
    }
    {
        Timer timer;
        std::array<std::unique_ptr<int>, 1000> a;
        for (int i = 0; i < 1000;i++)
            a[i] = std::make_unique<int>();
    }
}
```



### mutable关键字

##### 简述

​	可以用于在类中的const的方法中修改变量。

##### 用法

```C++
class A
{
private:
    mutable int a;//mutable允许在常量方法中修改的变量。
    int b;
public:
   	int GetX() const //意味着不能更改类中的变量
	{
        b = 4; //这样会报错
        a = 5; //这样不会报错
		return a;
    }
}
```

### 结构化绑定，返回多类型

适用函数返回多类型

```C++
std::tuple<std::string, int> Crate() 
{
    return {"Liu" , 23};
}

std::tuple<std::string, int> Crate()
{
    return { "Liu" , 23 };
}
int main() 
{
    //三种方法
    //一
    auto preson = Crate();
    std::string name1 = std::get<0>(preson);
    int age1 = std::get<1>(preson);
	//二
    std::string name;
    int age;
    std::tie(name,age) = Crate();
	//三
    auto [name, age] = Crate();
}
```

### 文件操作

#### 操作文件三大类

* ofsteam 写
* isteam 读
* fsteam  读写

#### 打开模式

#####  1. `std::ios::in`

- **含义**：以输入（读取）模式打开文件。用于 `std::ifstream` 类时，该模式是默认模式；对于 `std::fstream` 类，需要显式指定。
#####  2. std::ios::out
含义：以输出（写入）模式打开文件。如果文件不存在，则创建该文件；如果文件已存在，则清空文件内容。对于 std::ofstream 类，该模式是默认模式。
这些模式可以通过按位或运算符 `|` 组合使用。
##### 3. std::ios::app
含义：以追加模式打开文件，新写入的内容会添加到文件的末尾，而不会清空原有内容。该模式只能与 std::ios::out 一起使用。
##### 4. std::ios::ate
含义：打开文件后，将文件指针定位到文件末尾。但后续的写入操作可以覆盖原有内容，而不是追加。该模式可以与其他模式组合使用。
##### 5. std::ios::trunc
含义：如果文件已存在，打开文件时会清空文件内容。通常与 std::ios::out 一起使用，并且当指定 std::ios::out 时，std::ios::trunc 是默认行为。
##### 6. std::ios::binary
含义：以二进制模式打开文件，而不是文本模式。在二进制模式下，文件的读写操作不会对换行符等特殊字符进行转换，适用于处理二进制数据，如图片、音频等。
#### 写文件

```c++
#include <iostream>
#include <fstream>
#include <string>

int main() {
    // 创建一个 ofstream 对象并打开文件
    std::ofstream outFile("example.txt");

    // 检查文件是否成功打开
    if (outFile.is_open()) {
        // 写入文本内容
        outFile << "这是第一行文本。\n";
        outFile << "这是第二行文本。\n";

        // 关闭文件
        outFile.close();
        std::cout << "文件写入成功。" << std::endl;
    } else {
        std::cerr << "无法打开文件。" << std::endl;
    }

    return 0;
}
```



#### 读文件

##### 核心代码

```c++
#include <iostream>
#include <fstream>
#include <string>

int main() {
    // 创建一个 ifstream 对象并打开文件
    std::ifstream inFile("example.txt");

    // 检查文件是否成功打开
    if (inFile.is_open()) {
        std::string line;
        // 逐行读取文件内容
        while (std::getline(inFile, line)) {
            std::cout << line << std::endl;
        }

        // 关闭文件
        inFile.close();
    } else {
        std::cerr << "无法打开文件。" << std::endl;
    }

    return 0;
}
```

#### 读写文件

```C++
#include <iostream>
#include <fstream>
#include <string>
int main() {
    std::string filepath = "example.txt";
    // 创建一个输入文件流对象并尝试打开指定文件
    std::ifstream stream(filepath);
    if (stream.is_open()) {
        std::string line;
        // 逐行读取文件内容
        while (std::getline(stream, line)) {
            std::cout << line << std::endl;
        }
        // 关闭文件流
        stream.close();
    } else {
        std::cerr << "Unable to open file: " << filepath << std::endl;
    }
    return 0;
}
```
#### 二进制读写
##### 写入
```C++
#include <iostream>
#include <fstream>

int main() {
    int data = 123;
    std::ofstream outFile("binary.bin", std::ios::binary);

    if (outFile.is_open()) {
        outFile.write(reinterpret_cast<const char*>(&data), sizeof(data));
        outFile.close();
        std::cout << "二进制文件写入成功。" << std::endl;
    } else {
        std::cerr << "无法打开文件。" << std::endl;
    }

    return 0;
}
```
##### 读
```C++
#include <iostream>
#include <fstream>

int main() {
    int data;
    std::ifstream inFile("binary.bin", std::ios::binary);

    if (inFile.is_open()) {
        inFile.read(reinterpret_cast<char*>(&data), sizeof(data));
        inFile.close();
        std::cout << "读取的二进制数据: " << data << std::endl;
    } else {
        std::cerr << "无法打开文件。" << std::endl;
    }

    return 0;
}
```
### optional

#### 简述

std::optional：它是 C++17 引入的一个模板类，位于 <optional> 头文件中。std::optional 用于表示一个对象可能包含一个值，也可能不包含值（即处于 “空” 状态）。这种设计避免了使用指针和空指针（nullptr）来表示缺失值可能带来的空指针解引用风险，提供了一种更安全、更清晰的方式来处理可能缺失值的情况。

#### 用法

```C++
#include <iostream>
#include <string>
#include <tuple>
#include <optional>
#include <fstream>
std::optional<std::string> ReadFile(const std::string filepath)
{
    std::ifstream stream(filepath);
    if (stream) 
    {
        std::string content;
        stream.close();
        return content;
    }
    return std::string();
}
int main() 
{
    auto content = ReadFile("data");
    //判断为空指针，并输出错误信息的方法。
    std::string vulae= content.value_or("Not Found File"); 
    //value_or(default_value)：返回 std::optional<std::string> 对象中的字符串值，如果对象为空，则返回 default_value。
    std::cout << vulae;
	//判断为空指针，并输出错误信息的方法。
    if (content.has_value()) //has_value()：检查 std::optional<std::string> 对象是否包含值。
    {
        std::cout << "read succeed\n";
    }else{
        std::cout << "Not";
    }
    //value()：返回 std::optional<std::string> 对象中的字符串值，如果对象为空，会抛出 std::bad_optional_access 异常。

}
```



### 取偏移地址

```C++
struct vector
{
	int x, y, z;
};
int main() 
{
	int offest = (int)&((vector*)nullptr)->y;
	std::cout << offest;
}
```
### 断言

#### 静态断言

##### static_assert

```C++
//static_assert(常量表达式, 错误信息字符串);
static_assert(sizeof(void*)==4,"64位系统以上支持")
```



### 模板

#### 用法
```C++
template<typename T> //定义模板, typename 定义数据类型
void Print(T a)
{
	std::cout << a << std::endl;
}
template<typename T,int N>
class Arr
{
private:
	T m[N];
public:
	int GetSize() { return N; }
};
int main()
{
	Arr<int,6> a;
	int b = a.GetSize();
	Print<int>(b);
}
```
### 联合体

#### 简述

几个不同名的变量公用一块内存

#### 用法

```C++
struct Union
{
	union
	{
		int a;
		float b;
		//a和b公用内存，改变b就是改变a，改变a就是改变b。
	};
};
```

#### 类型双关

##### 另一种方式

```C++
int a;
float b = *(float *)&a;
```

### 类型转换

#### C 风格类型转换

#### static_cast<>()

static_cast 可以在不同的基本数据类型（如整数、浮点数等）之间进行转换。这种转换可能会导致数据截断或精度损失。

##### 用法

```C++
#include <iostream>

int main() {
    double d = 3.14;
    // 将 double 类型转换为 int 类型
    int i = static_cast<int>(d); 
    std::cout << "Converted value: " << i << std::endl;
    return 0;
}
```

#### dynamic_cast<>()

主要用于在继承体系中进行安全的向下转型（将基类指针或引用转换为派生类指针或引用）。它会在运行时进行类型检查，如果转换不安全，对于指针类型会返回 nullptr，对于引用类型会抛出 std::bad_cast 异常。

##### 用法

```C++
#include <iostream>

class Base {
public:
    virtual void print() {
        std::cout << "Base" << std::endl;
    }
};

class Derived : public Base {
public:
    void print() override {
        std::cout << "Derived" << std::endl;
    }
};

int main() {
    Base* basePtr = new Derived();
    // 安全的向下转型
    Derived* derivedPtr = dynamic_cast<Derived*>(basePtr);
    if (derivedPtr) {
        derivedPtr->print();
    }
    delete basePtr;
    return 0;
}
```

#### const_cast<>()

用于去除或添加对象的 const 或 volatile 限定符。需要注意的是，它不能用于改变对象的类型，只能改变其常量性或易变性。

```C++
#include <iostream>

void printValue(int* num) {
    std::cout << *num << std::endl;
}

int main() {
    const int num = 10;
    // 使用 const_cast 去除 const 限定符
    int* nonConstNum = const_cast<int*>(&num);
    // 这里修改值是未定义行为，因为 num 本身是 const 的
    // *nonConstNum = 20; 
    printValue(nonConstNum);
    return 0;
}
```

#### `reinterpret_cast`

用于进行最底层的类型转换，它可以将一种类型的指针或引用转换为另一种完全不相关的类型的指针或引用，甚至可以将指针转换为整数类型.

##### 用法

```C++
#include <iostream>

int main() {
    int num = 10;
    // 将 int 指针转换为 double 指针
    double* Ptr = reinterpret_cast<double*>(&num); 
    //相当于 double * ptr = *(double*)&num
    std::cout << *Ptr << std::endl;
    return 0;
}
```

### 左值和右值

```C++
void PrintName(const std::string& name) //可以接受右值引用和左值应用,但不可修改
{
	std::cout << "Name is: " << name << std::endl;
}
void PrintName(std::string&& name) //接收只左值引用
{
	name[0] = 'a';
	std::cout << "Name is: " << name << std::endl;
}
void PrintName(std::string&& name) //接收只右值引用
{
	name[0] = 'a';
	std::cout << "Name is: " << name << std::endl;
}

int main()
{
	std::string a = "Liu"; 
	std::string b = "Jia Yi"; //"Jia Yi" 右值
	std::string c = a + b; // c 为左值 ， a+b 为右值、
	PrintName(a);
	PrintName(a + b);
}
```

### 容器

#### std::array

创建静态数组。

```c++
int main()
{
	std::array<int,4> arr;
	int size = arr.size();
    arr[0]=12;
}
```

在栈上。
#### std::vector

创建动态数组。

```C++
#include <vector>

int main()
{
	std::vector<int> a; //创建一个int的动态数组。
    a.push_back(2);		//添加元素
    a.push_back(3);
    a.erase(a.begin() + 1); // 删除索引为1的元素
    for(int& v : a) //遍历元素
    {}
}
```

vector的优化

在每次push_back时，vector都会重新开辟更大的内存存放新的东西，并把之前东西cp进新的内存。

使用构造函数能明白这一点。这样会减慢运行速度，所以我们使用emplace_back,告诉它，在实际vector的内存中，使用这些参数，创建对象。再使用reserve规定vector容器的大小。这样每添加一个元素，都不会重新申请一块元素。

见以下代码：

```c++
#include <iostream>
#include <vector>

struct ver
{
	float x, y, z;
	ver(float x,float y,float z)
		:x(x),y(y),z(z)
	{}
	ver(const ver& v)
		:x(v.x),y(v.y),z(v.z)
	{
		std::cout << "hhh" << std::endl;
	}
};

int main() 
{
	std::vector<ver> a;
	a.reserve(3);
	a.emplace_back(1,2,3);
	a.push_back({ 3,4,5 });
	a.push_back({ 3,4,5 });
}
```

#### std::unordered_map
##### 用法
###### 初始化
```C++
#include <unordered_map>
int main
{
	std::unordered_map<std::string,int> myMap;
    //插入元素
    std::unordered_map<std::string, int> myMap = {
    {"apple", 1},
    {"banana", 2},
    {"cherry", 3}
	};
    myMap.insert({"date", 4}); //insert插入元素
    myMap["elderberry"] = 5; //使用下标
    
    //at查看元素，是否存在，不存在抛出异常
    try {
        int value = myMap.at("banana");
        cout << "Value of banana: " << value << endl;
    } catch (const out_of_range& e) {
        cout << "Key not found: " << e.what() << endl;
    }
    
    //find查找是否存在
    auto it = myMap.find("cherry");
    if (it != myMap.end()) {
        cout << "Key cherry found, value: " << it->second << endl;
    } else {
        cout << "Key cherry not found." << endl;
    }
    
    //通过键来删除
    myMap.erase("date");
    
    //便利
    for (const auto& pair : myMap) {
    	cout << "Key: " << pair.first << ", Value: " << pair.second << endl;
	}
    
    for (auto it = myMap.begin(); it != myMap.end(); ++it) {
    	cout << "Key: " << it->first << ", Value: " << it->second << endl;
	}
}
```


#### std::unordered_set

#### std::find_if
`std::find_if `是 C++ 标准库中的一个算法，用于在范围内查找满足特定条件的第一个元素。

它的参数包括：

* a.begin()：指向容器 a 的起始位置。

* a.end()：指向容器 a 的结束位置。

* [](int value) { return value > 3; }：一个 lambda 表达式，用于定义查找条件。

```C++
#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    std::vector<int> a = {1, 2, 3, 4, 5};

    auto b = std::find_if(a.begin(), a.end(), [](int value) { return value > 3; });

    if (b != a.end()) {
        std::cout << *b << std::endl;  // 使用 *b 访问元素
    } else {
        std::cout << "No element found" << std::endl;
    }

    return 0;
}
```

### 排序

#### std::sort

在<algorithm>头中。

```C++
#include <iostream>
#include <vector>
#include <functional>
#include <algorithm>
int main()
{		
	std::vector<int> vs = { 4,3,2,6,1,5 };
	std::sort(vs.begin(), vs.end(), [](int a,int b) 
		{
			return a < b; // 返回True，则 a 排在 b之前。
		});
	for (int v : vs) 
	{
		std::cout << v << std::endl;
	}
}
```

### std::accumulate

`std::accumulate` 是 C++ 标准库 `<numeric>` 头文件中提供的一个通用算法，用于对指定范围内的元素进行累加操作，它的功能不仅仅局限于简单的数值求和，还可以根据自定义的操作规则对元素进行聚合计算。

#### 用法

```c++
#include <iostream>
#include <numeric>
#include <vector>
int main() {
    std::vector<int> numbers = {1, 2, 3, 4, 5};
    // 自定义操作，计算元素的乘积，初始值为 1
    int product = std::accumulate(numbers.begin(), numbers.end(), 1, [](int acc, int num) {
        return acc * num; //acc 累加器，这里为1，num 是numbers.begin(), numbers.end()每个元素
    });
    std::cout << "Product: " << product << std::endl;
    return 0;
}
```



###  std::variant

####  简述

std::variant 是 C++17 标准库中引入的一个非常有用的类模板，它位于 <variant> 头文件中。std::variant 可以用来表示一个类型安全的联合体（Union），它能够存储多种不同类型的值，但在同一时间只能存储其中一种类型的值。

#### 用法

```C++
#include <variant>
int main() 
{
	std::variant<std::string, int> data; //定义变量 
	data = "Cherno";
	std::cout << std::get<std::string>(data)<<"\n";
    //std::get<>() 获得data值
	std::cout << "Index: " << data.index() << std::endl;
    //data.index() 获得当前类型索引
	if (auto* vulae = std::get_if<std::string>(&data))
    //std::get_if<>()判断是否是指定类型，不是返回空指针。
	{
		std::string *a = vulae;
	}
	else
	{ 
		std::cout << "err\n";
	}
	data.index();
	data = 23;
	data.index();
	std::cout << data[1] << "\n";
	
}
```

### std::any

#### 简述

可以存放任意类型数据，不建议使用。

### 调试方法

#### 跟踪内存分配情况

```c++
struct  AlloctionMetrics
{
	uint32_t TotalAllocated = 0;
	uint32_t TotalFreed = 0;
	
	uint32_t CurrentUsage() { return TotalAllocated - TotalFreed; }
};

static AlloctionMetrics s_AllocationMetrics;

void PrintMemoryMessage()
{
	std::cout << "Memory Usage: " << s_AllocationMetrics.CurrentUsage() << std::endl;
}
void* operator new(size_t size)
{
	s_AllocationMetrics.TotalAllocated += size;
	return malloc(size);
}
void operator delete(void* memory,size_t size)
{
	s_AllocationMetrics.TotalFreed += size;
	free(memory);
}
```

### 移动语句

#### std::move

##### 简述



##### 实列

```C++
#include <iostream>
#include <functional>
#include <string>
#include <memory>

class String
{
public:
	String() = default;
	String(const char* string)
	{
		std::cout << "created" << std::endl;
		m_size = sizeof(string);
		m_str = new char[m_size];
		memcpy(m_str, string, m_size);
	}
	String(const String& string)
	{
		std::cout << "cp" << std::endl;
		m_size = string.m_size;
		m_str = new char[m_size];
		memcpy(m_str, string.m_str, m_size);
	}
	String(String&& string) noexcept
	{
		std::cout << "mov" << std::endl;
		m_size = string.m_size;
		m_str = string.m_str;

		string.m_size = 0;
		string.m_str = nullptr;
	}
    String& operator=(String&& other) noexcept //移动赋值
    {
        std::cout << "oper_mov" << std::endl;
        if (this != &other) 
        {
            delete[] m_str;
            m_size = other.m_size;
            m_str = other.m_str;

            other.m_size = 0;
            other.m_str = nullptr;
        }

        return *this;
    }
	~String()
	{
		std::cout << "delete" << std::endl;
		delete[] m_str;
	}
	void Print()
	{
		std::cout << m_str << std::endl;
	}
	
private:
	size_t m_size;
	char* m_str;
};
class Entity
{
public:
	Entity(const String& name)
		:m_name(name)
	{
	}
	Entity(String&& name)
		:m_name(std::move(name))
	{
	}
	void PrintName()
	{
		m_name.Print();
	}
private:
	String m_name;
};
int main()
{
	Entity en("hhhh");
}
```

**代码的问题**

* `Entity en("hhh")`怎么运行的

  1. 首先编译器将"hhh" const char* 类型转换成 Entity构造函数需要的类型String
  2. 然后调用 String(const char * string) 构造函数
  3. 因为"hhh" 是右值调用Entity(String&& name)构造函数
  4. 然后执行m_name(std::move(name)) 
  5. 如果这里没加std::move 根据上下文name是一个左值，所以会调用String(const String& string)构造函数
  6. 所以要加上std::move 表示name是马上销毁的对象，变右值，从而调用String(String&& string)。

* 为什么要加`noexcept`
	
  * 标准库中的许多容器和算法在设计时都依赖于移动构造函数的 noexcept 特性。例如，std::vector 的 resize、reserve 等函数在重新分配内存时，会检查元素的移动构造函数是否为 noexcept。如果是，就会使用移动操作；否则，会使用拷贝操作。
  * 通过将移动构造函数标记为 noexcept，可以向标准库容器保证移动操作不会抛出异常.
  * 如果移动构造函数被标记为 `noexcept`，容器在进行这些操作时会优先选择使用移动构造函数而不是拷贝构造函数。
  
* 
### std::forword
std::forward 是 C++ 标准库中的一个模板函数，它位于 <utility> 头文件中，主要用于实现完美转发（Perfect Forwarding）。完美转发允许函数模板将其参数以原始的左值或右值属性传递给其他函数，避免不必要的对象复制或移动，从而提高程序的性能。

std::forward 的核心作用是根据传入的模板参数 T 来决定返回左值引用还是右值引用。当 T 被推导为左值引用类型时，std::forward 返回左值引用；当 T 被推导为右值引用类型时，std::forward 返回右值引用。
```C++
#include <iostream>
#include <utility>

// 接收左值的函数
void printValue(int& value) {
    std::cout << "Lvalue: " << value << std::endl;
}

// 接收右值的函数
void printValue(int&& value) {
    std::cout << "Rvalue: " << value << std::endl;
}

// 转发函数模板
template<typename T>
void forwardValue(T&& arg) {
    printValue(std::forward<T>(arg));
}

int main() {
    int x = 10;

    // 传递左值
    forwardValue(x);

    // 传递右值
    forwardValue(20);

    return 0
```

### 

