# Dirsearch 使用

### 基本参数

* `-u`  指定url

  ```
   dirsearch -u http://example.com/ 
  ```

* `-e`  指定扩展

  ```
   dirsearch -e php,js,html -u http://example.com/ 
  ```

* `-r`  递归扫描

  ```
   dirsearch -u http://example.com/ -r 
  ```

* `-o`  保存文件

* `--format` 保存文件的格式

  * xml
  * json
  * ... ...

  ```
   dirsearch -u http://example.com/ -o 1.json --format json 
  ```

