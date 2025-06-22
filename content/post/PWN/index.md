---
title: pwn.college Getting Started
description: pwn Getting Started部分的一些心得记录
slug: hello pwn.college
date: 2025-05-29 00:00:00+0000
categories:
    - PWN
tags:
    - PWN
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---


# PWN Getting Started

## **1、Linux Luminarium**

### **Practicing Piping**

#### Level8  grepping errors

知识点：

```
# 通过2>$ 1的方式将error输出到标准输出
/challenge/run 2>&1 | grep pwn

#注意是&符号而不是$

# grep的用法

grep [pattern] [filepath]
grep X ./x.txt
```

#### Level9  duplicating piped data with tee

知识点：

tee命令的作用：tee命令可以将标准输入的内容导入到标准输出以及多个文件中

```shell
/challenge/pwn | tee hint.txt | /challenge/college
```

![image.png](image.png)

随后按照提示的用法，将pwn命令的输出导入给college即可得到flag

#### Level11 split piping stderr and stdout

```shell
/challenge/hack 2> >(/challenge/the) | /challenge/planet 
/challenge/hack 2> >(/challenge/the) > >(/challenge/planet)
/challenge/hack 2> >(/challenge/the) 1> >(/challenge/planet)
```

分流输出stderr与stdout。

1. `2> >(...)`：将 stderr（文件描述符 2）重定向到一个子shell，子shell 中运行 `/challenge/the` 命令。
2. `> >(...)`：将 stdout（文件描述符 1）重定向到一个子shell，子shell 中运行 `/challenge/planet` 命令。

**`> >()` 和`|` 的区别**

- **`> >()`**：
    - 会创建子shell 来处理重定向，可能会稍微影响性能，但在大多数情况下差异不大。
    - 适用于需要灵活处理多个输出流的复杂场景。
- **`|`**：
    - 直接在当前 shell 中处理，性能较高。
    - 适用于简单的命令链式调用。

### **Data Manipulation**

这一章的关卡也挺简单的，首先是命令`tr`的使用方法，总结一些必要的知识点。

```shell
# 1、替换字符串，将字符串中的abc替换为ABC，第一关就是交换一下大小写
tr abc ABC 
/challenge/run | tr ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ

# 2、删除字符，使用-d标签
tr -d %^
/challenge/run | tr -d ^%

# 3、删除特殊字符\n，这一关提到了一个知识点，\字符要用\\表示
/challenge/run | tr -d "\n"

```

然后是`head`的使用，这些命令虽然简单，但是遗忘掉用法也很容易，还是得记录一下。

```shell
# 4、使用head -n num输出前num行的内容
/challenge/pwn | head -n 7 | /challenge/college 
```

最后是命令`cut`的使用。这条命令简单来说就是从列提取内容。

```shell
# 5、cut的-d参数用于指出分隔符，-f则指出具体提取哪一列的内容
/challenge/run | cut -d " " -f 2 | tr -d "\n"

```

### **Shell Variables**

这一部分的关卡都很简单

总结一部分知识点

```shell
# 设置变量，这样设置的变量仅对当前shell可见
VAR=value

# 使用export设置变量，这样设置的变量对当前shell及其子shell都可见
export VAR=value

# 可以使用echo和env查看设置的变量
echo $VAR
env

# 使用$()获取命令的输出
VAR = $(cat file)

# 使用read读取输入
read VAR

# 使用read读取文件

read VAR < file
```

### **Processes and Jobs**

知识点总结

```
Ctrl+Z 暂停
fg 启用并转到foreground
bg 启用并转到background

运行一条命令后，可以使用 echo $? 查看exit code
```

### **Perceiving Permissions**

```
chown 改变文件归属
chgrp 改变文件归属组
chmod u/g/o/a +/-/= rwx [file]
chmod u/g/o/a=- [file]直接清除权限
chmod u+s [file] 让其他用户以该文件的owner权限接触该文件

```

### **Silly Shenanigans**

1、Bashrc Backdoor

.bashrc中的命令会在启动时被执行，可以用作一些恶意行为。

2、Sniffing Input

这一关的通关思路是修改`/home/zardus/.bashrc`，在其目录下创建一个flag_checker，将该脚本的路径加入到PATH，使zardus执行flag_checker时执行的是该目录下的脚本，读取并打印flag。

修改.bashrc如下：

![bashrc](bashrc.png)

3、Overshared Directories

