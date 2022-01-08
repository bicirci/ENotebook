  
  
#  GDB调试环境搭建（交叉编译）
  
  
- [GDB调试环境搭建（交叉编译）](#gdb调试环境搭建交叉编译 )
  - [1.概述](#1概述 )
  - [2.GDB简介](#2gdb简介 )
  - [GDB的交叉编译](#gdb的交叉编译 )
  - [GDB调试环境部署](#gdb调试环境部署 )
    - [1. 现场挂载调试](#1-现场挂载调试 )
    - [2. 远程连接调试](#2-远程连接调试 )
    - [3. 离线core调试](#3-离线core调试 )
  
##  1.概述
  
  
* 本文旨在说明嵌入式项目通用gdb调试环境的搭建流程以及基本的调试步骤。  
  
* 个人是在扫地机配网压测调试流程中，测试脚本重复的执行配网操作，在连续执行到10-20次的区间范围内，程序发生崩溃，通过日志查看无法定位问题，因此需要通过GDB调试定位崩溃发生时的代码。  
  
##  2.GDB简介
  
  
* GDB是GNU Debugger的缩写。与make一样，同样来自于GNU项目。嵌入式开发测试人员通常是在，通过设备的日志log无法定位bug时，才会用到gdb进行详细的内存、运行态信息的调试。  
  
> 本文后续涉及的调试步骤都是针对程序崩溃问题，如果是希望利用GDB进行步进调试，详细的查看程序的状态变化，使用方法可自行百度，但编译和环境部署的方法是相同的。  
  
* coredump文件，简称core文件。当程序进程由异常信号触发结束时，生成该进程的内存转储文件(相当于把崩溃时的完整内存信息拷贝下来，由于虚拟页表的存在，该文件有可能大于实际内存占用)，该文件包含了进程当前的运行堆栈信息。  
  
> 通过GDB可以读取coredump中的堆栈信息，从而定位崩溃发生的实际位置。  
需要注意的是，coredump在linux系统不一定是默认生成的，需要通过ulimit配置系统参数, 控制 shell 启动进程所占用的资源。  
ulimit -a //查看当前系统设置参数  
ulimit -c unlimited //设置允许生成的core文件的最大尺寸（这里设置为无限制）  
此外，还可以通过设置 /proc/sys/kernel/core_pattern 内部参数，coredump文件的自定义位置保存，具体可以百度。  
  
##  GDB的交叉编译
  
  
这里我使用的方法仅是个人测试通过可用的，可能和标准的编译方式有细节上的区别。  
  
1. 编译准备
  
* libexpat(GDB的依赖库)，版本：2.2.10 [下载链接](https://github.com/libexpat/libexpat/releases/ )  
* GDB源码，版本：10.1 [下载链接](https://ftp.gnu.org/gnu/gdb/ )  
  
2.libexpat的静态库交叉编译
  
* 解压后进入expat-2.2.10文件夹  
* `./configure --host=aarch64-linux-gnu --prefix=<img src="https://latex.codecogs.com/gif.latex?PWD&#x2F;_install`%20--host配置为对应工具链版本,和需要编译的GDB配置保持一致%20--prefix为编译结果的安装位置%20%20*%20`make%20&amp;&amp;%20make%20install`%20%20*%20编译后生成的libexpat.a位于_install&#x2F;lib%20%203.GDB的交叉编译*%20解压后进入gdb-10.1文件夹%20%20*%20执行%20`.&#x2F;configure%20--host=aarch64-linux-gnu%20--target=aarch64-linux-gnu%20--prefix="/>PWD/_output --disable-werror --with-expat --includedir=<img src="https://latex.codecogs.com/gif.latex?HOME&#x2F;public&#x2F;expat-2.2.10&#x2F;_install&#x2F;include%20--libdir="/>$HOME/public/expat-2.2.10/_install/lib`  
**注意** ：  
--host 对应编译产物的宿主机(GDB程序实际在哪运行)架构  
--target 对应调试目标的架构（core文件在哪生成的）后文会详细说明。  
--with-expat 说明是附带expat库一起编译  
--includedir和--libdir分别指向刚刚编译出的libexpat输出文件夹的include和lib文件夹  
--prefix同上  
  
* `make && make install`  
  
* 编译后生成的可执行文件位于_output/bin内， 包含gdb 和 gdbserver两个可执行程序  
  
![GDB调试环境搭建-2021-09-16-2021-09-16_14-48](https://cdn.jsdelivr.net/gh/bicirci/PicBed@master/MarkDown/GDB调试环境搭建-2021-09-16-2021-09-16_14-48.png )
  
##  GDB调试环境部署
  
  
* GDB调试可分为现场挂载调试，远程连接调试和离线core调试。现场挂载调试是在出错进程的运行的linux环境中，将GDB挂载到该进程上，发生错误时直接进行调试。  
此方法最简单，但需要出错程序的运行环境有较大可用内存（几十mb以上）。  
  
* 远程连接调试是在出错进程运行的linux环境中，运行gdbserver挂载出错进程。而另一台电脑上运行gdb，和已挂载的gdbserver建立连接，通过网络进行远程调试。  
此方法适用于网络环境稳定的情况，不需来回拷贝coredump文件。  
  
* 离线core调试则是将进程出错后生成的coredump文件拷贝到另一台电脑上，运行gdb进行调试。  
此方法在嵌入式调试中最为常用。  
  
下面介绍一下这三种方式的操作步骤：  
  
调试环境：  
调试机：ubuntu 18.4 x86_64  
出错程序运行环境：Linux 4.4.143 aarch64 GNU/Linux  
交叉编译工具链： aarch64-linux-gnu  
  
###  1. 现场挂载调试  
  
  
* GDB编译时--target和--host都要指定为 aarch64-linux-gnu。将编译出的gdb文件拷贝到出错程序的运行环境内。  
  
* 查看出错程序的进程号 `ps -ef`  
  
* 执行gdb (假设出错进程id=111) `gdb -pid=111`  
  
![GDB调试环境搭建-2021-09-16-2021-09-16_17-32](https://cdn.jsdelivr.net/gh/bicirci/PicBed@master/MarkDown/GDB调试环境搭建-2021-09-16-2021-09-16_17-32.png )
  
* 待出错程序崩溃后通过`bt`查看堆栈信息。其他指令可自行百度。  
  
###  2. 远程连接调试  
  
  
需要出错程序运行环境本身可以联网。  
  
* GDB编译时需要编译两份。一份为调试机上运行的gdb， 一份为出错程序运行环境内执行的gdbserver。  
调试机上运行的gdb，在编译libexpat和gdb时--host可以删去， 编译时会自动选择x80_64_linux，但是GDB编译时的--target需要指定为aarch64-linux-gnu。  
出错程序环境内执行的gdb server， 和1的编译选项相同，--target和--host都要选择 aarch64-linux-gnu。  
  
* 将gdbserver拷贝至出错程序运行环境  
  
* 查看出错程序的进程号 `ps -ef`  
  
* 查看出错程序所在环境的网络ip  
  
* 执行gdbserver并随意指定一个调试端口 (假设调试端口prot=6666, 出错进程id=111, ip=192.168.1.2) `gdbserver 192.168.1.2:6666 --attach 111`  
  
![GDB调试环境搭建-2021-09-16-2021-09-16_18-40](https://cdn.jsdelivr.net/gh/bicirci/PicBed@master/MarkDown/GDB调试环境搭建-2021-09-16-2021-09-16_18-40.png )
  
* 将出错程序运行环境内，该程序使用到的库文件，按照原文件结构，拷贝到调试机上。 假设使用的库文件都位于/sys/lib， 则将整个lib文件夹拷贝到调试机上的 ....../test/**sys/lib**  
  
* 执行调试机上的gdb程序 `gdb`(可能程序名为aarch64-linux-gnu-gdb，注意替换)  
  
* 设置运行库的搜索的根目录路径(跟上文对应) `set sysroot ....../test`  
  
![GDB调试环境搭建-2021-09-16-gdb](https://cdn.jsdelivr.net/gh/bicirci/PicBed@master/MarkDown/GDB调试环境搭建-2021-09-16-gdb.png )
  
* 保证调试机和出错程序的环境位于同一局域网内，进行连接`target remote 192.168.1.2:6666`  
可以看到gdb成功的载入了所需的动态库  
  
![GDB调试环境搭建-2021-09-16-gdb2](https://cdn.jsdelivr.net/gh/bicirci/PicBed@master/MarkDown/GDB调试环境搭建-2021-09-16-gdb2.png )
  
* gdb连接时会中断正在调试的程序，需要执行`continue`继续  
  
* 待出错程序崩溃后通过`bt`查看堆栈信息。其他指令可自行百度。  
  
###  3. 离线core调试
  
  
* GDB编译时--target指定为 aarch64-linux-gnu, --host可删去  
  
* 待出错程序崩溃后，将出错程序以及生成的coredump文件拷贝到调试机上  
  
* 在调试机上运行gdb并指定出错程序和coredump文件 gdb ....../pathto/bugexe ....../pathto/bugexe.core (可能程序名为aarch64-linux-gnu-gdb，注意替换)  
  
![GDB调试环境搭建-2021-09-16-dbg3](https://cdn.jsdelivr.net/gh/bicirci/PicBed@master/MarkDown/GDB调试环境搭建-2021-09-16-dbg3.png )
  
* 将出错程序运行环境内，该程序使用到的库文件，按照原文件结构，拷贝到调试机上。 假设使用的库文件都位于/sys/lib， 则将整个lib文件夹拷贝到调试机上的 ....../test/**sys/lib**  
  
* 设置运行库的搜索的根目录路径(跟上文对应) `set sysroot ....../test`, 可以看到此时载入了所需的动态库
  
![GDB调试环境搭建-2021-09-16-dbg4](https://cdn.jsdelivr.net/gh/bicirci/PicBed@master/MarkDown/GDB调试环境搭建-2021-09-16-dbg4.png )
  
* 通过`bt`查看堆栈信息。其他指令可自行百度。  
  