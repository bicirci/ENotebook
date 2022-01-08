  
  
#  GTest 编译说明
  
  
- [GTest 编译说明](#gtest-编译说明 )
  - [1.概述](#1概述 )
  - [2.相关链接](#2相关链接 )
  - [3.编译流程](#3编译流程 )
    - [3.1 下载googletest源码](#31-下载googletest源码 )
    - [3.2 配置Cmake编译选项](#32-配置cmake编译选项 )
    - [3.2 GTest静态库编译](#32-gtest静态库编译 )
  
##  1.概述
  
  
本文旨在说明GTest在Linux系统下基于CMake的编译流程，生成可供使用的动态/静态库文件。以及如何进行交叉编译。  
  
* 注意：GoogleTest仅支持C++11以上版本  
  
##  2.相关链接
  
  
Google Test官方文档: <https://google.github.io/googletest/>  
Google Test Git仓库: <https://github.com/google/googletest>  
  
##  3.编译流程
  
  
###  3.1 下载googletest源码
  
  
`git clone https://github.com/google/googletest.git`  
下载的源码包含 googlemock 和 googletest 两个模块，本文只涉及googletest的编译。  
  
![GTest编译说明-2021-07-07-gtest1](https://cdn.jsdelivr.net/gh/bicirci/PicBed@master/MarkDown/GTest编译说明-2021-07-07-gtest1.png )  
  
###  3.2 配置Cmake编译选项
  
  
进入主目录下的googletest文件夹，CMakeLists.txt用来编译，README.md介绍了编译方法和可选的配置选项，比如是否编译sample等。googletest默认编译出的是静态库.a版本，如果希望生成动态库，需要修改BUILD_SHARED_LIBS这个选项。  
可以手动修改CMakeLists.txt文件，也可以通过cmake-gui编辑。  
  
![GTest编译说明-2021-07-07-gtest3](https://cdn.jsdelivr.net/gh/bicirci/PicBed@master/MarkDown/GTest编译说明-2021-07-07-gtest3.png )
  
**注意：由于CMAKE版本的区别，有些编译选项可能不支持，Cmake配置阶段如果提示set_target_properties的参数错误，可以将132行和134行的两句注释掉**  
  
```C++
132 # set_target_properties(gtest PROPERTIES VERSION ${GOOGLETEST_VERSION})  
...  
134 # set_target_properties(gtest_main PROPERTIES VERSION ${GOOGLETEST_VERSION})  
```
  
![GTest编译说明-2021-07-07-gtest4](https://cdn.jsdelivr.net/gh/bicirci/PicBed@master/MarkDown/GTest编译说明-2021-07-07-gtest4.png )
  
###  3.2 GTest静态库编译
  
  
* 如果只是想编译x86/x64的ubuntu版本，直接新建build文件夹，`cd build`进入该文件夹，执行`cmake ..`和`make`即可。生成的静态库文件位于build/lib文件夹内，想在其他项目使用gtest只需要将include文件夹和build/lib文件夹复制过去，配置好编译链接选项即可。  
* 如果希望进行交叉编译，gtest的编译脚本已经预留了自定义的文件接口。  
  
![GTest编译说明-2021-07-07-gtest2](https://cdn.jsdelivr.net/gh/bicirci/PicBed@master/MarkDown/GTest编译说明-2021-07-07-gtest2.png )
  
只需要在cmake文件夹下新建hermetic_build.cmake文件，将以下内容根据实际情况修改后粘贴到cmake/hermetci_build.cmake即可。  
  
```cmake
set(CMAKE_SYSTEM_NAME Linux)  
set(CMAKE_SYSTEM_PROCESSOR arm)  
# 根据实际情况，指定交叉编译工具链的bin文件夹路径  
set(toolchains /home/eric/project/toolchain/robot/gcc-linaro-6.4.1-2018.05-x86_64_aarch64-linux-gnu/bin)  
# 分别指定C和C++编译器  
set(CMAKE_C_COMPILER ${toolchains}/aarch64-linux-gnu-gcc)  
set(CMAKE_CXX_COMPILER ${toolchains}/aarch64-linux-gnu-g++)  
  
```
  
然后同样新建build文件夹，`cd build`进入该文件夹，执行`cmake ..`和`make`即可。生成的静态库文件位于build/lib文件夹内。  
  
![GTest编译说明-2021-07-07-gtest5](https://cdn.jsdelivr.net/gh/bicirci/PicBed@master/MarkDown/GTest编译说明-2021-07-07-gtest5.png )
  