这一关用我上一关的方法也可以成功读取flag。

4、Trikey Linking

这一关，zardus会向`/tmp/collab/evil-commnands.txt`中写入读取flag的命令，我们可以建立一个evil-commands.txt到.bashrc的链接，让zardus将命令写入.bashrc，从而读取flag。

```
rm /tmp/collab/evil-commands.txt
ln /home/zardus/.bashrc /tmp/collab/evil-commands.txt
/challenge/victim
/challenge/victim
```

5、Sniffing Process Arguments

这一关很简单，通过`ps -aux`查看zardus的进程及其涉及到的密码，登录并读取flag。

6、Snooping on configurations

这一关也很简单，它主要就是为了告诉我们用户的.bashrc对于其他用户是默认可读的。我们通过查看.bashrc获取key就可以。

### **Daring Destruction**

1、The Fork Bomb

这一关编写一个脚本，不停地执行fork操作，直到系统资源被耗尽。

```python
import os
while True:
    os.system("python3 test.py &")
```

```
/challenge/check
python3 test.py
```

不过这一关让我比较好奇的是，如果仅仅是执行`python3 test.py`，系统资源并不会被耗尽，而`python3 test.py &`这种background进程，则会很快消耗掉资源，这是为什么？

原因也很简单，加上了`&`之后，进程会变为非阻塞式，程序会不断运行，而去掉之后，则会等待命令执行完毕之后再执行后面的命令，因为不会带来很大的开销。

2、Disk-Space DoomsDay

```
yes > ./x.txt
/challenge/check
rm ./x.txt
/challengr/check
```

3、rm -rf /

阅读/challenge/check：
![check](check.png)

当根目录下的文件被删到一定程度时就会打印flag内容。

这一关开启两个terminal，一个执行`/challenge/check`，另一个执行`rm -rf --no-preserve-root /`，等待一段时间后，check就会打印flag。

4、Life after rm -rf /

这一关和上一关类似，但是`/challenge/check`不会直接打印/flag，而是在检测到条件满足后，将flag的值再次保存到新的/flag中。

然而使用了`rm -rf /`之后，cat已经无法使用，这时候只能使用buildin内置命令，比如使用`read`读取文件。

```
read x < /flag
$x
```

5、Finding meaning after rm -rf /

这一关和上一关的不同之处在于，flag会被保存为一个随机的名称，然而ls命令此时已经不可用，我们可以用echo读取出文件名。
```
echo /*
```

## **2、computing 101**

### Your first program

```asm
# 这一部分主要讲述了汇编代码的编写应用

汇编程序保存至.s文件中
使用syscall进行系统调用，系统调用编码保存在rax中
如：
mov rax, 60
syscall
调用60号系统调用exit

使用寄存器进行传参，如exit的参数保存在rdi中
mov rdi, 42
mov rax, 60
syscall

# 编译为可执行文件
编译前在文件头部加上
.intel_syntax noprefix 告知使用的是Intel汇编编码格式

as -o asm.o assemble.s
ld -o exe asm.o

# write syscall
syscall 编号为1
三个参数
rdi 要写入的文件描述符
rsi 字符串起始地址
rdx 字符串长度

```

### **Debugging Refresher**

set disassembly-flavor intel 设置为intel格式

**level8**

win+12 +20 +24 +33处，[rax]指向的地址为0，为nullptr，会引发segmentation fault

因而此处直接跳过错误代码，跳转到win+35进行后续操作即可

```
   0x580e06449951 <win>:        endbr64 
   0x580e06449955 <win+4>:      push   rbp
   0x580e06449956 <win+5>:      mov    rbp,rsp
   0x580e06449959 <win+8>:      sub    rsp,0x10
   0x580e0644995d <win+12>:     mov    QWORD PTR [rbp-0x8],0x0
   0x580e06449965 <win+20>:     mov    rax,QWORD PTR [rbp-0x8]
   0x580e06449969 <win+24>:     mov    eax,DWORD PTR [rax]
   0x580e0644996b <win+26>:     lea    edx,[rax+0x1]
   0x580e0644996e <win+29>:     mov    rax,QWORD PTR [rbp-0x8]
   0x580e06449972 <win+33>:     mov    DWORD PTR [rax],edx
   0x580e06449974 <win+35>:     lea    rdi,[rip+0x73e]        # 0x580e0644a0b9
   0x580e0644997b <win+42>:     call   0x580e06449180 <puts@plt>
   0x580e06449980 <win+47>:     mov    esi,0x0
   0x580e06449985 <win+52>:     lea    rdi,[rip+0x749]        # 0x580e0644a0d5
   0x580e0644998c <win+59>:     mov    eax,0x0
   0x580e06449991 <win+64>:     call   0x580e06449240 <open@plt>
   
   jump *win+35
   c
```

