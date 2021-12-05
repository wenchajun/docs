https://blog.csdn.net/u011630575/article/details/52151995

在shell脚本中，默认情况下，总是有三个文件处于打开状态，标准输入(键盘输入)、标准输出（输出到屏幕）、标准错误（也是输出到屏幕），它们分别对应的文件描述符是0，1，2 。

>  默认为标准输出重定向，与 1> 相同
>  2>&1  意思是把 标准错误输出 重定向到 标准输出.

&>file  意思是把标准输出 和 标准错误输出 都重定向到文件file中

/dev/null是一个文件，这个文件比较特殊，所有传给它的东西它都丢弃掉
举例：

当前目录只有一个文件 a.txt.
[root@redhat box]# ls
a.txt
[root@redhat box]# ls a.txt b.txt
ls: b.txt: No such file or directory 由于没有b.txt这个文件, 于是返回错误值, 这就是所谓的2输出
a.txt 而这个就是所谓的1输出
再接着看:

[root@redhat box]# ls a.txt b.txt 1>file.out 2>file.err
执行后,没有任何返回值. 原因是, 返回值都重定向到相应的文件中了,而不再前端显示
[root@redhat box]# cat file.out
a.txt
[root@redhat box]# cat file.err
ls: b.txt: No such file or directory
一般来说, "1>" 通常可以省略成 ">".
即可以把如上命令写成: ls a.txt b.txt >file.out 2>file.err
有了这些认识才能理解 "1>&2" 和 "2>&1".
1>&2 正确返回值传递给2输出通道 &2表示2输出通道
如果此处错写成 1>2, 就表示把1输出重定向到文件2中.
2>&1 错误返回值传递给1输出通道, 同样&1表示1输出通道.
举个例子.
[root@redhat box]# ls a.txt b.txt 1>file.out 2>&1
[root@redhat box]# cat file.out
ls: b.txt: No such file or directory
a.txt
现在, 正确的输出和错误的输出都定向到了file.out这个文件中, 而不显示在前端.
补充下, 输出不只1和2, 还有其他的类型, 这两种只是最常用和最基本的.

例如：
rm -f $(find / -name core) &> /dev/null，/dev/null是一个文件，这个文件比较特殊，所有传给它的东西它都丢弃掉。

例如：
注意，为了方便理解，必须设置一个环境使得执行grep da *命令会有正常输出和错误输出，然后分别使用下面的命令生成三个文件：
grep da * > greplog1
grep da * > greplog2 1>&2   
grep da * > greplog3 2>&1  //grep da * 2> greplog4 1>&2 结果一样
#查看greplog1会发现里面只有正常输出内容
#查看greplog2会发现里面什么都没有#查看greplog3会发现里面既有正常输出内容又有错误输出内容

