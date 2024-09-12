# LINUX

## 命令

### 文件

#### cat

concatenate: 连接文件并打印到标准输出设备上。

| Abbreviation                 | Explanation                                          |
| :--------------------------- | :--------------------------------------------------- |
| **-n 或 --number**           | 由 1 开始对所有输出的行数编号。                      |
| **-b 或 --number-nonblank**  | 和 -n 相似，只不过对于空白行不编号。                 |
| **-s 或 --squeeze-blank**    | 当遇到有连续两行以上的空白行，就代换为一行的空白行。 |
| **-v 或 --show-nonprinting** | 使用 ^ 和 M- 符号，除了 LFD 和 TAB 之外。            |
| **-E 或 --show-ends**        | 在每行结束处显示 $。                                 |
| **-T 或 --show-tabs**        | 将 TAB 字符显示为 ^I。                               |

#### chattr

| Abbreviation | Explanation                                                  |
| :----------- | :----------------------------------------------------------- |
| **a**        | 让文件或目录仅供附加用途                                     |
| **b**        | 不更新文件或目录的最后存取时间                               |
| **c**        | 将文件或目录压缩后存放                                       |
| **d**        | 将文件或目录排除在倾倒操作之外                               |
| **i**        | 不得任意更动文件或目录                                       |
| **s**        | 保密性删除文件或目录                                         |
| **S**        | 即时更新文件或目录                                           |
| **u**        | 预防意外删除                                                 |
| R            | 递归                                                         |
| V            | 显示                                                         |
|              | `chattr [-RV][-v<版本编号>][+/-/=<属性>][文件或目录...]` <br />+：开启	-：关闭	=：设置 |

#### cmp

compare: 比较两个文件是否有差异

| Abbreviation              | Explanation                                              |
| :------------------------ | :------------------------------------------------------- |
| **-c或--print-chars**     | 除了标明差异处的十进制字码之外，一并显示该字符所对应字符 |
| **-i<字符数目>**          | 指定一个数目                                             |
| **-l或--verbose**         | 标示出所有不一样的地方                                   |
| **-s或--quiet或--silent** | 不显示错误信息                                           |

#### diff



## 附录

### Section1 系统目录

![img](assets/d0c50-linux2bfile2bsystem2bhierarchy.jpg)

| package         | usage                                                        |
| :-------------- | :----------------------------------------------------------- |
| **/bin**        | 存放最经常使用的命令                                         |
| **/boot**       | 存放启动 Linux 时使用的一些核心文件，包括一些连接文件以及镜像文件 |
| **/dev**        | 存放 Linux 的外部设备(在 Linux 中访问**设备的方式**和**访问文件的方式**相同) |
| **/etc**        | Etcetera:存放所有的系统管理所需要的配置文件和子目录          |
| **/lib**        | 存放系统最基本的动态链接共享库，几乎所有的应用程序都要用到这些 |
| **/lost+found** | 一般为空，当系统非法关机后，这里就存放了一些文件             |
| **/media**      | Linux 会把识别的设备挂载到这个目录下(U盘、光驱等)            |
| **/mnt**        | 让用户临时挂载别的文件系统(如光驱的挂载)                     |
| **/opt**        | 给主机额外安装软件所摆放的目录                               |
| **/proc**       | Processes:存储的是当前内核运行状态的一系列特殊文件，这个目录是一个虚拟的目录，是系统内存的映射，可以通过直接访问这个目录来获取系统信息。这个目录的内容不在硬盘上而是在内存里 |
| **/sbin**       | 系统管理员使用的系统管理程序                                 |
| **/selinux**    | 安全机制防火墙相关文件                                       |
| **/srv**        | 存放一些服务启动之后需要提取的数据                           |
| **/sys**        | 该目录下安装了 2.6 内核中新出现的一个文件系统 sysfs 。       |
|                 | sysfs 文件系统集成了下面3种文件系统的信息：针对进程信息的 proc 文件系统、针对设备的 devfs 文件系统以及针对伪终端的 devpts 文件系统。 |
|                 | 该文件系统是内核设备树的一个直观反映。                       |
|                 | 当一个内核对象被创建的时候，对应的文件和目录也在内核对象子系统中被创建。 |
| **/tmp**        | 临时文件                                                     |
| **/usr**        | unix shared resources：用户的很多应用程序和文件都放在这个目录下 |
| **/usr/bin**    | 系统用户使用的应用程序                                       |
| **/usr/sbin**   | 超级用户使用的比较高级的管理程序和系统守护程序               |
| **/usr/src**    | 内核源代码默认的放置目录                                     |
| **/var**        | 习惯将那些经常被修改的目录放在这个目录下。包括各种日志文件   |
| **/run**        | 临时文件系统，存储系统启动以来的信息。<br />当系统重启时，这个目录下的文件应该被删掉或清除 |

