---
layout: post
title: linux shell script basic usage
categories: shell
description: about some usage about linux shell script, mainly about awk, sed and simple script with database (MySQL / TiDB).
keywords: shell, linux
---

## linux shell script

### shell 和 CLI

shell 不是操作系统内核的一部分，而是构建在内核上的一个应用程序，是人与系统交互的介质。

在早期 Linux 系统中，与 Unix 系统进行交互的唯一方式就是借助 shell 所提供的文本命令行（command line interface， CLI）来操作应用程序，管理文件等。

早期亚终端是利用串行电缆链接到 Unix 系统上的一台显示器和键盘，该亚终端是该 Unix 的控制台。现在 Linux 进入 CLI 的方式，可以直接退出图形化模式，进入文本模式，这种模式称作 Linux 控制台，它仿真早期的亚终端。

现代 Linux 启动之后，会自动创建出一些（5-6个）虚拟控制台，其是运行在内存中的终端会话，不需要在计算机上硬件连接多个真实的亚终端，当你在图形化 terminal 新建窗口时候，出现的 ttys00x 其实就是连接到已经运行的虚拟控制台-亚终端。

![Image](/images/lss_ttys.jpg)

shell 包含一组内部命令（builtin cmd），用这些命令可以完成诸如设备操作，文件操作，启动停止程序。你也可以将多个 shell 命令放到文件中作为程序串来执行，这些文件被称为 shell 脚本。

在 Linux 系统上，通常有好几种 shell 可用，不同的 shell 有不同的特征。不同 shell 的脚本之间不一定能够移植使用，由于几乎所有 Linux 发行版（除了 Ubuntu）都会以 bash shell 作为默认新 shell，所以写 bash shell 脚本会更加通用和朴素。当前有更加强大的 zsh 弥补了 bash 的不足，并且完全兼容 bash 语法。在日常开发使用 zsh 工具会更加强力和顺手。
  
### 基本 bash shell

本节会介绍一些除了日常 Linux 操作的命令之外的一些容易忽视但是还比较使用的点：

- ls 命令的简单匹配过滤，在 pattern 中使用 ？ 来代表一个字符，× 来代表 0 个或者多个字符： ls -l prefix*，此外使用 \[xx], \[!xx] 来指定字符组或者排除字符组：ls -l ca\[tp]
- ls -i 可以查看文件的 inode 号，硬链接的两个文件具有相同的 inode，软链接的两个文件，则有不同的 inode。
- less 是 more 翻页显示升级版，可以识别键盘上下左右键位和提供了一些高级搜索的功能，在查看很大的 tidb log 的时候比较有用。
- head 和 tail 可以专门查看文件的头部和尾部的指定的行数，可以使用 -n 显示指定的行数。
- ps 显示某个时刻的进程信息，top 可以实时显示进程信息。
- 信号是 Linux 进程通信的一个简约手段，对特定的信号规定其代表的特定意义，来达到默认的规约。
    - 1  HUP 挂起
    - 2  INT 中断
    - 3  QUIT 结束运行
    - 9  KILL 无条件截止
    - 11 SEGV 段错误
    - 15 TERM 尽可能终止
    - 17 STOP 无条件停止，但是不终止
    - 18 TSTP 停止或暂停，但继续在后台运行
    - 19 CONT 在 STOP 或 TSTP 之后恢复运行
- ctrl+c 会发送 SIGINT 信号，ctrl+z 会发送一个 sigtstp 信号，你可以在脚本中使用 trap 可以捕获信号。如：trap commands sig
- kill 命令会默认向程序发 term 命令，kill 也可以直接指定信号类型，-s 参数可以指定信号类型或者信号号码，也支持直接使用类似 -9/-kill 来只指定信号。
- df 显示总体磁盘使用情况，du 可以显示具体的文件夹磁盘使用情况。
- grep 的反向搜索功能，grep -v \[pattern] file, pattern 可以使用正则。
- tar & zip 归档和压缩实际上是两个命令，但很多情况我们会看到 xx.tar.zip 结尾的文件，tar 命令的 -z 参数可以将输出重定向给 zip 命令来压缩和解压，这样可以避免显示操作 zip 命令。其也是 Linux 中分发源程序的普遍方法。

