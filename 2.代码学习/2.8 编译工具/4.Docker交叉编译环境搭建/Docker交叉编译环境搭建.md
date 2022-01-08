  
#  Docker交叉编译环境搭建
  
  
##  1.Ubuntu下Docker Engine安装
  
  
参考官方说明进行安装[https://docs.docker.com/engine/install/ubuntu/](https://docs.docker.com/engine/install/ubuntu/ )  
  
##  2. Docker图形管理工具Portainer安装
  
  
由于linux版本的docker Engine不像mac上的docker desktop, 并没有自带图形管理界面, 为了便于操作, 使用第三方的图形管理工具Portainer.  
参考官方说明进行安装 Portainer CE 2.11版本[https://docs.portainer.io/v/ce-2.11/start/install/server/docker/linux](https://docs.portainer.io/v/ce-2.11/start/install/server/docker/linux )  
  
安装好后访问https://localhost:9443即可  
  
##  3. 建立基础的ubuntu镜像
  
  
先配置各项目都需要的基础编译环境ubuntu20.0 + clang 13 stable.  
  
新建Dockerfile文件夹, 文件夹结构如下:  
  
``` text
|--Dockerfile
|--sources.list
```
  
其中包含的文件内容如下  
  
* Dockerfile:
  
``` Dockerfile
FROM ubuntu:20.04
RUN apt-get update \
    && apt-get install -y wget apt-transport-https ca-certificates gnupg
COPY sources.list /etc/apt/sources.list
RUN wget -O - https://apt.llvm.org/llvm-snapshot.gpg.key| apt-key add - \
    && apt-get update
#因tzdata安装途中有交互式的输入,需要用echo模拟
RUN sh -c '/bin/echo -e "6\n70\n" | apt-get install -y tzdata' \
    && apt-get install -y build-essential cmake gdb vim git \
    && apt-get install -y openssh-server rsync \
    && apt-get install -y clang-13 lldb-13 lld-13 \
    && apt-get install -y libllvm-13-ocaml-dev libllvm13 llvm-13 llvm-13-dev llvm-13-doc llvm-13-examples llvm-13-runtime
  
```
  
* sources.list
  
``` text
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal main restricted
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-updates main restricted
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal universe
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-updates universe
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal multiverse
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-updates multiverse
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-backports main restricted universe multiverse
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-security main restricted
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-security universe
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-security multiverse
  
deb http://apt.llvm.org/focal/ llvm-toolchain-focal main
deb-src http://apt.llvm.org/focal/ llvm-toolchain-focal main
# 12
deb http://apt.llvm.org/focal/ llvm-toolchain-focal-12 main
deb-src http://apt.llvm.org/focal/ llvm-toolchain-focal-12 main
# 13
deb http://apt.llvm.org/focal/ llvm-toolchain-focal-13 main
deb-src http://apt.llvm.org/focal/ llvm-toolchain-focal-13 main
```
  
然后运行`docker build -t ubuntu_base .` 等待镜像构建完成  
  
完成后可以通过`docker container ls -a`查看新建的ubuntu_base镜像  
  
##  3. 建立编译环境镜像
  
  
新建Dockerfile文件夹, 从官网下载工具链[下载地址](https://releases.linaro.org/components/toolchain/binaries/6.4-2018.05/aarch64-linux-gnu/gcc-linaro-6.4.1-2018.05-x86_64_aarch64-linux-gnu.tar.xz ),放入该文件夹, 文件夹结构如下:  
  
``` text
|--Dockerfile
|--gcc-linaro-6.4.1-2018.05-x86_64_aarch64-linux-gnu.tar.xz
```
  
其中包含的文件内容如下
  
* Dockerfile:  
  
``` Dockerfile
FROM ubuntu_base:latest
RUN useradd -r -u 1000 developer \
    && mkdir /home/developer \
    && chmod 777 /home/developer
#避免docker exec时以默认的root进行操作
USER developer
RUN mkdir -p $HOME/toolchain \
    && mkdir -p $HOME/cleanRobot/thirdparty \
    && mkdir -p $HOME/cleanRobot/robotbase/
# 切换工作目录
WORKDIR $HOME/toolchain
COPY gcc-linaro-6.4.1-2018.05-x86_64_aarch64-linux-gnu.tar.xz $HOME/toolchain
RUN tar -xf gcc-linaro-6.4.1-2018.05-x86_64_aarch64-linux-gnu.tar.xz -C $HOME/toolchain/ \
    && mv -f $HOME/toolchain/gcc-linaro-6.4.1-2018.05-x86_64_aarch64-linux-gnu $HOME/toolchain/aarch64_linux \
    && echo "export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib/" >> $HOME/.bashrc \
    && echo "export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib64/" >> $HOME/.bashrc \
    && echo "export ARM64_GCC_HOME=$HOME/toolchain/aarch64_linux" >> $HOME/.bashrc
WORKDIR $HOME/cleanRobot/robotbase/
#初始命令
CMD ["bash"]
```
  
将交叉编译工具链下载完成后, 进行镜像构建`docker build -t qrob_env .`  
  
##  4.创建所需的编译环境的容器
  
  
1. 如果是直接下载的预先构建好的镜像, 需要先将镜像导入本地  
  
假设下载的镜像为 qrob_env.tar, 则执行`docker load -i ./qrob_env.tar`(-i表示读入的是压缩包)  
  
2. 使用以下脚本创建本地运行的容器
  
S8项目编译环境: s8penv_deploy.sh  
  
``` bash
#!/bin/bash
  
THIRD_DIR="替换为s8p分支的thirdparty目录"
ROBBASE_DIR="替换为s8p分支的robotbase目录"
USER_ID=$(id -u)
USER_GP=$(id -g)
sudo docker run -it\
    --name s8penv \
    --mount type=bind,source=$THIRD_DIR,target=/home/developer/cleanRobot/thirdparty \
    --mount type=bind,source=$ROBBASE_DIR,target=/home/developer/cleanRobot/robotbase \
    -e LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib:/usr/local/lib64 \
    -e ARM64_GCC_HOME=/home/developer/toolchain/aarch64_linux \
    -w /home/developer/cleanRobot/robotbase \
    -u $USER_ID:$USER_GP \
    qrob_env
```
  
容器部署好后会进入容器内的bash终端.此时需要exit退回到host机上,容器会自动停止运行, 需要执行`docker container start s8penv`使得名字为s8penv的容器长期运行.  
  
由于已经指定了Workdir为/home/developer/cleanRobot/robotbase, 现在就可以直接调用`sudo docker exec -it s8penv ./build.sh qiot` 触发一次s8p分支的编译.  
  
X200项目编译环境:  
如果需要同时进行x200的编译, 则推荐将thirdparty文件夹复制到其他位置, 切换成x200分支. robotbase可以使用同一文件夹, 但要在编译前自行切换为x200分支.  
使用以下脚本进行x200编译环境的部署, THIRD_DIR, ROBBASE_DIR 以及容器的名字 --name和前文脚本不同.  
  
x200env_deploy.sh  
  
``` bash
#!/bin/bash
  
THIRD_DIR="替换为x200分支的thirdparty目录"
ROBBASE_DIR="替换为x200分支的robotbase目录"
USER_ID=$(id -u)
USER_GP=$(id -g)
sudo docker run -it\
    --name x200env \
    --mount type=bind,source=$THIRD_DIR,target=/home/developer/cleanRobot/thirdparty \
    --mount type=bind,source=$ROBBASE_DIR,target=/home/developer/cleanRobot/robotbase \
    -e LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib:/usr/local/lib64 \
    -e ARM64_GCC_HOME=/home/developer/toolchain/aarch64_linux \
    -w /home/developer/cleanRobot/robotbase \
    -u $USER_ID:$USER_GP \
    qrob_env
```
  
同样需要exit回宿主机执行`docker container start x200env`, 然后通过`sudo docker exec -it x200env ./build.sh qiot` 触发编译  
  
##  4. 导出镜像
  
  
为了能将制作好的镜像进行导出, 可以通过`docker save qrob_env:latest > qrob_env.tar` 将已经制作好的镜像打包成tar压缩包. 导入时参考上一步.  
如果是想将已经修改内容的容器进行打包, 则要使用docker export 和 docker import进行操作.  
  
##  参考资料
  
  
1. [llvm官方下载](https://apt.llvm.org/ )  
2. [mac上的可调试编译环境](https://blog.csdn.net/m0_37407587/article/details/109639434 )  
  