# Flask

### 安装Flask

```
pip install flask
```

### run()

```python
app.run(host='127.0.0.1', port=5000, debug=False)
```
- host：指定监听的主机地址。默认

  ```
  127.0.0.1
  ```

  仅允许本机访问；设置为

  ```
  0.0.0.0
  ```
* port：绑定端口号，默认5000。若端口冲突需手动修改（如

  ```
  port=8000
  ```
  
* debug：调试模式开关。

  ```
  debug=True
  ```
  


### debug

我们可以使用debug，方便调试。这样就不用每次运行app.py文件，可以直接修改，不然每次修改app.py都要重新运行。

```python
if __name__ == '__main__':
    app.debug = True
    app.run()
```

```python
    app.run(debug=True) #相同效果。
```

### 装饰器

#### route

使用route（）装饰器告诉Flask什么样的URL能触发我们的函数.route（）装饰器把一个函数绑定到对应的URL上，这句话相当于路由，一个路由跟随一个函数。

```python
@app.route('/')
def test()"
   return 123
```

**设置动态网址**

```python
@app.route("/hello/<name>")
def hello_user(name):
  return "user:%s"%name
```

**转化器**

```python
@app.route('/hello/<int:id>')
def show_post(id):
    return '%d' %id
```

* int    接受整数
* float   接受浮点数
* path  默认

#### methods 设置请求方式

通过`methods`参数定义支持的HTTP方法，默认仅处理GET请求：

```python
@app.route('/submit', methods=['GET', 'POST'])
def submit_form():
    if request.method == 'POST':
        return "表单已提交"
    return "显示表单页面"
```



### 模板渲染

#### render_template()

```python
#app.py
from flask import Flask, render_template
app = Flask(__name__, template_folder='custom_templates')
@app.route('/')
def index():
   user = {'name': 'Admin'}#传入一个字典数组
   return render_template("index.html",title='Home',user=user)

if __name__ == '__main__':
    app.debug = True
    app.run()
```

* **自定义模块路径** ：Flask中 template_folder ，默认路径：templates

```html
index.html
<html>
  <head>
    <title>{{title}}</title>
  </head>
 <body>
      <h1>Hello, {{user.name}}!</h1>
  </body>
</html>
```

#### render_template_string()

```python
def test():
    template = '''
        <div class="center-content error">
            <h1>Oops! That page doesn't exist.</h1>
            <h3>%s</h3>
        </div> 
    ''' %(request.url) #SSTI 注入
```



### request

#### request.url

返回客户端请求的**完整URL**

```python
request.url
#http://localhost:5000/user?id=1
```

#### request.args 

获取**URL查询参数**（即`?key=value`格式），返回`ImmutableMultiDict`对

```python
# 访问 /search?q=flask&name=hhhh
search_term = request.args.get('q')
name = request.args.name
```

#### request.form

获取**表单提交的POST数据**（Content-Type为`application/x-www-form-urlencoded`或`multipart/form-data`）

```python
@app.route('/login', methods=['POST'])
def login():
    username = request.form.get('username') # request.form.username
    password = request.form.get('password')
```

#### request.values

**统一参数获取**
`request.values` 是 `ImmutableMultiDict` 类型的对象，合并了以下两类数据：

- `request.args `通过 URL 查询字符串传递的参数（如 `/submit?name=John&age=20`
- **`request.form`**：通过 POST 请求的表单数据（如 `<form method="post">` 中的输入字段

```python
@app.route('/login')
def login():
    username = request.values.get('username') # request.values.username
```

#### request.json` / `request.get_json()

解析JSON格式的请求体（Content-Type需为`application/json`）

```python
data = request.get_json()
user_id = data.get('user_id')
```

#### request.cookies

