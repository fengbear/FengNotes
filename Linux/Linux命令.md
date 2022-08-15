## Linux命令

### 1、top

top命令用于实时的监控系统的处理器状态，以及其他硬件负载信息还有动态的进程信息等等。

还可以按照排名先后的显示 ，比如某个进程CPU，内存的使用情况排名。

> top命令的第一大板块，系统的负载信息

```c
top - 08:41:43 up 10 days, 19:28, 13 users,  load average: 1.22, 1.00, 0.84
Tasks: 1905 total,   2 running, 1903 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.0 us,  0.3 sy,  0.0 ni, 99.6 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem : 515878.1 total,  82799.0 free,  10683.7 used, 422395.4 buff/cache
MiB Swap:   8192.0 total,   8102.7 free,     89.3 used. 501153.0 avail Mem 
    
08:41:43 up 10 days // 当前的机器时间  date命令可以查看时间
19:28 // 当前系统运行了多久
13 users // 当前机器几个用户在使用
load average: 1.22, 1.00, 0.84 // 显示系统的平均负载情况
Tasks: 1905 total,   2 running, 1903 sleeping,   0 stopped,   0 zombie // 总共的进程任务情况
%Cpu(s):  0.0 us,  0.3 sy,  0.0 ni, 99.6 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st // cpu的使用百分比情况 us 用户占用的cpu百分比情况 sy 系统内核空间占用的cpu百分比 ni 用户进程空间占用的cpu百分比 id 空闲的cpu百分比情况 wa 等待输入输出占用的cpu百分比情况
MiB Mem : 515878.1 total,  82799.0 free,  10683.7 used, 422395.4 buff/cache // 内存的状态 
          // 物理内存总大小      // 空闲    
MiB Swap:   8192.0 total,   8102.7 free,     89.3 used. 501153.0 avail Mem // 交换空间的状态
```

> 系统负载（System Load）是系统CPU繁忙程度的度量，即有多少进程在等待被CPU调度（进程等待队列的长度）。
>
> 平均负载（Load Average）是一段时间内系统的平均负载，这个一段时间一般取1分钟、5分钟、15分钟。

> top命令的第二大板块

```bash
PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND 
PID 进程id号
USER 执行进程的用户是谁
PR 进程的优先级
NI nice值 越高优先级越高
VIRT 进程使用的虚拟内存总量 VIRT = RES + swap
RES 进程使用的物理内存大小
SHR 共享内存大小
S 进程状态

%CPU
%MEM cpu和内存的使用百分比情况
```

> top的实际使用

```bash
1 查看linux逻辑cpu个数
M 内存使用量，从大到小排序
top -c 会显示进程的绝对路径
top -d 秒数 // top进程刷新时间
top -n num // 刷新num次不刷新
top -p pid // 查看某一个进程的信息
```

### 2、Linux三剑客（grep sed awk）

文本处理工具，均支持正则表达式引擎

+ grep：文本过滤工具
+ sed：stream editor 流编辑器 文本编辑工具（对文本替换、删除）
+ awk：Linux的文本报告生成器（格式化文本）Linux是gawk

#### 正则表达式

![image-20220815172252439](picture/image-20220815172252439.png)

![image-20220815172649419](picture/image-20220815172649419.png)

#### grep

作用：文本搜索工具，根据用户指定的“模式”（过滤条件）对目标文本逐行进行匹配检查，打印匹配到的行

```bash
grep [options] [pattern] file
      参数       匹配模式
参数：
    -i 忽略字符大小写
    -o 仅显示匹配到的字符串本身
    -v 显示不能被模式匹配到的行
    -E 支持使用扩展的正则表达式
    -q 静默模式 不输出任何信息 
    -n 显示匹配的行号

匹配模式：
	可以是普通的文字符号，也可以是正则表达式
```

#### sed

sed是操作、过滤和转换文本内容的强大工具

常用功能包括结合正则表达式对文件实现快速增删改查，其中查询的功能中最常用的两大功能就是过滤（过滤指定字符串）、取行（取出指定行）。

![image-20220815173648806](picture/image-20220815173648806.png)

