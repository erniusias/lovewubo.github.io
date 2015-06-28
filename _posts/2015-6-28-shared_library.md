---
layout: post
title: linux共享库的版本控制和使用
category: blog
description: 
---


##linux约定
经常看到linux中，共享库的名字后面跟了一串数字，比如：`libperl.so.5.18.2`。其实就是版本号，作用是为了更加方便的管理动态库，比如升级。往往系统中存在一个库的多个版本，那么Linux 系统如何控制多个版本的问题？Window之前没有处理好，为此专门有个名词来形容这个问题：“Dll hell”，其严重影响软件的升级和维护。“Dll hell”是指windows上动态库的新版本覆盖了旧版本，﻿﻿但是却不兼容老版本，所以程序升级之后，动态库更新导致程序运行不起来。在Linux操作系统下也有同样的问题，那么它是怎么解决的呢？ 

Linux引入了一套机制，如果遵守这个机制就可以避免这个问题。 但是这只事一个约定，不是强制的。通常建议用户遵守这个约定，否则也会出现Linux版的“Dll hell”问题。 下面来介绍一个这个机制。 这个机制是通过文件名，来控制动态库（shared library）的版本。

Linux上的shared library有三个名字，分别是：

* **共享库本身的文件名（real name)**

    其通常包含完整的版本号，比如：libmath.so.1.1.1234 。lib是Linux库的约定前缀，math是共享库名字，so是共享库的后缀名，1.1.1234的是共享库的版本号，由`主版本号+小版本号+build号`组成。主版本号，代表当前动态库的版本，如果共享库的接口发生变化，那么这个版本号就要加1；后面的两个版本号（`小版本号`和 `build号`）是用来指示库的更新迭代号，表示在接口没有改变的情况下，由于需求发生变化等因素，开发的新代码。

 

* **共享库的soname（Short for shared object name）**

    用来告诉应用程序，在加载共享库的时候，应该使用的文件名。其格式为`lib + math + .so + (major version number)`
    其只包含主版本号，换句话说，也就是只要共享库的接口没有变，soname就能与real name保持一致，因为主版本号一样。所以在库的real name的`小版本号`和 `build号`发生改变时，应用程序仍然可以通过soname得知，要使用的是哪个real name。不明白？等会给个例子来说明。

* **共享库的链接名（link name）**

	是专门为应用程序在编译时的链接阶段而用的名字。这个名字就是lib + math +.so ,比如libmath.so。其是不带任何版本信息的。在共享库的编译过程中，编译器将生成一个共享库及real name，同时将共享库的soname写在共享库文件里的文件头里面。可以用命令`readelf -d sharelibrary | grep soname`查看。
    在应用程序引用共享库时，链接选项里面用的是共享库的link name。通过link名字找到对应的real name动态库，并且把其中的soname提取出来，写在应用程序自己的文件头的共享库字段里面。当应用程序运行时，就会通过soname，结合动态链接程序（ld.so），在给定的路径下加载real name的共享库。

##如何使用
这里我们写了一个简单的例子，包含了三个文件，分别是：

* test.h

```c
#ifndef _TEST_H_
#define _TEST_H_
void print_hello();
#endif
```

* test.c

```c
#include <stdio.h>
#include "test.h"

void print_hello()
{
    printf("this is a test for shared lib");
}
```

* main.c

```c
#include <stdio.h>
#include "test.h"

int main()
{
	print_hello();	
	return 0;
}
```

首先编译共享库：

```shell
gcc -fPIC -o test.o -c test.c
gcc -shared -Wl,-soname,libtest.so.0 -o libtest.so.0.0.0 test.o
```
然后就生成了libtest.so.0.0.0,这就是库的real name。另外，链接选项里面的`-Wl,-soname,libtest.so.0`告诉编译器，库的soname是libtest.so.0,我们可以看到real name的头文件里面，已经包含了这样的信息：

```shell
readelf -d libtest.so.0.0.0 | grep soname
 0x0000000e (SONAME) Library soname: [libtest.so.0]
```

如果没有指定soname，库的头文件里面是没有这个字段的。
有了库以后，下一步是链接到应用程序里面，我们需要这样写：

```shell
gcc -c -o main.o main.c
gcc -L. -o main main.o -ltest
```
但是会报错“cannot find -ltest”。这里因为，链接选项制定的是link name，而根据linux的规则，此目录下面并没有libtest.so文件，所以需要先生成link name文件。

