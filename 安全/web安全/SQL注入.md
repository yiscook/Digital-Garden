---
title: SQL注入
tags:
  - web安全
creation date: 2024-03-17
done: true
---
# 1. MySQL基础  
---
## 1.1 数据类型  
![MySQL 数据类型](https://image.yiscook.top/blog-image/202403181234895.png)

## 1.2 基础语法  
- 查看所有数据库：`show databases;`  
- 创建数据库：`create database <database-name>;`  
- 删除数据库：`drop database <database-name>;`  
- 选择数据库：`use <database-name>;`  
- 查看数据库中的表：`show tables [from <table-name>];`   
- 查看数据库表结构：`desc <table-name>;` → `show columns from <table-name>`  
- 增：`insert into <table-name> (columns ……) values (values ……);`  
- 删：`delete from <table-name> where <条件>;`  
- 改：`update <table-name> set <column-name> = <value> [where <条件>];`  
- 查：`select * / <column-name> from <table-name> [where <条件>];`  
- 排序：`SELECT fieldN... FROM table_nameN... ORDER BY fieldN [ASC [DESC][默认 ASC]];`  
    - `order by <number>`，可以在 order by 后面添加数字，可以用于判断表有多少字段。当数字大于列数的时候会报错。  
- 控制输出：`SELECT columnN... FROM <table-name> LIMIT offset , count;`  
    - `offset`：指定要返回的偏移量。第一行的偏移量为`0`，而不是`1`。  
    - `count`：指定要返回的最大行数。  
    - 也可以仅跟一个数字`N`，表示返回`N`行  
- 注释方法：  
    - `/**/`，可用于多行注释  
    - `--`，仅注释该行字符后面的内容。必须跟空格。需要注意分号`;`，需要在`--`之前，或者在下一行。  
    - `#`，仅注释该行字符后面的内容。  
    - `;%00`，同上。  
    - `/*!*/`，内联注释，会执行注释内的代码。  
- 字符串连接：  
    - `concat(str1, str2, …)`：将多个字符串连成一个字符串。  
    - `concat_ws(separator, str1, str2, …)`：将多个字符串连成一个字符串，可以指定分隔符。  
    - `group_concat([distinct] 字段名 [order by 排序字段 asc/desc] [separator '分隔符'])`：将 group by 产生的同一个分组中的值连接起来，返回一个字符串，可以指定分隔符。  
- `union`：用于组合两个或多个 `SELECT` 语句的结果集。  
    - `UNION` 中的每个 `SELECT` 语句必须具有相同的列数  
    - 列还必须具有相似的数据类型  
    - 每个 `SELECT` 语句中的列也必须是相同的顺序  
    - `UNION` 运算符默认只选择不同的值。 要允许重复值，请使用 `UNION ALL`。  
- 字符的替换：  
    - tab（水平）：`%09`，tab（垂直）：`%0b`  
    - 新建一行：`%0a`  
    - 新的一页：`%0c`  
    - 空格：`%a0`  
- 返回字符串长度：`length(str)`  
- 截取字符串：  
    - `SUBSTR(string, start, length)`；`SUBSTR()`和 `MID()`函数等于 `SUBSTRING()` 函数。  
    - `left(str,length)`、`right(str,length)`：获取指定字符串**左**侧/**右**侧部分的指定长度  
- 编码解码类：  
    - `ascii(str)`、`ord(str)`：返回字符串最左侧字符的ASCII码  
    - `char(number)`：将ASCII码转换为对应字符。  
- 字符串反转：`reverse(str)`  
  
## 1.3 信息收集  
- 版本信息：`version()` → `@@version`  
- datadir，`@@datadir`：MySQL 数据文件存储的目录路径，也就是数据库文件的存储路径。  
- basedir，`@@basedir`：MySQL 的安装目录路径，也就是 MySQL 二进制文件的存储路径。  
- 操作系统信息：`@@version_compile_os`  
- 当前 MySQL 实例连接的 MySQL 用户名：`session_user()`  
- 当前 MySQL 实例连接的操作系统用户名：`system_user()`  
- 当前连接的数据库：`database()`  
  
## 1.4 information_schema  
用于存储关于数据库结构的元数据信息的数据库。它包含了关于数据库中各种对象（如表、列、约束、索引等）的信息，以及用户、权限、存储引擎、字符集、分区等方面的信息。常用的表如下：  
- 数据库信息：`schemata`  
- 所有数据库的表信息：`tables`  
- 所有数据库的表的列信息：`columns`  
![information_schema](https://image.yiscook.top/blog-image/202403181559975.png)

### 1.4.1 其他查询用法  
- 查询当前所在的数据库：`select database();`  
- 查询当前数据库的表名：`select table_name from information_schema.tables where table_schema=database();`  
- 查询表中的列名：`select columns_name from information_schema.columns where table_name=<table-name> [and table_schema=database()];`  
成功获取到了数据库的表名和列名，就可以正常构造语句了。  
  
# 2. SQL注入  
---
攻击者利用输入的数据未经过滤或转义，拼接到 SQL 语句中，从而改变 SQL 语句的含义，导致数据库执行非预期的操作。  
## 2.1 危害  
1. 绕过登录验证：可以尝试使用万能密码登录网站后台。  
2. 数据泄露：可以尝试访问和获取应用程序中的敏感信息，如：用户名、密码等。  
3. 服务器攻击：执行任意命令，对文件系统进行操作，操作注册表等。  
  
## 2.2 注入类型的判断  
| **$id**   | **数字型**           | **字符型**             | **字符型**                 |
| --------- | ----------------- | ------------------- | ----------------------- |
| 1 and 1=1 | id=1 and 1=1 返回正常 | id='1 and 1=1' 返回正常 | id='1' and '1'='1' 返回正常 |
| 1 and 1=2 | id=1 and 1=2 返回错误 | id='1 and 1=2' 返回正常 | id='1' and '1'='2' 返回错误 |
| 1 or 1=1  | id=1 or 1=1 返回正常  | id='1 or 1=1' 返回正常  | id='1' or '1'='1' 返回正常  |
| 1 or 1=2  | id=1 or 1=2 返回正常  | id='1 or 1=2' 返回正常  | id='1' or '1'='2' 返回正常  |
| 1'        | id=1' 返回错误        | id='1'' 返回错误        | id='1'' 返回错误            |
| 1' --+    | id=1' --+ 返回错误    | id='1' --+' 返回正常    | id='1' --+' 返回正常        |
  
## 2.3 手动注入基本步骤  
1. 寻找注入点；  
2. 判断参数类型，以及闭合符号；  
3. 判断字段个数；`order by <idnumber>`；  
4. 判断回显位；`?id=-1 union select 1,2,3`  
5. 利用注入查询基本信息。  
  
## 2.4 注入方法  
### 2.4.1 堆叠注入  
Stacked injections（堆叠注），一堆 sql 语句(多条)一起执行。`;` 是用来表示一条 sql 语句的结束。叠注入可以执行的是任意的语句.  
局限性：可能受到 API或者数据库引擎不支持的限制。  
  
### 2.4.2 报错注入  
人为去制造错误条件，使得查询结果能够出现在错误信息中。  
适用于有报错回显，无法使用 union，但是没有数据回显的情况。  
以下所有方法都需要注意 MySQL 的版本  
  
#### 2.4.2.1 `floor(number)`数学函数，用于向下取整。  
- `x`代表的是：`concat(user(),floor(rand(0)*2))`的别名  
- `a`代表的是：`select count(*),concat(<SQL>,floor(rand(0)*2))x from information_schema.tables group by x`的别名  
- `<SQL>`：可以替换为各种需要获取的信息语句  
```MySQL 
select * from test where id=1 and (select 1 from (select count(*),concat(<SQL>,floor(rand(0)*2))x from information_schema.tables group by x)a);  
```
#### 2.4.2.2 `extractvalue(xml_document, xpath_expr)`：用于提取 XML 文档中的数据。  
- 因 xpath 表达式错误而报错，但是会执行其中的 SQL 语句。  
- `xml_document`：xml 文档。  
- `xpath_expr`：XPath 表达式，用于指定要提取的节点。  
- `<SQL>`：可以替换为各种需要获取的信息语句  
```MySQL
select * from test where id=1 and (extractvalue(1,concat(0x7e,(<SQL>),0x7e)));    
```
#### 2.4.2.3 `updatexml(xml_target, xpath_expr, new_val)`：用于在 XML 数据中更新或插入值。  
- 因 xpath 表达式错误而报错，但是会执行其中的 SQL 语句。  
- `xml_target`：要更新的 XML 数据。  
- `xpath_expr`：要更新的节点路径。  
- `new_val`：新值。  
- `<SQL>`：可以替换为各种需要获取的信息语句  
```MySQL
select * from test where id=1 and (updatexml(1,concat(0x7e,(<SQL>),0x7e),1));  
```
#### 2.4.2.4 `geometrycollection()`：用于创建一个几何图形集合对象的函数。  
- `<SQL>`：可以替换为各种需要获取的信息语句  
```MySQL
# 方法1：  
select geometrycollection((select * from(select * from(<SQL>)a)b));  
# 方法2：  
SELECT geometrycollection((select * from(select count(*),concat((<SQL>),floor(rand(0)*2))x from information_schema.tables group by x)a));  
```
#### 2.4.2.5 `multipoint()`：空间数据类型函数，用于创建一个多点对象  
- `<SQL>`：可以替换为各种需要获取的信息语句  
```MySQL
# 方法1：  
select * from user_info where id=1 and multipoint((select * from(select * from(<SQL>)a)b));  
# 方法2：  
SELECT multipoint((select * from(select count(*),concat((<SQL>),floor(rand(0)*2))x from information_schema.tables group by x)a));  
```
#### 2.4.2.6 其他的几何对象方法  
`polygon`、`multipolygon`、`linestring`、`multilinestring`  
- `<function-name>`：上述函数  
- `<SQL>`：可以替换为各种需要获取的信息语句  
```MySQL
select * from test where id=1 and <function-name>((select * from(select * from(<SQL>)a)b));  
```
#### 2.4.2.7 `exp()`：用于计算自然指数 e 的幂次方  
- `<SQL>`：可以替换为各种需要获取的信息语句  
```MySQL
select * from user_info where exp(~(select * from(<SQL>)a));  
```

> [!tip]
> exp 函数报错注入在 MySQL 5.5.49 后不再生效，有关几何图形和空间数据类型的函数报错注入在 MySQL 5.5 中被陆续修复。
### 2.4.3 宽字节注入  
它利用 MySQL 编码中的漏洞，将单字节字符编码成双字节字符，从而绕过常规的过滤器和防护机制。在 MySQL 中，字符集可以由客户端指定，而不是服务器强制指定。因此，攻击者可以使用不同的字符集来编码攻击载荷，绕过输入验证和过滤。一般思路是将攻击载荷中的单字节字符通过特定的编码方式转换成双字节字符，从而绕过输入验证和过滤。  
GB2312、GBK、GB18030、BIG5、Shift_JIS 等这些编码都是常说的宽字节。  
    常见转义函数与配置：`addslashes`、`mysql_real_escape_string`、`mysql_escape_string`、**php.ini** 中 `magic_quote_gpc` 的配置。  
简单来说是需要通过宽字节，将转移函数所添加的字符进行合并，令其识别为其他的字符。  
  
### 2.4.4 盲注  
攻击语句不会返回想要得到的预期反应或者错误信息。需要其他的特殊方法来判断或者尝试。  
盲注实际上也是一种爆破手段。主要是通过字符串的截取，逐渐的每个字符的爆破 数据库名 → 数据库的表名 → 表中的字段名。  
**字符串截取**  
字符串截取的常用方法：`left`、`right`、`substr`、`mid`、`substring`  
通常字符串截取常与编码函数进行配合。  
编码：`ascii`、`ord`，如果可以使用 ascii 或者 ord ，需尽可能使用，理由如下：  
1. 明文字符可能存在特殊字符，干扰 SQL 语句的查询。  
2. 利用数字可以使用二分注入，加快效率。  
**字符串比较**  
- `like`：没有 `%` 可以尝试代替等号
- 正则表达式：  
    - `rlike`：使用的是 POSIX 扩展正则表达式语法  
    - `regexp`：使用的是 MySQL 正则表达式语法。如果大小写敏感，需要在目标字符串之前添加 `binary` 关键字  
- `between`：`value BETWEEN low AND high;`  
    - `x between i and i`就是表示`x`是否等于`i`的意思。  
- `in (expr1, expr2, expr3)`：判断是否存在于一个集合内。如果需要大小写敏感，要用`binary`关键字  
- `case`：一般不用来做比较，而是构造条件语句，在时间盲注中会用到。  
    - 简单 case 语句：`expression` 是要进行判断的表达式，`value1`、`value2` 等是需要判断的值，`result1`、`result2` 等是对应的返回结果，`else` 是当 `expression` 不等于 `value1`、`value2` 等时的默认返回结果。  
        ```MySQL
        CASE expression  
            WHEN value1 THEN result1  
            WHEN value2 THEN result2  
            ...  
            ELSE result  
        END  
        ```
    - 搜索 case 语句：`condition1`、`condition2` 等是需要进行判断的条件，`result1`、`result2` 等是对应的返回结果，`else` 是当所有条件都不满足时的默认返回结果。  
        ```MySQL
        CASE  
            WHEN condition1 THEN result1  
            WHEN condition2 THEN result2  
            ...  
            ELSE result  
        END  
        ```
#### 2.4.4.1 布尔盲注  
最常见的就是回显的内容不同，两者返回的数据包长度不同。也可以观察响应头，成功可以有 set-cookie。或者成功或者失败的服务器响应结果也会有所不同。  
  
#### 2.4.4.2 延时盲注  
通过在 SQL 语句中添加判断语句和延时函数，观察查询语句执行后的响应时间来判断是否成功执行。  
延时盲注是一种非常隐蔽的攻击方式，攻击者可以通过适当的延时时间控制方式，减小被检测到的风险，而且可以在不获取实际数据的情况下，通过时间差分析等方式推测出目标数据库中的数据信息。  
**常用方法：**  
- `IF(expr,if_true_expr,if_false_expr)`：`expr` 是一个条件表达式，如果该表达式为真，则返回 `if_true_expr`，否则返回 `if_false_expr`。  
- `SLEEP(seconds)`：让当前的线程暂停指定的时间。  
- `CASE WHEN (condition) THEN sleep(5) ELSE 0 END;`：`condition`需要判断的内容。  
**绕过方法：**  
- **`if`和`case`被禁用**  
    可以将需要判断的条件整合到`sleep`函数中，做乘法即可：`sleep(5*(condition))`。  
    如果判断条件为真，返回为1，乘法后为5，即延时5秒。如果判断条件为假，延时0秒。  
- **`sleep`被禁用**  
    1. `benchmark(count,expr)`：重复计算**`expr`**表达式**`count`**次。通过修改执行次数和执行的操作，可以精准控制延时时间。  
    2. 笛卡尔积：也就是**Heavy Query**，用的不多。例如：`select count(*) from information_schema.columns a, information_schema.columns b;`  
  
# 3. 常用绕过方法    
---
## 3.1 空格的替换  
1. 内行注释，`/**/`  
2. 换行符，`%0d%0a`  
3. 括号，`()`。注意：括号不可用于 mysql 自带关键字。例如：`select(id)from(student);`  
4. 反引号
	```MySQL
	例如：select`id`from`student`;
	```
## 3.2 `select` 关键字   
 在 MySQL 8.0 版本中，`table student` 等价于 `select * from student;`  
## 3.3 单引号和字符串替换  
写字符串的方法：可以利用 MySQL 的 `hex` 和 `unhex` 函数  
- `hex()`将输入的**字符串**或者**十进制数**转换为十六进制的字符串  
- `unhex()`，`hex()`的逆运算  
可以连续使用`unhex(hex())`进行操作，可以绕过大多数过滤。  
例如：`unhex(hex(6e6+382179))` → `'abc'`  
    1. `abc`转成16进制是`616263`  
    2. `616263`转十进制是`6382179`  
    3. 用科学计数法表示`6e6+382179`  
构造单引号的方法：需要用户能操作两个参数。  
1. 宽字节注入  
2. 转义法  
  
# 4. sqlmap  
---
sqlmap中文手册：  
[文档介绍 - sqlmap 用户手册](https://sqlmap.highlight.ink/)    

# 5. 漏洞修复和预防  
---
1. 在编程时使用预编译语句，此种方法会将用户输入部分视为数据，而不是 SQL 语句的一部分；  
2. 尽可能减少字符串拼接等不安全的构造 SQL 语句的方式，如试用，需要做好用户输入的验证；  
3. 加强用户输入的验证，检查长度及内容，配合使用白名单验证确保只输入预期内的字符；  
4. 严格控制数据库的权限，即便成功注入，减少攻击者写和执行的权限；  
5. 定期更新和数据库相关的软件及开发组件，修复已知的安全漏洞。

# 6. [[MyBatis的SQL注入]]  

