LIBTOOL
===
使用 GNU Libtool 可以容易的在不同的系统中建立动态链接库。它通过一个称为 Libtool 库的抽象，隐藏了不同系统之间的差异，给开发人员提供了一致的的接口。对于大部分情况，开发人员甚至不用去查看相应的系统手册，只需要掌握 GNU Libtool 的用法就可以了。并且，使用 Libtool 的 Makefile 也只需要编写一次就可以在多个系统上使用。  
Libtool 库可以是一个静态链接库，可以是一个动态链接库，也可以同时包含两者。在这篇文档中，我们围绕 Libtool 库的建立和使用，只是在适当的说明 Libtool 库和系统动态或者静态链接库之间的映射关系。  
## Libtool 是一个工具
虽然 Libtool 隐藏了在不同平台创建链接库的复杂性，但其最终还是需要底层系统对链接库的支持，它不能超越系统的限制，例如，Libtool 并不能在不支持动态链接库的系统中创建出动态链接库。  
## Libtool 基本用法
*	创建 Libtool 对象文件 ;
*	创建 Libtool 库；
*	安装 Libtool 库 ;
*	使用 Libtool 库 ;
*	卸载 Libtool 库 ;

准备一个源文件 compress.c  
这个文件实现了一个函数 compress_file()，它接收一个文件名作为参数，然后对文件进行压缩，生成一个 .z结尾的压缩文件。在这个文件中使用了 compress()函数，这个函数是有由 libz 提供的。  
从源文件建立 Libtool 库需要经过两个步骤，先建立 Libtool 对象文件，再建立 Libtool 库。  
### 一、建立 Libtool 对象文件
如果使用传统的方式，建立对象文件通常使用下面的命令 :  
```
$ gcc -c compress.c
```
使用 Libtool 则使用下面的命令 :  
```
$ libtool --mode=compile gcc -c compress.c
```
可以看到，使用 Libtool 只需要将“传统”的命令 (gcc -c foo.c) 作为参数传递给 Libtool 即可。  
在上面的命令中，libtool 使用 compile模式 (--mode=compile 选项 )，这是建立对象文件的模式，Libtool 还有其它的模式，后面将介绍。  
上面的命令输出如下 :  
```
mkdir .libs 
gcc -c compress.c  -fPIC -DPIC -o .libs/compress.o  
gcc -c compress.c -o compress.o >/dev/null 2>&1  
```
它建立了两个文件，一个是 .libs/compress.o，在建立这个文件时，Libtool 自动插入了 -fPIC和 -DPIC选项，告诉编译器生成位置独立的代码，之后将用这个文件来建立动态链接库。生成第二个文件 compress.o没有添加额外的选项，它准备用来建立静态链接库。  
除了上面的两个文件之外，Libtool 还建立了一个文件 compress.lo，这个文件就是 Libtool 对象文件，实际上也就是一个文本文件，里面记录了建立动态链接库和静态链接库分别所需要的真实文件名称，后面 Libtool 将使用这个文件而不是直接的使用 .libs/compress.o 和 compress.o。  
### 二、建立 Libtool 库
用下面的命令建立 Libtool 库 :
```
$ libtool --mode=link gcc -o libcompress.la compress.lo -rpath /tmp -lz
```
注意这里使用 compress.lo 作为输入文件，并且告诉 Libtool 生成的目标文件为 libcompress.la，.la 是 Libtool 的库文件后缀。  
-rpath选项告诉 Libtool 这个库将被安装到什么地方，如果省略了 -rpath选项，那么不会生成动态链接库。  
因为我们的库中使用了 libz 提供的 compress 函数，所以也提供了 -lz 选项，Libtool 会记住这个依赖关系，后续在使用我们的库时自动的将依赖的库链接进来。  
上面的命令输出如下 :  
```
gcc -shared  .libs/compress.o  -lz  -Wl,-soname -Wl,libcompress.so.0 -o .libs/libcompress.so.0.0.0
(cd .libs && rm -f libcompress.so.0 && ln -s libcompress.so.0.0.0 libcompress.so.0) 
(cd .libs && rm -f libcompress.so && ln -s libcompress.so.0.0.0 libcompress.so) 
ar cru .libs/libcompress.a  compress.o 
ranlib .libs/libcompress.a 
creating libcompress.la 
(cd .libs && rm -f libcompress.la && ln -s ../libcompress.la libcompress.la)
```
可以看到，Libtool 自动的插入了建立动态链接库需要的编译选项 -shared。并且，它也建立了静态链接库 .libs/libcompress.a，后面我们将会介绍如何控制 Libtool 只建立需要的库。
你可能会奇怪为什么建立的动态链接库有 .0 和 .0.0.0 这样的后缀，这里先不用理会它，后面在介绍 Libtool 库版本信息时将会解释这点。
值得注意的是，Libtool 希望后续使用 libcompress.la 文件而不是直接使用 libcompress.a 和 libcompress.so 文件，如果你这样做，虽然可以，但会破坏 Libtool 库的可移植性。  
### 三、安装 Libtool 库
如果打算发布建立好的 Libtool 库，可以使用下面的命令安装它 :  
```
$ libtool --mode=install install -c libcompress.la /tmp
```
我们需要告诉 Libtool 使用的安装命令，Libtool 支持 install 和 cp，这里使用的是 install。  
虽然前面我们在建立库时，通过 -rpath 选项指定了库准备安装的路径 (/tmp)，但是这里我们还得要提供安装路径。请确保它们一致。  
这个命令的输出如下 :   
```
install .libs/libcompress.so.0.0.0 /tmp/libcompress.so.0.0.0 
(cd /tmp && { ln -s -f libcompress.so.0.0.0 libcompress.so.0 || 
{ rm -f libcompress.so.0 && 
ln -s libcompress.so.0.0.0 libcompress.so.0; }; }) 
(cd /tmp && { ln -s -f libcompress.so.0.0.0 libcompress.so || 
{ rm -f libcompress.so && ln -s libcompress.so.0.0.0 libcompress.so; }; }) 
install .libs/libcompress.lai /tmp/libcompress.la 
install .libs/libcompress.a /tmp/libcompress.a 
chmod 644 /tmp/libcompress.a 
ranlib /tmp/libcompress.a 
...
```
可以看到它安装了真实的动态链接库和静态链接库，同时也安装了 Libtool 库文件 libcompress.la，这个文件可以被后续的 Libtool 命令使用。  
在安装完成之后，可能还需要做一些配置才能正确使用，Libtool 的 finish 模式可以在这方面给我们一些提示 :  
```
$ libtool -n --mode=finish /tmp
```
这个命令的输出有点长，所以不在这里列出，如果不能正常的使用安装好的库，请运行这个命令。  
### 四、使用 Libtool 库
要在应用程序中使用前面创建的 Libtool 库很简单，准备一个源文件 main.c，它将使用 libcompress.la 库中定义的函数  
我们还是要先为 main.c 建立 Libtool 对象文件，这和前面的方法一样 :
```
$ libtool --mode=compile gcc -c main.c
```
#### 1）使用安装的库
然后使用下面的命令链接执行文件 :  
```
$ libtool --mode=link gcc -o main main.lo /tmp/libcompress.la
```
我们也可以直接使用 libcompress.a 或者 libcompress.so，但是使用 Libtool 更加简单，因为它会将帮助你解决依赖关系，例如我们的 libcompress 依赖 libz。
上面命令的输出如下 :
```
gcc -o main .libs/main.o  /tmp/libcompress.so -lz 
-Wl,--rpath -Wl,/tmp -Wl,--rpath -Wl,/tmp
```
这里，Libtool 自动选择链接动态链接库，并且加上了运行时需要的 --rpath 选项，以及依赖的库 -lz。  
如果要使用静态链接库，只需要加上 -static-libtool-libs选项即可，如下 :  
```
 $ libtool --mode=link  gcc  -o main main.lo /tmp/libcompress.la -static-libtool-libs
```
这个命令的输出如下 :
```
gcc -o main .libs/main.o  /tmp/libcompress.a -lz
```
#### 2）使用未安装的库
也可以使用还没有安装的库，这和使用安装好的库几乎相同，只是指定的输入文件位置不一样，假如我们在同一个目录中开发 compress.c 和 main.c，那么使用下面的命令 :
```
$ libtool --mode=link gcc -o main main.lo ./libcompress.la
```
和使用安装的库不一样，这个时候建立的 main 程序只是一个封装脚本，如果你直接执行它不会有什么问题，但是如果你想调试它，例如 :
```
$ gdb main
```
gdb 会报怨 main 不是可执行格式，不能接受。这个时候我们需要使用 Libtool 的执行模式，使用下面的命令调试程序 :
```
$ libtool --mode=execute gdb main
```