```shell
ln -s libtest.so.0.0.0  libtest.so.0
ln -s libtest.so.0 libtest.so
```

或者一步到位：

```shell
ln -s libtest.so.0.0.0 libtest.so
```

然后再进行应用程序的编译：**gcc -L. -o main main.o -ltest**

ok，生成了我们需要的main可执行程序。查看一下，其引用的共享库：

```
ldd main
	linux-gate.so.1 =>  (0xb76fb000)
	libtest.so.0 => not found
	libc.so.6 => /lib/i386-linux-gnu/libc.so.6 (0xb752e000)
	/lib/ld-linux.so.2 (0xb76fc000)

```

没错，链接时，应用程序需要的库正是我们指定的soname，而不是link name或者real name。所以应用程序正是通过soname去寻找真正的real name库。这有什么好处吗？答案后面揭晓。另外，上面的输出中，我们发现，libtest.so.0是`not found`的状态。为什么呢？因为这个库的当前所在路径并不在链接程序（ld.so)的搜索路径之中，所以无法找到。如何解决？这篇文章就不多说了，这里提供几个方案：

* **改变LD_LIBRARY_PATH**
	
    `export LD_LIBRARY_PATH=/home/bow/all/program/test/lib_version_test:$LD_LIBRARY_PATH`

	这里`/home/bow/all/program/test/lib_version_test`是共享库的路径。虽然改变LD_LIBRARY_PATH能达到目的，但是不推荐使用，因为这是一个全局的变量，其他应用程序可能受此影响，导致各种库的覆盖问题。如果要清楚这个全局变量，使用命令**unset LD_LIBRARY_PATH**

* **用rpath**

	在编译应用程序时，利用rpath指定加载路径。
	`gcc -L. -Wl,-rpath=/home/bow/all/program/test/lib_version_test -o test main.o -ltest`
	这样，虽然避免了各种路径找不到的问题，但是也失去了灵活性。因为库的路径被定死了。

* **改变ld.so.conf**

	将路径添加到此文件，然后使用`ldconfig`更新加载程序的cache。
	可以使用命令`ldconfig -p`查看当前所有库的soname->real name的对应关系信息

这里我们选择最后一种方式。
```
bow@bow-Aspire-4752:vim /etc/ld.so.conf
##添加路径到文件末：/home/bow/all/program/test/lib_version_test
bow@bow-Aspire-4752:ldconfig
bow@bow-Aspire-4752:~/all/program/test/lib_version_test$ ./main
this is a test for shared lib
```


最后说一下，应用程序在编译链接和运行加载时，库的搜索路径的先后顺序。

* 编译链接时，查找顺序
	* /usr/local/lib
	* /usr/lib
	* 用-L指定的路径，按命令行里面的顺序依次查找
* 运行加载时的顺序
	* 可执行程序指定的的DT_RPATH
	* LD_LIBRARY_PATH. 但是如果使用了setuid/setgid，由于安全因素，此路径将被忽略.
	* 可执行程序指定的的DT_RUNPATH. 但是如果使用了setuid/setgid，由于安全因素，此路径将被忽略
	* /etc/ld/so/cache. 如果链接时指定了‘-z nodeflib’，此路径将被忽略.
	* /lib. 如果链接时指定了‘-z nodeflib’，此路径将被忽略
	* /usr/lib. 如果链接时指定了‘-z nodeflib’，此路径将被忽略


##版本控制

基于上面的例子，看看linux的这种约定，如何达到版本控制的目的。上面我们留下了一个问题：通过soname去寻找真正的real name库，这有什么好处？
假设，我们现在对上面的test共享库进行升级，有2中情况：
* 修改了原来的接口
* 增加了新的接口

(1) 修改了原来的接口

我们修改test.c的代码，并修改原来的接口：

* test.c

```c
#include <stdio.h>
#include "test.h"

void print_hello()
{
    printf("this is a test for shared lib");
	printf("this is a new building version");
}
```

然后我们重新编译生成共享库，并且定义为新的building版本
```
gcc -fPIC -o test.o -c test.c
gcc -shared -Wl,-soname,libtest.so.0 -o libtest.so.0.0.1 test.o
```