**不要随意变动以下目录**：**/etc**,**/bin, /sbin, /usr/bin, /usr/sbin**,**/var**

### Section2:文件系统

### Section3:Shell脚本

最简单的脚本：

```shell
#!/bin/bash
echo "Hello World!"
```

###### Shell定义变量：

```shell
#有效变量名↓
RUNOOB="www.runoob.com"
LD_LIBRARY_PATH="/bin/"
_var="123"
var2="abc"
#无效变量名↓
variable_with_$=42
?var=123
user*name=runoob
variable with space="value"# 避免空格
```

> 注意：不要在等号两边加空格

##### Shell使用变量

```shell
echo $your_name
```

举例：

```shell
for skill in Ada Coffe Action Java; do
    echo "I am good at ${skill}Script"
done
```

> 设置只读变量：`readonly var`
> 删除变量：`unset var`

##### Shell声明高级变量

```shell
declare -i my_integer=42 	# 声明为Integer
declare -A associative_array    #关联数组
associative_array["name"]="John"
associative_array["age"]=30
```

##### Shell字符串

```shell
str='this is a string'

your_name="runoob"
str="Hello, I know you are \"$your_name\"! \n"
echo -e $str
# 输出：Hello, I know you are "runoob"! 
```

> 区分：单引号‘’，双引号“”，反引号``单引号字符串的限制：
>
> - 单引号里的任何字符都会原样输出，单引号字符串中的变量是无效的；
>
> - 单引号字串中不能出现单独一个的单引号（对单引号使用转义符后也不行），但可成对出现，作为字符串拼接使用。
>
> - 双引号的优点：
>
> 	- 双引号里可以有变量
> 	- 双引号里可以出现转义字符
>
> - 反引号在现代已经被替代：
>
> - ```shellfiles_list=`ls`
> 	files_list=`ls`
> 	files_list=$(ls)
> 	```

获取字符串长度：

```shell
string="abcd"
echo ${#string}   # 等价于${#string[0]} 输出 4
```

提取子字符串：

```shell
string="runoob is a great site"
echo ${string:1:4} # 输出 unoo
```

查找子字符串：

```shell
string="runoob is a great site"
echo `expr index "$string" io`  # 输出 4
```

> 查找字母i或o的位置，哪个先出现返回哪个

##### Shell数组

> bash只支持一维数组，不限定大小

```shell
array_name=(value0 value1 value2 value3)
# 或者
array_name=(
value0
value1
value2
value3
)
array_name[0]=value0
array_name[1]=value1
array_name[n]=valuen
```

读取数组：

```shell
valuen=${array_name[n]}
```

> 获取全部元素： `echo ${array_name[@]}`

获取数组的长度：

```shell
# 取得数组元素的个数
length=${#array_name[@]}
# 或者
length=${#array_name[*]}
# 取得数组单个元素的长度
length=${#array_name[n]}
```

##### Shell多行注释

```shell
: <<'COMMENT'
这是注释的部分。
可以有多行内容。
COMMENT

:<<'
注释内容...
注释内容...
注释内容...
'

:<<!
注释内容...
注释内容...
注释内容...
!
#或者：
: '
	这是注释的部分。
	可以有多行内容。
'
```

##### Shell传入参数

