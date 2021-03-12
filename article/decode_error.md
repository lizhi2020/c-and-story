---
title: 编码那些破事
date: 2020-12-04 18:16:17
categories:
- 探索
tags:
- 编码
---

照例一个问题引入，下面代码保存的文本文件有问题吗？
```
char s[]="linux\n";
fd=open("./test3.txt",O_WRONLY|O_CREAT,0666);
write(fd,s,sizeof(s));
close(fd);
```
答案是有问题，用vscode打开文件会发现有乱码。
原因很简单，c语言的字符串最后会自动添加一个'\0'字符，所以你无意中将一个不可显示字符写入了文本文件。

这个乱码问题是好理解的。

但实际编程过程中莫名其妙的出现乱码问题，就很需要一番折腾了。

所谓编码，就是将字符映射到一个数字或者说二进制值。
所谓解码，就是将一个数字或者二进制值映射回字符。

## 分类
源文件使用的字符集被称为编码字符集，也就是写代码的时候使用的字符集。
程序中的字符或者字符串使用的字符集被称为运行字符集，也就是程序运行后使用的字符集。

## C语言的编码
在C语言中字符有两种，一种是窄字符，另一种是宽字符。
*多字节字符？*
- 只有 char 类型的窄字符才使用 ASCII 编码
- char 类型的窄字符串、宽字符和宽字符串都不使用 ASCII 编码！

wcout 不能用来输出 string对象。
cout 也不能用来输出 wstring 对象。
cout是用来输出char的，如果用cout输出wchar，只会输出地址，要使用wcout输出wchar。
wcout默认是不输出中文的，要想输出中文，要先加入`setlocale( LC_ALL, "chs" )`或`wcout.imbue(std::locale("chs"))`。

*注：wprintf是C的标准库函数，但wcout不是C++的标准成员，C++中的 L"……" 是宽字符，却未必是unicode字符，这与编译器实现相关。*

## ANSI编码
注意ANSI不是ANSII编码！
ANSI 是兼容 ASCII 在不同国家推行的字符集的总称，不是单独指某个字符集。而是在不同的系统中，ANSI表示不同的编码。
“ANSI编码”只存在于Windows系统。
**那么Windows系统是如何区分ANSI背后的真实编码的呢？**
微软用一个叫“Windows code pages”（在命令行下执行chcp命令可以查看当前code page的值）的值来判断系统默认编码，比如：简体中文的code page值为936（它表示GBK编码，win95之前表示GB2312，详见：Microsoft Windows' Code Page 936），繁体中文的code page值为950（表示Big-5编码）。

## 常见IDE，编辑器使用编码
- 绝大多数的编译器，如：Xcode、Sublime、Text、Gedit、Vim等，在创建源文件时一般也默认使用 UTF-8 编码。
- 奇葩 Visual Studio 它默认使用本地编码来创建源文件。
> 所谓本地编码，就是像 GBK、Big5、Shift-JIS 等这样的国家编码（地区编码）；针对不同国家发行的操作系统，默认的本地编码一般不同。简体中文本的 Windows 默认的本地编码是 GBK。

## 程序编译时的编码
1. 微软编译器使用本地编码来保存这些字符。不同地区的 Windows 版本默认的本地编码不一样，所以，同样的窄字符串在不同的 Windows 版本下使用的编码也不一样。对于简体中文版的 Windows，使用的是 GBK 编码。
2. GCC、LLVM/Clang 编译器使用和源文件相同的编码来保存这些字符：如果源文件使用的是 UTF-8 编码，那么这些字符也使用 UTF-8 编码；如果源文件使用的是 GBK 编码，那么这些字符也使用 GBK 编码。

## 什么是内码

## 什么是多字节字符集

　　在最初的时候，Internet上只有一种字符集——ANSI的ASCII字符集，它使用7 bits来表示一个 字符，总共表示128个字符，其中包括了 英文字母、数字、标点符号等常用字符。之后，又进行扩展，使用8 bits表示一个字符，可以表示256个字符，主要在原来的7 bits字符集的基础上加入了一些特殊符号。后来，由于各国语言的加入，ASCII已经不能满足信息交流的需要，为了能够表示其它国家的文字，各国在 ASCII的基础上制定了自己的字符集，这些从ANSI标准派生的字符集被习惯的统称为ANSI字符集，它们正式的名称应该是MBCS(Multi-Byte Chactacter System，即多字节字符系统)。这些派生字符集的特点是以ASCII 127 bits为基础，兼容ASCII 127，他们使用大于128的编码作为一个Leading Byte，紧跟在Leading Byte后的第二（甚至第三）个字符与 Leading Byte一起作为实际的编码。这样的字符集有很多，我们常见的GB-2312就是其中之一。

## 什么是Unicode字符集

　　Unicode的学名 是"Universal Multiple-Octet Coded Character Set"，简称为UCS。UCS可以看作是"Unicode Character Set"的缩写。UCS只是规定如何编码，并没有规定如何传输、保存这个编码。UTF是“UCS Transformation Format”的缩写。

　　Unicode字符集有多种编码形式，它固定使用16 bits（两个字节、一个字）来表示一个字符，共可以表示65536个字符。将世界上几乎所有语言的常用字符收录其中，方便了信息交流。标准的Unicode称为UTF-16。后来为了双字节的Unicode能够在现存的处理单字节的系统上正确传输，出现了UTF-8(注意UTF-8是编码，它属于Unicode字符集)，使用类似MBCS的方式对Unicode进行编码。UTF-8以字节为编码单元，没有字节序的问题。UTF-16以两个字节为编码单元。