注意，按照约定，由于新的版本只是修改了接口，可以兼容之前的版本，所以soname并不需要改变。生成新的real name库以后，我们只需要执行ldconfig，即可自动
更新soname到新real name库的软链接。

* 之前的链接

```shell
bow@bow-Aspire-4752:~/all/program/test/lib_version_test$ ll
total 44
drwxrwxr-x  2 bow bow 4096  6月 28 12:53 ./
drwxrwxr-x 10 bow bow 4096  6月 27 21:21 ../
lrwxrwxrwx  1 bow bow   12  6月 28 12:50 libtest.so -> libtest.so.0*
lrwxrwxrwx  1 bow bow   16  6月 28 12:50 libtest.so.0 -> libtest.so.0.0.0*
-rwxrwxr-x  1 bow bow 6894  6月 28 12:42 libtest.so.0.0.0*
-rwxrwxr-x  1 bow bow 7287  6月 28 12:53 main*
-rwxrwxr-x  1 bow bow   81  6月 27 23:16 main.c*
-rw-rw-r--  1 bow bow  936  6月 28 12:46 main.o
-rw-rw-r--  1 bow bow  104  6月 27 21:20 test.c
-rw-rw-r--  1 bow bow   61  6月 27 21:19 test.h
-rw-rw-r--  1 bow bow 1344  6月 28 12:39 test.o
```

* ldconfig更新之后

```shell
bow@bow-Aspire-4752:~/all/program/test/lib_version_test$ ll
total 52
drwxrwxr-x  2 bow  bow  4096  6月 28 15:07 ./
drwxrwxr-x 10 bow  bow  4096  6月 27 21:21 ../
lrwxrwxrwx  1 bow  bow    12  6月 28 12:50 libtest.so -> libtest.so.0*
lrwxrwxrwx  1 root root   16  6月 28 15:07 libtest.so.0 -> libtest.so.0.0.1*
-rwxrwxr-x  1 bow  bow  6894  6月 28 12:42 libtest.so.0.0.0*
-rwxrwxr-x  1 bow  bow  6894  6月 28 15:06 libtest.so.0.0.1*
-rwxrwxr-x  1 bow  bow  7287  6月 28 12:53 main*
-rwxrwxr-x  1 bow  bow    81  6月 27 23:16 main.c*
-rw-rw-r--  1 bow  bow   936  6月 28 12:46 main.o
-rw-rw-r--  1 bow  bow   104  6月 27 21:20 test.c
-rw-rw-r--  1 bow  bow    61  6月 27 21:19 test.h
-rw-rw-r--  1 bow  bow  1344  6月 28 12:39 test.o
```

看到没有，soname自动更新到了新版本的共享库。所以之前的应用程序main会使用新的共享库。

```shell
bow@bow-Aspire-4752:~/all/program/test/lib_version_test$ ./main
this is a test for shared libthis is a new building version
```


(2) 增加了新的接口
	
我们修改test.c的代码，并增加接口：

* test.h

```c
#ifndef _TEST_H_
#define _TEST_H_
void print_hello();
void print_hello2();
#endif
```

* test.c

```c
#include <stdio.h>
#include "test.h"

void print_hello()
{
	printf("this is a test for shared lib");
	printf("this is a new release version");
}

void print_hello2()
{
	printf("this is a new release version");
}
```
然后编译，根据约定，real name的主版本号需要更新。

```shell
gcc -fPIC -o test.o -c test.c
gcc -shared -Wl,-soname,libtest.so.1 -o libtest.so.1.0.0 test.o
```

这时，就会生成新的共享库libtest.so.1.0.0，然后ldconfig更新软链接和cache。

```shell
total 60
drwxrwxr-x  2 bow  bow  4096  6月 28 15:15 ./
drwxrwxr-x 10 bow  bow  4096  6月 27 21:21 ../
lrwxrwxrwx  1 bow  bow    12  6月 28 12:50 libtest.so -> libtest.so.0*
lrwxrwxrwx  1 root root   16  6月 28 15:07 libtest.so.0 -> libtest.so.0.0.1*
-rwxrwxr-x  1 bow  bow  6894  6月 28 12:42 libtest.so.0.0.0*
-rwxrwxr-x  1 bow  bow  6894  6月 28 15:06 libtest.so.0.0.1*
lrwxrwxrwx  1 root root   16  6月 28 15:15 libtest.so.1 -> libtest.so.1.0.0*
-rwxrwxr-x  1 bow  bow  6894  6月 28 15:15 libtest.so.1.0.0*
-rwxrwxr-x  1 bow  bow  7287  6月 28 12:53 main*
-rwxrwxr-x  1 bow  bow    81  6月 27 23:16 main.c*
-rw-rw-r--  1 bow  bow   936  6月 28 12:46 main.o
-rw-rw-r--  1 bow  bow   104  6月 27 21:20 test.c
-rw-rw-r--  1 bow  bow    61  6月 27 21:19 test.h
-rw-rw-r--  1 bow  bow  1344  6月 28 12:39 test.o
```