| 参数处理 | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| $#       | 传递到脚本的参数个数                                         |
| $*       | 以一个单字符串显示所有向脚本传递的参数。 如\$*"用「"」括起来的情况、以"\$1 \$2 … \$n"的形式输出所有参数。 |
| $$       | 脚本运行的当前进程ID号                                       |
| $!       | 后台运行的最后一个进程的ID号                                 |
| $@       | 与\$*相同，但是使用时加引号，并在引号中返回每个参数。 如"\$@"用「"」括起来的情况、以"\$1" "\$2" … "\$n" 的形式输出所有参数。 |
| $-       | 显示Shell使用的当前选项，与[set命令](https://www.runoob.com/linux/linux-comm-set.html)功能相同。 |
| $?       | 显示最后命令的退出状态。0表示没有错误，其他任何值表明有错误。 |

##### Shell运算与比较

| 运算符 | 说明                                          | 举例                          |
| ------ | --------------------------------------------- | ----------------------------- |
| +      | 加法                                          | `expr $a + $b` 结果为 30。    |
| -      | 减法                                          | `expr $a - $b` 结果为 -10。   |
| *      | 乘法                                          | `expr $a \* $b` 结果为  200。 |
| /      | 除法                                          | `expr $b / $a` 结果为 2。     |
| %      | 取余                                          | `expr $b % $a` 结果为 0。     |
| =      | 赋值                                          | a=$b 把变量 b 的值赋给 a。    |
| ==     | 相等。用于比较两个数字，相同则返回 true。     | [ $a == $b ] 返回 false。     |
| !=     | 不相等。用于比较两个数字，不相同则返回 true。 | [ $a != $b ] 返回 true。      |

| 运算符 | 说明：假定变量 a 为 10，变量 b 为 20                  | 举例                        |
| ------ | ----------------------------------------------------- | --------------------------- |
| -eq    | 检测两个数是否相等，相等返回 true。                   | [ \$a -eq $b ] 返回 false。 |
| -ne    | 检测两个数是否不相等，不相等返回 true。               | [ \$a -ne $b ] 返回 true。  |
| -gt    | 检测左边的数是否大于右边的，如果是，则返回 true。     | [\$a -gt $b ] 返回 false。  |
| -lt    | 检测左边的数是否小于右边的，如果是，则返回 true。     | [ \$a -lt $b ] 返回 true。  |
| -ge    | 检测左边的数是否大于等于右边的，如果是，则返回 true。 | [ \$a -ge $b ] 返回 false。 |
| -le    | 检测左边的数是否小于等于右边的，如果是，则返回 true。 | [ \$a -le $b ] 返回 true。  |

| 布尔运算符 | 说明：假定变量 a 为 10，变量 b 为 20                | 举例                                      |
| ---------- | --------------------------------------------------- | ----------------------------------------- |
| !          | 非运算，表达式为 true 则返回 false，否则返回 true。 | [ ! false ] 返回 true。                   |
| -o         | 或运算，有一个表达式为 true 则返回 true。           | [ \$a -lt 20 -o $b -gt 100 ] 返回 true。  |
| -a         | 与运算，两个表达式都为 true 才返回 true。           | [ \$a -lt 20 -a $b -gt 100 ] 返回 false。 |

| 逻辑运算符 | 说明：假定变量 a 为 10，变量 b 为 20 | 举例                                        |
| ---------- | ------------------------------------ | ------------------------------------------- |
| &&         | 逻辑的 AND                           | [[ \$a -lt 100 && $b -gt 100 ]] 返回 false  |
| \|\|       | 逻辑的 OR                            | [[ \$a -lt 100 \|\| $b -gt 100 ]] 返回 true |

| 字符串运算符 | 说明：假定变量 a 为 "abc"，变量 b 为 "efg"   | 举例                     |
| ------------ | -------------------------------------------- | ------------------------ |
| =            | 检测两个字符串是否相等，相等返回 true。      | [ $a = $b ] 返回 false。 |
| !=           | 检测两个字符串是否不相等，不相等返回 true。  | [ $a != $b ] 返回 true。 |
| -z           | 检测字符串长度是否为0，为0返回 true。        | [ -z $a ] 返回 false。   |
| -n           | 检测字符串长度是否不为 0，不为 0 返回 true。 | [ -n "$a" ] 返回 true。  |
| $            | 检测字符串是否不为空，不为空返回 true。      | [ $a ] 返回 true。       |

| 文件操作符 | 说明                                                         | 举例                      |
| ---------- | ------------------------------------------------------------ | ------------------------- |
| -b file    | 检测文件是否是块设备文件，如果是，则返回 true。              | [ -b $file ] 返回 false。 |
| -c file    | 检测文件是否是字符设备文件，如果是，则返回 true。            | [ -c $file ] 返回 false。 |
| -d file    | 检测文件是否是目录，如果是，则返回 true。                    | [ -d $file ] 返回 false。 |
| -f file    | 检测文件是否是普通文件（既不是目录，也不是设备文件），如果是，则返回 true。 | [ -f $file ] 返回 true。  |
| -g file    | 检测文件是否设置了 SGID 位，如果是，则返回 true。            | [ -g $file ] 返回 false。 |
| -k file    | 检测文件是否设置了粘着位(Sticky Bit)，如果是，则返回 true。  | [ -k $file ] 返回 false。 |
| -p file    | 检测文件是否是有名管道，如果是，则返回 true。                | [ -p $file ] 返回 false。 |
| -u file    | 检测文件是否设置了 SUID 位，如果是，则返回 true。            | [ -u $file ] 返回 false。 |
| -r file    | 检测文件是否可读，如果是，则返回 true。                      | [ -r $file ] 返回 true。  |
| -w file    | 检测文件是否可写，如果是，则返回 true。                      | [ -w $file ] 返回 true。  |
| -x file    | 检测文件是否可执行，如果是，则返回 true。                    | [ -x $file ] 返回 true。  |
| -s file    | 检测文件是否为空（文件大小是否大于0），不为空返回 true。     | [ -s $file ] 返回 true。  |
| -e file    | 检测文件（包括目录）是否存在，如果是，则返回 true。          | [ -e $file ] 返回 true。  |

##### echo命令

```shell
echo -e "OK! \n" # -e 开启转义
echo "It is a test"
```

##### printf命令

```shell
#!/bin/bash
 
printf "%-10s %-8s %-4s\n" 姓名 性别 体重kg  
printf "%-10s %-8s %-4.2f\n" 郭靖 男 66.1234 
printf "%-10s %-8s %-4.2f\n" 杨过 男 48.6543 
printf "%-10s %-8s %-4.2f\n" 郭芙 女 47.9876
# 输出
#姓名     性别   体重kg
#郭靖     男      66.12
#杨过     男      48.65
#郭芙     女      47.99
```

>**%s %c %d %f** 都是格式替代符，**％s** 输出一个字符串，**％d** 整型输出，**％c** 输出一个字符，**％f** 输出实数，以小数形式输出。
>**%-10s** 指一个宽度为 10 个字符（**-** 表示左对齐，没有则表示右对齐），任何字符都会被显示在 10 个字符宽的字符内，如果不足则自动以空格填充，超过也会将内容全部显示出来。
>**%-4.2f** 指格式化为小数，其中 **.2** 指保留2位小数。

| 转义序列 | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| \a       | 警告字符，通常为ASCII的BEL字符                               |
| \b       | 后退                                                         |
| \c       | 抑制（不显示）输出结果中任何结尾的换行字符（只在%b格式指示符控制下的参数字符串中有效），而且，任何留在参数里的字符、任何接下来的参数以及任何留在格式字符串中的字符，都被忽略 |
| \f       | 换页（formfeed）                                             |
| \n       | 换行                                                         |
| \r       | 回车（Carriage return）                                      |
| \t       | 水平制表符                                                   |
| \v       | 垂直制表符                                                   |
| \\       | 一个字面上的反斜杠字符                                       |
| \ddd     | 表示1到3位数八进制值的字符。仅在格式字符串中有效             |
| \0ddd    | 表示1到3位的八进制值字符                                     |

##### 流程控制

if语句：

```shell
if condition1
then
    command1
elif condition2 
then 
    command2
else
    commandN
fi

# 或者
if [ $(ps -ef | grep -c "ssh") -gt 1 ]; then echo "true"; fi
```

> 条件判断的两种写法：
>
> ```shell
> if [ "$a" -gt "$b" ]; then
>     ...
> fi
> if (( a > b )); then
>     ...
> fi
> ```

for语句：

```shell
for var in item1 item2 ... itemN
do
    command1
    command2
    ...
    commandN
done
# 或者
for var in item1 item2 ... itemN; do command1; command2… done;while 语句
```

while语句：

```shell
#!/bin/bash
int=1
while(( $int<=5 ))
do
    echo $int
    let "int++"
done
```

> 无限循环：
>
> ```shell
> while :
> do
>     command
> done
> ##########
> while true
> do
>     command
> done
> ##########
> for (( ; ; ))
> ```

until语句：

```shell
#!/bin/bash
a=0

until [ ! $a -lt 10 ]
do
   echo $a
   a=`expr $a + 1`
done
```

case语句：

```shell
case 值 in
模式1)
    command1
    command2
    ...
    commandN
    ;;
模式2)
    command1
    command2
    ...
    commandN
    ;;
esac
```

跳出循环：

```shell
#!/bin/bash
while :
do
    echo -n "输入 1 到 5 之间的数字:"
    read aNum
    case $aNum in
        1|2|3|4|5) echo "你输入的数字为 $aNum!"
        ;;
        *) echo "你输入的数字不是 1 到 5 之间的! 游戏结束"
            break
        ;;
    esac
done
```

##### Shell函数

```shell
[ function ] funname [()]
{
    action;
    [return int;]
}
```

> - 1、可以带function fun() 定义，也可以直接fun() 定义,不带任何参数。
> - 2、参数返回，可以显示加：return 返回，如果不加，将以最后一条命令运行结果，作为返回值。 return后跟数值n(0-255)

```shell
#!/bin/bash
# author:菜鸟教程
# url:www.runoob.com

demoFun(){
    echo "这是我的第一个 shell 函数!"
}
echo "-----函数开始执行-----"
demoFun
echo "-----函数执行完毕-----"
```

##### Shell重定向

| 命令            | 说明                                                |
| :-------------- | :-------------------------------------------------- |
| command > file  | 将输出重定向到 file。                               |
| command < file  | 将输入重定向到 file。                               |
| command >> file | 将输出以追加的方式重定向到 file。                   |
| n > file        | 将文件描述符为 n 的文件重定向到 file。              |
| n >> file       | 将文件描述符为 n 的文件以追加的方式重定向到 file。  |
| n >& m          | 将输出文件 m 和 n 合并。                            |
| n <& m          | 将输入文件 m 和 n 合并。                            |
| << tag          | 将开始标记 tag 和结束标记 tag 之间的内容作为输入。a |