　　UTF-16包括三种：UTF-16，UTF-16BE(Big Endian)，UTF-16LE(Little Endian)，UTF-16需要通过在文件开头以名为BOM(Byte Order Mark)的字符来表明文件是Big Endian还是Little Endian。Unicode规范中推荐的标记字节顺序的方法是BOM(Byte Order Mark)。在UCS编码中有一个叫做"ZERO WIDTH NO-BREAK SPACE"的字符，它的编码是FEFF。而FFFE在UCS中是不存在的字符，所以不应该出现在实际传输中。UCS规范建议我们在传输字节流前，先传输字符"ZERO WIDTH NO-BREAK SPACE"。这样如果接收者收到FEFF，就表明这个字节流是Big-Endian的；如果收到FFFE，就表明这个字节流是Little-Endian的。因此字符"ZERO WIDTH NO-BREAK SPACE"又被称作BOM。

　　UTF-8不需要BOM来表明字节顺序，但可以用BOM来表明编码方式。字符"ZERO WIDTH NO-BREAK SPACE"的UTF-8编码是EF BB BF（读者可以用我们前面介绍的编码方法验证一下）。所以如果接收者收到以EF BB BF开头的字节流，就知道这是UTF-8编码了

## MBS和WCS转换函数
- mbstowcs
- wcstombs
- int WideCharToMultiByte(
    UINT                               CodePage,
    DWORD                              dwFlags,
    _In_NLS_string_(cchWideChar)LPCWCH lpWideCharStr,
    int                                cchWideChar,
    LPSTR                              lpMultiByteStr,
    int                                cbMultiByte,
    LPCCH                              lpDefaultChar,
    LPBOOL                             lpUsedDefaultChar
    );
- int MultiByteToWideChar(
    UINT                              CodePage,
    DWORD                             dwFlags,
    _In_NLS_string_(cbMultiByte)LPCCH lpMultiByteStr,
    int                               cbMultiByte,
    LPWSTR                            lpWideCharStr,
    int                               cchWideChar
    );

## Windows平台下实验发现
样例代码：
```
    WCHAR buf[10] = TEXT("hhhh李智");
	printf("%d\n",sizeof(buf));
	for (int i = 0; i < 10; i++) {
		printf("%x ", (char)*((char*)buf+i));
	}
```
第一行输出为20，表明一个WCHAR固定占两个字节
第二行输出`68 0 68 0 68 0 68 0 4e 67`,0x68是h的ascii码,0x674e是`李`的unicode码，采用小端字节序存放。
样例代码：
```
    char temp[20] = "hhhh李智";
	for (int i = 0; i < 10; i++) {
		printf("%x ", (char)temp[i]);
	}
```
此时输出`68 68 68 68 ffffffc0 ffffffee ffffffd6 ffffffc7 0 0`，`fff`的出现是因为传入的char变量最高位为一，printf自动将其拓展了。
实际值应为0xc0ee,注意此时又变成了大端的GBK。

## 注意控制台的编码页Code page
当我们使用标准输出在控制台显示字符的时候，可以认为是向控制台写入字符数据，显然控制台本身也是有编码字符集的！
所以我们最终输出的字符还应当满足控制台的编码字符集！
**对于传统字符串输出**`printf("%s\n", str);`
程序运行时,直接将二进制文件中存储的那串数字丢进输出流。
char str[]中的字符串保存在源文件中是使用的是GBK编码,存储在二进制文件中还是GBK,到控制台的输出环境也是GBK!三者一致,自然输出正常.(当然,如果你修改三者中任一的一个编码,输出结果都会不一样)

对于宽字符串，源文件中依然使用GBK，在二进制文件中用的是Unicode编码(注意：不是UTF-XX)，那么输出到控制台时就必须进行转换！
我们可以使用函数`printf("%ls",wcs)`;或`wprintf(wcs)`。它们会一次处理两个byte（因为一个宽字符占两个byte）,根据一定条件转换转成本地区域编码，然后再将转换之后的编码传递给控制台。
那么这本地区域编码程序是怎么得到就成关键中的关键了。这时咱们来看看setlocale这个函数吧。
setlocale是用来程序运行时,设置当前的区域信息。
值得注意是: 在所有C程序启动前,locale的默认设置为`setlocale(LC_ALL,"C")`
所谓的C环境的定义：
> The "C" locale is the minimal locale. It is a rather neutral locale which has the same settings across all systems and compilers, and therefore the exact results of a program using this locale are predictable. This is the locale used by default on all C programs.

其实就是C/C++为了保持兼容性，选择了一个适用于所有编译器和系统的最小本地编码。其实就是在默认情况下，几乎不会进行编码转换的意思：
**例子**：现在有一个`WCHAR buf="h中文"`；那么他在二进制文件中的布局应该是`68 00 2d 4e 87 65`,`h`的unicode码是0x0068，`中`的Unicode码是`0x4e2d`,`文`的Unicode码是`0x6587`,一切正常。当我们使用默认的C环境时，它可以把`h`输出为0x68，因为所有的编码都兼容ANSI编码，但是不能编码`中`,因为不知道当前的控制台默认编码是什么，如果在台湾，那控制台编码可能是Big5，如果在中国大陆，那应该是GBK。所以在不特意设置的情况下，`printf("%ls",wcs)`;或`wprintf(wcs)`都是无法输出中文的。

## 参考
* [UTF-8 and Unicode FAQ for Unix/Linux](https://www.cl.cam.ac.uk/~mgk25/unicode.html)[英文译文](https://blog.csdn.net/lovekatherine/article/details/1765903)
* [浅谈C中的wprintf和宽字符显示](https://blog.csdn.net/lovekatherine/article/details/1868724?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.control)
* 同样基于上文的拓展[控制台程序的中文输出乱码问题,printf,wprintf与setlocale](https://www.cnblogs.com/dejavu/archive/2012/09/16/2687586.html)