```shell
bow@bow-Aspire-4752:~/all/program/test/lib_version_test$ ./main
this is a test for shared libthis is a new building version
```

这个时候，虽然我们更新了共享库，但是main还是会加载旧的共享库。这就保证了，即使共享库更新，以前的程序也能正常工作。
因为main里面的soname没变，而且soname对应的real name没变。
如果你想更新应用程序main，使其使用新的库，那么需要重新编译，让libtest.so的软链接指向libtest.so.1

```shell
bow@bow-Aspire-4752:~/all/program/test/lib_version_test$ ln -sf libtest.so.1 libtest.so
bow@bow-Aspire-4752:~/all/program/test/lib_version_test$ ll
total 60
drwxrwxr-x  2 bow  bow  4096  6月 28 15:32 ./
drwxrwxr-x 10 bow  bow  4096  6月 27 21:21 ../
lrwxrwxrwx  1 bow  bow    12  6月 28 15:32 libtest.so -> libtest.so.1*
lrwxrwxrwx  1 root root   16  6月 28 15:07 libtest.so.0 -> libtest.so.0.0.1*
-rwxrwxr-x  1 bow  bow  6894  6月 28 12:42 libtest.so.0.0.0*
-rwxrwxr-x  1 bow  bow  6894  6月 28 15:24 libtest.so.0.0.1*
lrwxrwxrwx  1 root root   16  6月 28 15:15 libtest.so.1 -> libtest.so.1.0.0*
-rwxrwxr-x  1 bow  bow  6923  6月 28 15:25 libtest.so.1.0.0*
-rwxrwxr-x  1 bow  bow  7287  6月 28 12:53 main*
-rwxrwxr-x  1 bow  bow    81  6月 27 23:16 main.c*
-rw-rw-r--  1 bow  bow   936  6月 28 12:46 main.o
-rw-rw-r--  1 bow  bow   216  6月 28 15:25 test.c
-rw-rw-r--  1 bow  bow    82  6月 28 15:25 test.h
-rw-rw-r--  1 bow  bow  1516  6月 28 15:25 test.o
bow@bow-Aspire-4752:~/all/program/test/lib_version_test$ gcc -L. -o main main.o -ltest
bow@bow-Aspire-4752:~/all/program/test/lib_version_test$ ./main
this is a test for shared libthis is a new release version
```

ok。通过以上操作，linux下面的共享库的版本约定和控制，就很清楚了。当碰到一些譬如无法找到共享库或接口，或者引用的库
发生变化的时候，看看应用程序到底时如何查找共享库的，已经在使用的是哪个版本的共享库，根据以上原则去分析，一定可以解决问题。
一般通过这几个命令去分析：

* nm
    查看共享库暴露的接口
* ldconfig 可以自动生成soname的链接接文件。并更新共享库的搜索cache，加速查找。
	example: ldconfig -p | grep libtest
* readelf
	可以查看动态库的信息，本身的soname。
	example: readelf -d libtest.so.0.0.0 | grep soname
* ldd
	可以查看应用程序或共享库依赖的库
	example: ldd main
* objdump 与readelf 类似

##参考资料
[http://tldp.org/HOWTO/Program-Library-HOWTO/shared-libraries.html](http://tldp.org/HOWTO/Program-Library-HOWTO/shared-libraries.html)

[http://www.cprogramming.com/tutorial/shared-libraries-linux-gcc.html](http://www.cprogramming.com/tutorial/shared-libraries-linux-gcc.html)

[http://www.linuxidc.com/Linux/2012-04/59071p2.htm](http://www.linuxidc.com/Linux/2012-04/59071p2.htm)