### 理解 shell
运行的 shell 是一个进程，在显式运行 bash 命令之后，就会开启一个新的 shell 进程，这种关系叫做 shell 的父子关系，如此称为显式的开启子 shell。在命令列表（用`()`包括一连串命令，`;`分隔）执行时也可以隐式的开启子进程来运行命令。此外后台模式（使用 & 后缀）运行的作业也是开子进程来运行。

内建命令和外部命令的区别是前者不需要使用子 shell 来执行，因为它们已经和 shell 编译成了一体，使用 type 命令可以区分命令是内建的还是外部的，如： type cd。

环境变量基本都是大写，global 变量对于本 shell 及子 shell 都是可见的（全局变量的传递性），local 变量只对创建它们的 shell 可见。这在执行 shell 脚本时候传递参数时候会比较有用。使用 env / printenv 显示当前 shell 可见的全局变量。

Linux 并没有特殊的命令来只看局部变量，但是可以 set 命令来查看所有的变量（包括全局的，局部的和用户自定义的）。定义用户自定义变量时，需要注意等号两端不能有空格，对于有空格的值必须使用单引号来括起来。如：a='aaa', 如果想让其变为全局变量可以 export a（最好是定义成 A），该变量可以传递到子 shell。

### 构建 shell 脚本
创建 shell 脚本时，必须在文件的第一行指定要使用的 shell 类型，并以注释符号（#）开头。

```
#!/bin/bash 
```

shell 脚本使用的命令必须是内建命令，或者是外部命令，这意味着你的二进制文件所在的 path 必须要在全局变量 $PATH 中。

shell 脚本中使用全局变量，可以使用 $var 或者 ${var} 来 eval 变量的值。这种 eval 效果即使在字符串中，也能正确的 eval，而不是识别为字符串的一部分。

```
echo $HOME, ${HOME}
ehco "hahah $HOME"
``` 