### **Building a Web Server**

正好借这一部分复习一下汇编的知识。这一章一方面是要经常查表，查看各个[系统调用](https://www.cnblogs.com/tcctw/p/11450449.html)的用法，另一方面就是考察一些程序设计基本功。题目要求写一个处理GET请求和POST请求的汇编程序。

#### PART1

创建`socket`，以及执行`listen`，`bind`等系统调用，完成一个初始化。调用`bind`的时需要在栈内构建一个结构体`sockaddr`。
```
    mov rdi, 2
    mov rsi, 1
    mov rdx, 0
    mov rax, 41     # create socket
    syscall 
    mov qword ptr [rsp - 24], rax

    mov qword ptr [rsp - 24], 0          # fd
    mov word  ptr [rsp - 16], 2          # AF_INET (2 bytes)
    mov word  ptr [rsp - 14], 0x5000     # Port 80 (network byte order: 0x5000)
    mov dword ptr [rsp - 12], 0x0000000  # 0.0.0.0 (network byte order: 0x7F000001)
    mov qword ptr [rsp - 8], 0           # sin_zero (8 bytes padding

    mov rdi, [rsp - 24]      # fd
    lea rsi, [rsp-16]
    mov rdx, 16
    mov rax, 49     # bind
    syscall 

    mov rdi, [rsp - 24]      # fd
    mov rsi, 0
    mov rax, 50     # listen
    syscall 

```

#### PART2

接收request并创建response，使用`accept`获取request，从中读出要处理的文件名，将本地的文件内容`write`到相关的fd中。
这里我简单粗暴在栈上开辟了一个超大的空间用于保存request的内容，然后从头遍历，**把文件名的后一个byte写为`\0`**，再把指向文件名的地址传送给系统调用`open`即可。`accept`会返回一个套接字文件描述符，往里面`write`文件内容即可。

#### PART3

对于每一个请求，使用子进程进行处理。

这里涉及到系统调用`fork`的用法。调用之后，父子进程都将从当前程序的位置往后执行，且父子进程**不共享栈空间**。在`fork`后要加一个判断逻辑，根据fork的返回值判断当前进程是父进程还是子进程，**返回值为0则是子进程，否则为父进程**。对于父进程，程序进入循环，接收下一个请求，对于子进程，程序处理现在的这个请求。

#### PART4
处理POST请求。POST请求的内容包括要写入的文件名，要写入的内容以及长度。文件头`Content-Length`会说明要写入内容的长度。

这里要处理两部分的内容，第一是匹配该文件头，第二是读取文件长度。下面的代码是我的处理逻辑。

```
    # 获取length所处的位置
    mov rdi, 0
    mov rsi, 0 
    mov rcx, 0
    mov rax, 15
xxx:
    mov cl, byte ptr [rsp-924+rdi]
    cmp cl, byte ptr [length+rsi]
    je xx
    add rdi, 1
    mov rsi, 0
    jmp xxx
    
xx:
    add rdi, 1
    add rsi, 1
    cmp rsi,rax
    jne xxx

    # 计算length 
    mov rax, 0
    mov rcx, 0
    add rdi, 1
xxxx:
    # 一个字节一个字节读，计算长度
    # 如123等于 (('1'-'0')*10+('2'-'0'))*10+('3'-'0')
    imul eax, 10 
    mov cl, byte ptr [rsp - 924+rdi]
    sub cl, '0'
    add rax,  rcx
    add rdi, 1
    cmp byte ptr [rsp-924+rdi], '\r'
    jne xxxx

.section .data
msg: .asciz "HTTP/1.0 200 OK\r\n\r\n" 
length: .asciz "Content-Length:"


```


#### 完整代码

```
.intel_syntax noprefix
.globl _start

.section .text


_start:

    mov qword ptr [rsp - 24], 0         # fd
    mov word ptr [rsp - 16], 2          # AF_INET (2 bytes)
    mov word ptr [rsp - 14], 0x5000     # Port 80 (network byte order: 0x5000)
    mov dword ptr [rsp - 12], 0x0000000 # 127.0.0.1 (network byte order: 0x7F000001)
    mov qword ptr [rsp - 8], 0          # sin_zero (8 bytes padding

    mov rdi, 2
    mov rsi, 1
    mov rdx, 0
    mov rax, 41     # create socket
    syscall 

    # 将文件描述符保存在栈上
    mov qword ptr [rsp - 24], rax

    mov rdi, [rsp - 24]      # fd
    lea rsi, [rsp-16]
    mov rdx, 16
    mov rax, 49     # bind
    syscall 

    mov rdi, [rsp - 24]      # fd
    mov rsi, 0
    mov rax, 50              # listen
    syscall 

    mov word  ptr [rsp - 324], 0 # 保存文件内容
    mov word  ptr [rsp - 924], 0 # 保存读取的request
    mov qword ptr [rsp - 932], 0 # 备用
    mov qword ptr [rsp - 940], 0
    

handle:

    mov rdi, [rsp - 24]      # fd
    mov rsi, 0               # use 0 to represent NULL
    mov rdx, 0
    mov rax, 43              # accept
    syscall 

    mov rbx, rax

    mov rax, 57 # fork
    syscall

    # 这里需要判断当前进程是子进程还是主进程
    cmp rax, 0
    je child

    mov rdi, rbx
    mov rax, 3
    syscall         # close
    
    jmp handle

child:
    mov rdi, [rsp - 24]
    mov rax, 3
    syscall         # close
 
    mov rdi, rbx
    lea rsi, [rsp-924]
    mov rdx, 600
    mov rax, 0      
    syscall         # read

    cmp byte ptr [rsp - 924], 'G'
    je handle_get
    cmp byte ptr [rsp - 924], 'P'
    je handle_post
    jmp _end

    
handle_post:

    # 截取file name
    mov rax,0
loop1:
    add rax, 1
    cmp byte ptr [rsp-919+rax], ' '
    jne loop1
    mov byte ptr [rsp-919+rax], 0

    # 获取length所处的位置
    mov rdi, 0
    mov rsi, 0 
    mov rcx, 0
    mov rax, 15
xxx:
    mov cl, byte ptr [rsp-924+rdi]
    cmp cl, byte ptr [length+rsi]
    je xx
    add rdi, 1
    mov rsi, 0
    jmp xxx
    
xx:
    add rdi, 1
    add rsi, 1
    cmp rsi,rax
    jne xxx

    # 计算length 
    mov rax, 0
    mov rcx, 0
    add rdi, 1
xxxx:
    imul eax, 10 
    mov cl, byte ptr [rsp - 924+rdi]
    sub cl, '0'
    add rax,  rcx
    add rdi, 1
    cmp byte ptr [rsp-924+rdi], '\r'
    jne xxxx

    # 获取到了长度存储在rax中
    mov qword ptr [rsp - 932], rax
    mov qword ptr [rsp - 940], rdi

    # open file
    lea rdi, [rsp - 919]
    mov rsi, 0x41
    mov rdx, 0777
    mov rax, 2
    syscall

    mov rcx, [rsp - 940]
    add rcx, 4
    mov rdi, rax
    lea rsi, [rsp-924+rcx]
    mov rdx, [rsp - 932]
    mov rax, 1 
    syscall         # write

    mov rax, 3
    syscall         # close

    mov rdi, rbx
    lea rsi, [msg]
    mov rdx, 19
    mov rax, 1 
    syscall         # write msg

    jmp _end




handle_get:

    # 截取file name
    mov rax,0
loop2:
    add rax, 1
    cmp byte ptr [rsp-920+rax], ' '
    jne loop2
    mov byte ptr [rsp-920+rax], 0

    
# 根据GET或者POST选择不同的策略

    
    # open file
    lea rdi, [rsp - 920]
    mov rsi, 0
    mov rdx, 16
    mov rax, 2
    syscall


    # read file
    mov rdi, rax
    lea rsi, [rsp-324]
    mov rdx, 300
    mov rax, 0  
    syscall
    mov qword ptr [rsp-932], rax

    mov rax, 3
    syscall         # close


    mov rdi, rbx
    lea rsi, [msg]
    mov rdx, 19
    mov rax, 1 
    syscall         # write

    mov rdi, rbx
    lea rsi, [rsp-324]
    mov rdx, [rsp-932]
    mov rax, 1 
    syscall         # write

    mov rdi, rbx
    mov rax, 3
    syscall         # close


_end:
    mov rdi, 0
    mov rax, 60     # SYS_exit
    syscall

.section .data
msg: .asciz "HTTP/1.0 200 OK\r\n\r\n" 
length: .asciz "Content-Length:"

```

## **Playing With Programs**

### Program Misuse

这一关讲述了用各种命令获取flag。

cat、more、less、tail、head、vim、emacs、nano，这些命令都可以直接读取。

这里面，我对`emacs`确实稍微没那么了解，使用也很少，它其实也是一种文本编辑器。

紧随其后的其他命令就没那么直接了。

#### rev

`rev`命令，其实就是`reverse`，它会把文件内容倒着输出，**每一行内容都反向输出**。那么我们要做的就是将flag反向输出的内容保存到一个文件里，再反向输出一次，就可以得到flag了。

```
/challenge/rev /flag > ./galf
/challenge/rev ./galf
```

#### od、hd、xxd

这三个都是进制查看工具，hd即`hexdump`应用稍微广泛一些。

od命令，全称为`octal dump`。是Linux中用于以八进制和其他格式（如十六进制、十进制和ASCII）显示文件内容的工具。这个命令在查看通常不易读的文件，如编译过的二进制文件时非常有用。

常用选项

+ -b：以单字节八进制显示文件内容。

+ -c：以ASCII字符显示文件内容。

+ -x：将输入转换为十六进制格式。

+ -d：将输入转换为十进制格式。

+ -j：跳过文件的初始字节数。

+ -N：限制输出的字节数。

+ -w：自定义输出的宽度。

+ -v：输出重复的值。

```
/challenge/od -c /flag
```



hd，全名`hexdump`，功能和od似乎差不多，也是用各种进制的形式显示文件内容。

```
/challenge/hd -c /flag
```

xxd同理。

```
/challenge/xxd /flag
```

#### base32、base64

这两个命令都是用来加解码的。只是编码的格式不一样而已。

```
/challenge/base32 /flag > ./galf
/challenge/base32 -d ./galf

/challenge/base64 /flag > ./galf
/challenge/base64 -d ./galf

```

#### split

`split`命令可以将大文件分割为多个小文件，默认情况下会创建每一千行一个新文件。

+ -l 指定分割行数

+ -b 指定分割的文件大小

+ -d 将分割后的文件名以数字结尾

```
/challenge/split -l 1 /flag xx
cat xxaa
```

这个命令将文件按照每行进行分割，分隔到前缀为xx的多个文件之中。

#### gzip、bzip2、zip、tar

这几个命令就很眼熟了，都是用于压缩和解压的。这几个命令又是如何运用到读取文件的呢？

gzip，gzip进行压缩时，会默认不保留原文件，压缩后的文件以`.gz`后缀结尾，同时继承原文件的权限信息。

+ -d：解压缩 .gz 文件。相当于使用 gunzip 命令。
+ -k：保留原始文件，不删除。
+ -r：递归压缩目录下的所有文件。
+ -v：显示详细的压缩或解压缩过程。
+ -c: 输出到标准输出

```
gzip /flag
# -d 进行解压，-c则输出到标准输出，从而显示文件内容
gzip -d -c /flag.gz
```

### SQL

这一章的内容主要讲述了sql的基础语法。这里我只总结一部分有意思的内容。

1、`substr`的用法

包含三个参数：目标字符串，起始（需要注意的是，sql字符串的第一个字符下标为1），长度。

```
select text from table where substr(text,1,3)="pwn"

```

2、`sqlite_master`表

在SQLite数据库中，`sqlite_master`是一个存储数据库元信息的特殊表，它会在每个SQLite数据库创建时自动生成。这个表包含了数据库中所有其他表、索引、触发器和视图的描述信息。

这个表包含的字段有：

+ type: 记录项目的类型，如table、index、view、trigger。

+ name: 记录项目的名称，如表名、索引名等。

+ tbl_name: 记录所从属的表名，对于表来说，该列就是表名本身。

+ rootpage: 记录项目在数据库页中存储的编号。对于视图和触发器，该列值为0或者NULL。

+ sql: 记录创建该项目的SQL语句。

SQL的语法我从一开始进入大学学习计算机时就有接触，但是没有什么很高深的应用，只是做一些简单的增删改查以及数据库的管理。但是在安全领域以及数据库落地，SQL的使用还是很重要的。如**SQL注入**，以及**高效mysql**。