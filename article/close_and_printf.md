你或许已经很熟悉C/C++的输入输出了，那么你能辨别清楚以下这些概念吗？
> 文件描述符，文件句柄，文件指针

如果上面的难不倒你，试一下这个问题：
> fclose(stdout)和close(1)有什么区别？
> fclose(stdout)或者close(1)之后printf还有效吗？

事实上只要理解了开篇提出的那几个概念，就能回答后面的问题。

对文件的操作本质上都是由系统来完成的。而不同的操作系统管理文件的方式显然不同。
linux是通过文件描述符`int fd`，而windows是通过文件句柄`HANDLE`。但站在用户的角度看来，我并不需要关心它到底是描述符还是句柄，
它们都是对文件的抽象罢了，于是有了C的`FILE *`指针。

所以可以说文件描述符，文件句柄，文件指针都是对文件的抽象。而前两者是操作系统给出的概念，而后者则是C语言上的概念，可以跨平台使用。

`close(int fd)`是Linux下的系统调用，而且window下`fclose(stdout)`是非法语句，所以对于后面两个问题，我们将在Linux下探究。（受限能力和时间，我没能在windows上做类似的探索，十分欢迎有志者做出补充或指正）

Linux下每个进程创建的时候都会打开三个文件：stdin，stdout和stderr。它们的文件描述符对应为0，1，2。
系统分配文件描述符时总是分配最小可用的值。如果当前进程只使用了0，1，2这三个文件描述符，那下次分配的将是3。如果当前进程只使用了0，1，2，3，5这五个文件描述符，那下次分配的将是4。
你可以使用以下代码来验证这一点：

```
close(1);//关闭文件描述符1
int fd=open("***",O_WRONLY|O_CREAT,0666);//函数的具体调用方法自行查找，如果文件打开成功，则返回的fd等于1
//现在如果我们向标准输出写入字符则会直接写入文件（如果打开的方式正确），而屏幕上什么也没有

//上面两行其实等价于下面这一行
stdout=fopen(***,**);

```
`open`和`close`是匹配的，返回的是`int`类型的描述符，`fopen`和`fclose`是匹配的，返回的是`FILE*`类型。

显然`fclose`最终调用了`close`,`fopen`调用了`open`,而`FILE`结构体必然要包含描述符信息。
因为`close,open`是Linux的系统调用，编程语言对它们只能做向上的封装而不可能绕过它们。

那为什么要封装出另一套接口呢？只用`open, close`简洁又明了，少掉一点头发不好吗？
原因无非这么几种啦：
1. 跨平台
2. 用起来不方便
3. 效率不高

主要说一下第三点：以`write`和`printf`为例
`write`系统调用是软中断，频繁调用，会导致频繁陷入内核态，这样的效率不是很高，而`printf`实际是向用户空间的IO缓冲写，在满足条件的情况下才会调用`write`系统调用，减少IO次数，提高效率。
也就是说你可能调用了`printf`但屏幕上不会立即输出字符！看看下面的代码会输出什么
```
char s[10]="linux\n";
close(1);
fd=open("./test3.txt",O_WRONLY|O_CREAT,0666);//成功的情况下fd应该是1
if(fd!=-1){
    write(fd,s,2);
    printf("%d",fd);
    write(fd,s+2,2);
    close(fd);
}；
return 0；
```
surprise!
答案是`linu`,前后的write都输出成功了，但中间的printf却失败了！

现在我们差不多可以解释那两个问题了：
1. fclose(stdout)和close(1)有什么区别？
   前者是在关闭一个文件，后者是在关闭一个描述符。`fclose(stdout)`等价于`close(stdout对应的描述符)`。注意stdout对应描述符不一定是1！
2. fclose(stdout)或者close(1)之后printf还有效吗？
   当然是无效了，`printf`会返回错误值-1

## 引用
本文部分参考以下资料:
[fclose(stdout)和close(1)的区别](https://blog.csdn.net/wangzuxi/article/details/43445599)
[printf()详解之终极无惑](https://cloud.tencent.com/developer/article/1176463)