命令替换应该是 shell 脚本中最为重要的一种使用方法了，其本意为从命令输出中提取信息，并将其赋值给变量。这个特性在处理流式数据时候尤其方便。有两种方法可以将命令的输出赋值给变量：
- 反引号 (`)
- $() 注意这个与 eval 变量的 ${} 的区别

```
a=`ls`
a=$(ls)
echo $a, ${a} # a 是变量，存储的是 ls 的结果。
```

重定向可以将命令输出定向输出到文件中（或者从文件读），管道可以将命令输出重定向到另外一个命令。

```
ls > ls_output
ls | cat
```

表达式计算, 由于 expr 表达式比较受限这里不予介绍，这里介绍方括号数学表达式计算。注意方括号只能支持整数运算，实数运算需要 bc, zsh 下提供实数计算的支持，可以使用双 (()) 或者 let 命令.

```
a=$[1+5]  # 将数学表达计算结果 eval 出来赋值给 a
b=$[$a+5] # 表示式中需要先 eval 一下 a 变量的具体值
```

命令退出码，$?保存了上一个命令的退出状态码，成功退出为 0, 错误就是一个整数（一般在1-255）

```
# finish script successfully (对于 shell 脚本来说自身也是一个程序，需要有一定的退出码供外部识别)
exit 0
```

结构化操作命令，是 shell 脚本逻辑处理的骨架，if 后面的对象的逻辑是靠其对出码来识别的，0 則继续，不然 else（这里区别其他编程语言）

```
if command
then
    commands
else
    commands # 另外还有 elif
fi
```

test 命令：由于 if 命令只能是被 0-255 的状态退出码，退出状态码之外的条件需要使用 test 命令或者方括号 [] 来检测。简单来说 [] 可以放普通的条件表达式，条件不满足组，test 命令会帮你返回非 0 状态码，反之亦然。

```
if [ condition ]   # 注意`[`之后，`]`之前的空格，才能被正确识别为 test 条件表达式
then               # condition 通常有数值比较，字符串比较，文件比较
    commands
fi

# 示例
if [ 5 -gt 4 ]
if [ "aa" < "bb"]     # `<` 只能用于字符串比较
if [ -f $file_name]
```

for 命令：遍历的一种方式，一般常用对 list 变量进行遍历，也可以对命令输出进行遍历。

```
for one in $(cat $file_name)  # 变量替换，命令输出暂存一个变量，$temp 就会显示 cat 的所有输出
do
  commands
done 

my_list = "ha he hi"          # 直接用 my_list 也可以，默认空格分割，可以改动 IFS 调节默认分割符

for (( i = 0; i < n; i++ ))   # c 风格的 for 也可以
do
  commands
done
```

使用外部参数，$# 代表运行脚本时命令行参数个数，$1, $2... 可以分别引用对应值，$* 将所有参数看成一个字符串，而 $@ 将所有参数看成一个 list 变量。

shell 函数向 C 语言一样，需要定义在前，使用在后，定义函数的方式有两种：

```
function name{
}

name(){
}
```

在编写完成 shell.sh 之后，需要给予文件可执行权限，才能被正确执行（chmod u+x shell.sh）。

### awk & sed

awk 和 sed 是 linux 中处理数据常见的工具，两者都是基于 readline 流式处理的（区别 vim 的全局打开），awk 更加聚焦列上的操作，sed 则更加聚焦行上的操作。awk 和 sed 不会对原有的数据进行修改，在读处理的过程中进行内存副本操作之后会输出到其他的地方 STDOUT 或者重定向。

```
sed options program file       # 此为基本的操作模式
sed 's/a/b' file_name          # s 是 substitute 命令，会将 line 中的数据的模式 a 替换称模式 b
sed 'address s/a/b' file       # address 可以指定特定行, num,num 可以指定行数区间，$ 代表尾部
sed '/pattern/s/a/b' file      # 也可以加正则过滤
sed '2,3d' file                # d 是 delete，表是删除动作
sed '1i\new line' file         # i 是 insert，表示在第一行之前插入新行“new line”。注意`\`。
sed '/cat/a\append line' file  # a 是 append，表示在匹配的cat的行后添加一列“append line”。注意`\`。
sed '/hello/c\new line' file   # c 是 change，表示将匹配的hello行修改为新行“new line”。注意`\`。
```

```
awk options program file
awk '{print "the second field is", $2}' # 对于一行中的不同列（默认以空格分割），输出第 2 列，注意这里 $2 （类似命令行参数引用方式）放到字符串的引号中是无法 eval 的，但是 shell 可以。
awk 'BEGIN{print "start"}{print "doing", $1}END{print "end"}'     # 可以在流式数据进来做一个动作 {}，中间处理是一个动作 {}，处理完之后是一个动作 {}。
awk '/pattern/{}'              # awk 支持 extended 级别的正则语法，sed 只能支持一般的正则
awk '{if($1 > 20){print $1}}'  # awk 中甚至可以使用结构化语法来处理逻辑，sed 则不行，更多结构语法请参考附录
awk '{print tolower($1)}'      # awk 内嵌入许多内建函数（区别 shell 函数），在 awk 命令中只能使用 awk 能识别函数或者变量，想 sed，ls 这些是 shell 的命令，无法使用！
```

关于 sed 的多行操作、模式引用、模式空间、分支跳转等高级用法，这里不再展开，详情可以查看附录中书籍。
关于 awk 更多的函数介绍，自定义函数，变量，结构化语法，这里不再展开，详情可以查看附录中的书籍。

### shell 操作数据库

shell 上操作 MySQL 和 TiDB，比较常见的通过 mysql client 命令进入到数据库交互模式之前，然后输入用户命令，等待结果并返回。但是这种结果是无法提取到交互创建之外的 shell 环境中的，并且这中基于交互式的 SQL 命令窗口无法在 shell 中使用自动化脚本来操作。

同样是外部的 mysql 命令，要想避免进入交互是 SQL 窗口，可以使用 -e 参数，将 SQL 作为 mysql 命令的一部分，这样就可以直接在 shell 窗口部分接受数据。这点是有很多好处的，比如可以直接重定向到 sed，awk 来处理 SQL 的结果集合：

```
mysql -uroot -P4000 -e 'select * from test.t' | awk '{if ($1="Tom"){print $2}}'  # 在结果集中处理第一列为 Tom 的行，并打印其第二列（类似 SQL 算子中的 filter 和 projection）。
```

但是，由于 -e 参数只能附带一个字符串的内的 SQL，所以不是很方便，使用 << 可以将输入重定向到具有交互式窗口的应用中。<< 在 bash 文档中称为 “here documents” 是一种特殊的重定向模式，用来将输出重定向到由该命令打开的交互式 shell（比如 python，mysql）中。

```
mysql -uroot -P4000 <<EOF   # 需要使用 EOF 标示需要重定向的文本开始和结束的位置
show tables;
select * from test.t;
select * from test.t1;
EOF | sed -n '/Tom/p'       # 只打印带有 Tom 行的结果集输出
```

### 附录

shell 脚本细节比较多，在写 sed & awk 中时，需要注意不要在其 commands 范围内引入 shell 脚本用的语法习惯，变量，函数和相关命令。本文介绍的仅仅是 shell,sed,awk 的简单日常使用部分，更多的高级用法需要大家参考《Linux命令行和shell脚本编程大全》，在日常工作中需要多读多写，这样再需要 shell 或者感觉重复工作的时候会比较自然的联想自动化 shell 脚本的解决方式。

以下是一个比较例子，查找一个 commit hash 所在的 tag，在 oncall 中会比较用的多，人为查找估计会比较耗时：

```shell
#!/bin/bash

# This shell script will help you to locate where the git tag is when given a commit hash.
# Run it in correct git branch and git directory.
# Pass the COMMIT_HASH in outter shell as parameter.

if [ -z $1 ]
then
 echo "You should pass the git hash in"
else
 echo "find the git tag where the <"$1"> is in."
fi

function HandleTags
{
 for tag in $(git tag)
 do
  GetTagHash "git show $tag"
  echo "$tag $hash"
 done
}

# The return value of function only can be numeric.
# So here we use global variable to receive commit hash string.
function GetTagHash
{
  hash=$(eval "$1" | sed -n -e '/^commit /p' | awk '{print $2}')
}


# Find all the git tag out as input.
# For every tag find it's commit hash.
hash=""
# store the standard descriptor.
exec 3>&1
# redirect 1
exec 1>tag_hash_map.txt
HandleTags
# restore the standard descriptor to 1.
exec 1>&3
# destory the descriptor 3.
exec 3>&-

# list all the git log out
# Streaming scan the git log hash with record recent git tag info.
OLDIFS=IFS
IFS=$'\n'
tag="hasn't been released"
for line in $(git log --pretty=oneline)
do
 # update tag state firstly if it has.
 left_hash=${line%% *}
 temp_tag=`awk '{ if ( var == $2 ){ print $1 }}' var=$left_hash tag_hash_map.txt`
 # compare target hash and git log hash
 # the blank here shouldn't be eliminated, otherwise it's a assginment.
 # the blank in [ and ] is also necessary.
 # use ${#tag} to get the str length of tag.
 if [ ${#temp_tag} -ne 0 ]
 then tag=$temp_tag
 fi
 # judge the every hash in git log and $1.
 # if equal, the tag is which tag it is in.
 if [ $left_hash == $1 ]
 then break
 fi
done
IFS=OLDIFS
rm tag_hash_map.txt
echo "The commit hash <"$1"> is in tag ["$tag"]."
```





