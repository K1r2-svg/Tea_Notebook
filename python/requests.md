# requests库使用

### 导入库

```python
import requests
```
### 基本用法
```python

headers={
    "Host":"xxxx" #更改host头
}
parmas={ #添加GET请求参数
    "a":"xxx"
}
x=requests.get(url,parmas=parmas,headers=headers) #发送get请求,
print(info.status_code) #打印状态码
print(x.text) #打印返回内容
```