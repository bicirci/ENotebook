  
  
#  Mcheck进行malloc内存调试
  
  
  
##  1.linux的堆栈结构
  
  
大致上堆(Heap)从低地址向高地址发展, 栈(Stack)从高地址向低地址增长.  
  
![Linux系统内存分析(及Glibc内存管理)-2021-10-14-Linux_memory](https://cdn.jsdelivr.net/gh/bicirci/PicBed@master/MarkDown/Linux系统内存分析(及Glibc内存管理 )-2021-10-14-Linux_memory.jpg)
  
##  2.malloc的作用
  
  
系统的内存分配是通过brk()这一system call实现的. 由于system call比较耗时, 所以诞生了malloc.  
  
对于==小块内存==的申请, malloc先统一申请一块较大的内存, 然后将其进行分割, 再划分给应用. 这样多次小内存的调用只需要呼叫一次brk()即可.同理, 释放时free也不会立即将内存通过brk返回给系统, 而是等空闲的连续内存积累到一定程度, 才统一返回给系统.  
  
而对于==大块的内存==申请, malloc则统一通过mmap()这一system call来实现, 通过独立管理, 避免大小内存块间隔, 小内存块占用而无法释放大内存块的情况.(mmap可以寻址释放, 而brk不行?)  
  
##  3.malloc的状态查看
  
  
通过mallinfo() 和 malloc_stats()可以查看malloc的情况, 前者直接打印到stderr, 后者返回一个可查看的结构体.  
  
> Arena 0:
system bytes     =     205892
in use bytes     =     101188
Total (incl. mmap):
system bytes     =     205892   //通过brk申请的总值
in use bytes     =     101188   //malloc管理的正在使用中的部分
max mmap regions =          0   //mmap申请的大块内存数目
max mmap bytes   =          0   //mmap申请的总内存大小
  
##  4.其他接口
  
  
除了以上接口, 还有其他的内存相关接口:  
  
malloc_usable_size():查看实际可用的memory block(因为对齐的缘故, 比如申请30byte 实际会分配36byte的内存)  
mallopt():设置malloc的一些默认行为.比如可以设置malloc_trim的触发阈值. 要注意这些选项也可以通过环境变量设置.  
  
| mallopt() option | Env var                | Default value | Notes        |
| ---------------- | ---------------------- | ------------- | ------------ |
| M_TRIM_THRESHOLD | MALLOC_TRIM_THRESHOLD_ | 128KB         | -1U disables |
| M_TOP_PAD        | MALLOC_TOP_PAD_        | 0             |              |
| M_MMAP_THRESHOLD | MALLOC_MMAP_THRESHOLD_ | 128KB         | 0 disables   |
| M_MMAP_MAX       | MALLOC_MMAP_MAX_       | 64            | 0 disables   |
[详细说明](https://www.linuxjournal.com/files/linuxjournal.com/linuxjournal/articles/063/6390/6390s2.html )
  
需要注意的是, Trim_threshold的减少,可以显著的增加内存回收的几率, 而不是一直在malloc内部被占用.  
  
##  5. mcheck调试内存污染memory corruption
  
  
内存污染通常有两种情况, 1.超出了内存的允许范围(通常会报Segmentation fault) 2.同一块内存被共用污染, 这种情况一旦出现, 很难debug, 一定要尽早发现.  
  
通过设置环境变量,可以自动检查是否存在内存问题.  
  
`MALLOC_CHECK_=1 my_prog`  
  
设置为1-打印错误但不停止程序 2-退出程序,但无任何打印 3-打印错误并退出  
  
这种自动检测仅仅在下次调用malloc或free时才会触发, 而不是在写入污染处触发, 也就是说可能能知道是哪个pointer出的问题, 但不知道是在哪重复写入的.  
  
或与也可以通过mcheck() 搭配 mcheck_check_all()-全局检测 或 mprobe()-针对某个指针检测 进行手动检测内存情况.mcheck需要在程序第一次调用malloc前使用,其作用是向内存库内注册一个错误发生时的回调函数.然后在代码中调用mcheck_check_all() 或 mprobe(), 如果检测到内存异常,就会触发这个注册的回调.  
  
``` c
#include <mcheck.h>
  
void mcheck_func_cb(enum mcheck_status mc_stats)
{
    printf("[%s:%d]mcheck_status = %d", __FILE__, __LINE__, mc_stats);
}
  
int main(int argc, char* argv[]){
    int ret = mcheck(mcheck_func_cb);
    ......
    mprobe(string_ptr); //进行手动触发检测
}
```
  
[回调中mcheck_status的种类](https://www.linuxjournal.com/files/linuxjournal.com/linuxjournal/articles/063/6390/6390s3.html )
[使用案例](https://blog.csdn.net/slvher/article/details/10453547 )
  
除了自动检测和手动触发外, 还有第三种方式, 即通过链接libmcheck实现检测. 当编译时添加了-lmcheck选项后, 可以自动在初次调用 malloc前注册默认的mcheck回调, 这种方式可以用于对于进入main函数前就已经进行内存分配的程序.
  
##  6. mtrace进行内存调试
  
  
相比较mcheck, GNU C还提供了一个可追踪内存历史的调试接口mtrace. 所有的heap操作都会记录在一个文件内(可通过MALLOC_TRACE进行自定义), 当程序执行完成后, 会通过一个Perl脚本进行分析,进而找出内存问题.  
  
所以希望进行全局记录的话, 只调用mtrace()即可. 但如果只想对某一段代码进行调试, 也可以通过muntrace进行停止记录,并立即进行分析.  
  
需要特别注意的是,如果程序发生崩溃, 则不会调用对trace文件的分析, 所以通常做法是将mun
trace注册到崩溃的Signal Handler中, 从而可以在崩溃前先调用trace文件的分析脚本.  
  
``` c
#include <stdio.h>
#include <stdlib.h>
#include <mcheck.h>
#include <signal.h>
  
void handler(int s) {
  muntrace();
  abort();
}
  
int main() {
  char *ptr;
  
  signal(SIGSEGV, handler);
  mtrace();
  ptr = malloc(10);
  free(ptr);
  free(ptr);
  
}
```
  
##  参考文档
  
  
[Advanced Memory Allocation 2003
](https://www.linuxjournal.com/article/6390)
  
  