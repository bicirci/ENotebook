  
  
#  Valgrind交叉编译
  
  
##  1. 下载
  
  
下载地址：[https://www.valgrind.org/downloads/](https://www.valgrind.org/downloads/ )
  
##  2. 交叉编译
  
  
解压后进入valgrind-3.17.0文件夹。  
  
配置好交叉编译工具链环境后执行 `./configure --host=aarch64-linux-gnu --prefix=$PWD/_install`  
  
`make -j8  && make install`  
  
生成的可执行程序位于prefix定义的文件夹下/bin文件夹内。  
需要使用的程序为valgrind，运行时需要的库文件在/libexec/valgrind文件夹下。  
  
将以上文件拷贝到调试环境中  
  
##  3. 调试
  
  
首先保证需要调试的程序可以运行，配置好所需的环境参数  
  
将拷贝到调试环境的库文件夹配置到环境参数里。  
`export VALGRIND_LIB=/path/to/libexec/valgrind`  
  
运行valgrind进行调试  
`valgrind --log-file=/tmp/valgrind.log --tool=memcheck --leak-check=full --show-leak-kinds=all --trace-children=yes ./your_app arg1 arg2`  
  
* --log-file 生成报告路径。如果没有指定，输出到stderr。  
* --tool=memcheck 指定Valgrind使用的工具。Valgrind是一个工具集，包括Memcheck、Cachegrind、Callgrind等多个工具。  
* --leak-check 指定报告内存泄漏的详细程度   
* --show-leak-kinds 指定检查哪几种内存泄漏类型  
* --trace-children=yes 是否检查forked的进程  
  
然后等待valgrind模拟开启被调试程序后，进行一段时间的交互后，即可通过ctrl+c关闭进程  
  
##  4. 注意事项
  
  
* 程序和库在编译时，使用-g，并最好使用O1进行优化。 O0速度慢， O2以上会造成uninitialised value错误的误报。  
  