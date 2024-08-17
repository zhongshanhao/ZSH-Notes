## linux

### linux的权限管理

使用`ls -l`查看文件的权限信息

[![linux.png](https://i.postimg.cc/GhVjG33x/linux.png)](https://postimg.cc/vc79dsV4)

可以从上图第一列中得知文件的权限信息

`drwxrwxr-x`中第一个字符表示文件的类型，后面3个字符一组，分为3组，分别对应着文件的所属用户、用户所在组、其他用户对文件的权限信息。

文件类型

- d：目录文件
- -：代表普通文件
- l：代表软链接（可以认为是window中的快捷方式）

权限信息

对于普通文件来说

- r：代表文件可读
- w：代表修改文件内容
- x：表示可以执行

对于目录来说

- r：可以查看（ls）目录下文件
- w：可以创建可删除和修改目录下文件
- x：可以使用cd进入目录

![img](https://www.runoob.com/wp-content/uploads/2014/06/file-llls22.jpg)

chmod

```
chmod +r 文件名		// 给当前用户、用户所在组、其他用户加上r权限
chmod +w 文件名  	  // 给当前用户加上w权限
chmod +x 文件名		// 给当前用户、用户所在组、其他用户加上x权限
chmod 777 文件名     // 加上所有权限
```

### 常用命令

查看进程

```c
// 查看运行的java线程
ps -ef | grep java
// jdk自带
jps 

// 杀死进程
kill pid

// 查看进程信息，某进程cpu和内存的使用情况
top
// 查看java进程使用的cpu和内存的情况
top | grep java
// 查看某进程下的线程
top -H -p pid

// java监视和管理控制台，选择要查看的进程，可以看到进程的内存、cpu使用、该进程的线程等信息
jconsole

// 查看当前jvm运行线程的状态信息，GC线程、查看当前是否死锁
jstack pid
```

​	目录切换命令

```
cd usr		 切换到该目录下的usr目录
cd ..			切换到上一层目录
cd /			切换到系统根目录
cd ~		   切换到用户主目录（同cd后面不带任何参数）
cd -		   	切换到上一个操作所在目录
```

目录的操作命令（增删改查）

```
mkdir usr		增加usr目录
mv 目录名称 新目录名称
mv 目录名称 目录的新位置
cp -r 目录名称 目录拷贝的目标位置			：-r表示对目录递归拷贝
rm [-rf] 目录
```

**grep**（global search regular expression(RE) and print out the line，全面搜索正则表达式并把行打印出来）是一种强大的文本搜索工具，它能使用正则表达式搜索文本，并把匹配的行打印出来。

示例

```
grep -A 4 wikipedia 密码文件.txt 
```

就是搜索密码文件，找到匹配“wikipedia”字串的行，显示该行后面紧跟的4行。

-B num

搜索文本，找到匹配的行，并把该行的前num行也打印出来

-C num

搜索文本，找到匹配的行，并把该行的前后的num行也打印出来

pwd 打印当前工作目录



资源信息（内存和磁盘）

top

top命令是Linux下常用的性能分析工具，能够实时显示系统中各个进程的资源占用状况

![image-20210329110742600](C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210329110742600.png)

free

查看物理内存和swap交换区空间的使用情况

![image-20210329110657289](C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210329110657289.png)

> swap 分区通常被称为交换分区，这是一块特殊的硬盘空间，即当实际内存不够用的时候，操作系统会从内存中取出一部分暂时不用的数据，放在交换分区中，从而为当前运行的程序腾出足够的内存空间。当程序需要该数据时，从磁盘中把它给拷出来。

df -h 

查看各磁盘分区的使用情况 

![image-20210329111825099](C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210329111825099.png)

目录文件操作

```
cd
cp
pwd
ls
mkdir
```

网络

```
ifconfig	// 查看网卡配置信息

netstat     // 查看网络连接	
netstat -r   // 查看路由表
netstat -tunlp // 查看端口占用情况

ping ip    // 测试当前主机是否与目标主机联通
```

#### netstat

```netstat -tunlp```用于显示tcp、udp的端口和进程等相关情况。

netstat查看端口占用语法格式

```
netstat -tunlp | grep 端口号
```

- -t (tcp) 仅显示tcp相关选项
- -u (udp)仅显示udp相关选项
- -n 拒绝显示别名，能显示数字的全部转化为数字
- -l 仅列出在Listen(监听)的服务状态
- -p 显示建立相关链接的程序名

使用示例

![image-20210402144246261](C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210402144246261.png)

#### lsof

lsof（list open files）是一个列出当前系统已打开文件的工具。

lsof 查看端口占用情况

```
lsof -i:端口号
```

使用示例

![image-20210402145251400](C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210402145251400.png)

### 统计词频

words.txt

```
aa bb cc
aa bb
```

方法1

```cmd
cat words.txt | tr -s " " "\n"| sort | uniq -c | sort -r| awk '{print $2" "$1}'
```

方法2

```bash
cat words.txt | awk '{
    for(i=1;i<=NF;i++){
        print $i
    }
}' | sort | uniq -c | sort -r| awk '{print $2" "$1}'
```

### Java

jmap -heap pid 查看堆内存

jmap -dump:format=b,file=file.dump pid  把Java内存镜像dump下来，然后用jvisualvm分析

jvisualvm 可以看到java内存的使用情况，那些类占用了多少内存等等

[必须要会的JVM性能监测工具（JVisualVM)](https://zhuanlan.zhihu.com/p/339676111)