```bash
sed [选项] [sed内置命令字符] file
选项：
	-n 取消默认sed的输出 常和sed内置命令p一起使用
	-i 直接将修改结果写入文件 不用-i sed修改的是内存数据
	-e 多次编辑
	-r 支持正则表达式
内置命令字符：
	a append对文本进行追加 在指定行后添加一行/多行文本
	d delete 删除匹配行
	i insert 插入文本，在指定行前插入一行/多行文本
	p print 打印匹配行内容
	s/正则/替换内容/g 匹配正则内容，然后替换内容（支持正则），结尾g代表全局匹配

sed匹配范围：
	空地址 全文处理
	单地址 指定文件某一行
	/pattern/ 被模式匹配到的每一行
	范围区间 10,20 十到二十行  10,+5 第10行向下5行  /pattern1/, /pattern2/
	步长 1~2 表示1、3、5、7  2~2 表示2、4、6、8
```

##### 案例

数据

```
My name is chaoge.
I teach linux.
I like play computer game.
My qq is 877348180.
My website is https://pvthonav.cn.
```

**输出第2、3行**

```bash
sed -n '2,3p' xf.txt 
/*
I teach linux.
I like play computer game.
*/ 
sed -n '2,+3p' xf.txt 
I teach linux.
I like play computer game.
My qq is 877348180.
```

**过滤出含有linux的字符串行**

```bash
sed -n "/linux/p" xf.txt 

I teach linux.
```

**删除含有game的行**

```bash
sed "/game/d" xf.txt
// 没加-n 会默认输出到屏幕
// 此时文件中是没有改变的，修改的是内存数据 加-i会将文件中删除
```

**将文件中My全部替换位His**

```bash
sed 's/My/His/g' xf.txt
```

**替换所有My为His，同时换掉QQ号为888**

```bash
sed -e 's/My/His/g' -e 's/877348180/888/g' xf.txt
```

> -e 多次编辑

**在文件第二行追加内容a字符XX，写入到文件，还得添加-i**

```bash
sed -i '2a XX' xf.txt
```

**对文件每一行都加一个X**

```bash
sed 'a X' xf.txt
```

> 空格表示全文处理

#### awk

格式化文本内容，对文本进行复杂处理，更像是一门编程语言，支持条件判断、数组、循环等功能

```bash
awk [option] 'pattern {action}' file 
awk 参数 '条件动作' 文件
```

![image-20220815194518876](picture/image-20220815194518876.png)

action指的是动作，awk擅长文本格式化，且输出格式化的结果，因此最常用的动作就是print和printf

> 当执行命令 awk '{print $2}' xf.txt 表示把xf文本第二列信息输出
>
> awk默认以空格为分隔符，且多个空格也识别为一个空格，作为分隔符
>
> awk是按行处理文件，一行处理完毕，处理下一行，根据用户指定的分隔符去工作，没有就默认空格

指定分隔符后，awk把每一行切割后的数据对应到内置变量

![image-20220815195206591](picture/image-20220815195206591.png)

##### awk内置变量

![image-20220815195636905](picture/image-20220815195636905.png)

##### 案例

```
xf1 xf2 xf3 xf4 xf5 
xf6 xf13 xf14 xf19 xf111 
xf7 xf12 xf15 xf18 xf122
xf8 xf11 xf16 xf17 xf133 
```

**一次性输出多列**

```bash
awk '{print $1,$2}' xf.txt

加逗号相当于会输出空格
```

**自定义输出内容**

awk必须外层单引号，内层双引号，内置变量不得添加双引号，否则会被识别为文本。

```bash
awk '{print "第一列",$1,"第二列",$2}' xf.txt 
第一列 xf1 第二列 xf2
第一列 xf6 第二列 xf13
第一列 xf7 第二列 xf12
第一列 xf8 第二列 xf11
```

##### awk参数

```
-F 指定分割字段符
-v 定义或修改一个awk内部变量
-f 从脚本文件中读取awk命令
```

##### 案例

```
a:/sdf/asdfasdf
b:/asdf/asdfasdf
c:/asdf/asdfasdgasdg
d:/dsf/afsdf/asdf
e:/adsa/asdf/asdg/asdfasdgasdg
f:/asdf/asdfasdf/
g:/sad
```

**显示文件第五行**

```bash
# NR在awk表示行号，NR==5表示行号是5的那一行
# 一个等号是修改变量值
awk 'NR==5' xf.txt

e:/adsa/asdf/asdg/asdfasdgasdg
```

**显示3-5行且输出行号**

```bash
awk 'NR==3,NR==5 {print NR,$0}' xf.txt
```

**以冒号进行切割**

```bash
awk -F ":" '{print &1}' xf.txt
```

##### awk变量



##### awk格式化输出



##### awk模式

### 3、ps

用于显示当前进程的状态。

显示其他用户启动的进程（a）
查看系统中属于自己的进程（x）
启动这个进程的用户和它启动的时间（u）

### 4、find

### 5、netstate