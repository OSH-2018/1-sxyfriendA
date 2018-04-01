# OSH LAB 01
于颖奇PB16110846
## 目录
1. [工具及实验环境](#index1)
2. [实验步骤](#index2)
     * [配置实验环境](#index3)
     * [使用gdb调试操作系统启动过程](#index4)
3. 实验小结

## <span id="index1">工具及实验环境</span>
* 工具：gdb、qemu、kernel Linux-4.15.14
* 环境：Fedora 25

## <span id="index2">实验步骤</span>

### <span id="index3">配置实验环境</span>

#### 下载gdb及qemu
在shell中输入以下代码并使用管理员权限执行即可

    sudo dnf install gdb
    sudo dnf install qemu    

#### 下载并编译内核
由[https://www.kernel.org/](https://www.kernel.org/)下载最新版的内核Linux4.15.15并将其解压编译
为使调试文件可用 输入并执行

    make menuconfig
    
系统提示

    install ncurses(ncurses-devel) and try again
    
根据系统提示安装指定文件，输入并执行
  
    sudo dnf install ncurses-devel
    
之后在shell中输入

    make menuconfig
    
并在配置内核时，将“kernel hacking”中的“compile the kernel with debug info”，使得调试信息可用，为之后的gdb调试做准备。

接下来对内核进行编译，在对应目录下输入

    make -j4
    
等待便以完成即可

至此gdb、qemu以及Linux内核准备完毕

### <span id="index4">使用gdb调试操作系统启动过程</span>

#### 使用qemu运行内核

在Linux的shell中输入

    qemu-system-x86_64 -kernel ./bzImage ./initrd.img -smp 2 -gdb tcp::1234 -S -append "nokaslr"
    
以上代码使选择qemu选择编译好的内核 bzImage运行，运行于虚拟硬盘initrd.img。

 -initrd：用来指定内核启动时使用的ram disk。
 
 -gdb tcp::1234：表示启动gdbserver，并在tcp的1234端口监听，-S表示在开始的时候冻结CPU直到远程的gdb输入相应的控制命令
 
 -S:启动内核时停止，等待调试器命令
 
 添加启动参数nokaslr是为了在调试内核时关闭kaslr，这样在解压时可以解压缩到内存中的固定位置，否则会解压到随机内存地址
 
 打开虚拟机如下图所示
 ![picture1](http://m.qpic.cn/psb?/V10tKyT84U4n1M/JixeF0urBk4*6j19Z1iWpCQkbd75h4t1JO3cY*AU8ds!/b/dEABAAAAAAAA&bo=GAXrAQAAAAARB8c!&rf=viewer_4)
 
 #### 启动gdb调试内核
 另外打开一个shell窗口运行gdb
 
 首先启动gdb，并和Qemu进行联系。
 将以下命令行在shell中输入并执行
 
    gdb
    (gdb) file linux-4.15.14/vmlinux
    (gdb) target remote:1234
    
以上三行代码第一行用于启动gdb,第二行代码用于在gdb界面中targe.remote之前加载符号表,建立gdb和gdbserver之间的连接,按c即可让qemu上的Linux继续运行

输入并执行,查看当前寄存器状态

    (gdb) info registers
    
如下所示
![picture2](http://m.qpic.cn/psb?/V10tKyT84U4n1M/HCCnbr0rOzhLBOqVkBQdpg4XTZU2VqOr5HRGjT5FHI8!/b/dF4BAAAAAAAA&bo=HgXvAQAAAAARF9U!&rf=viewer_4)
由上图可知此时寄存器CS与IP值均为xfff0,则其指向的位置地址为 0xfffffff0 是 4GB - 16 字节。 这个地方是 复位向量(Reset vector) 。 这是CPU在重置后期望执行的第一条指令的内存地址。它包含一个 jump 指令，这个指令通常指向BIOS入口点。

##### start_kernel
接下来，我们设置第一个断点，断点位置设置在start_kernel处，输入并执行

    (gdb) break strat_kernel
    (gdb) c
    (gdb) list 
    
![picture3](http://m.qpic.cn/psb?/V10tKyT84U4n1M/.UvTsUfjoErqbtXyzYnGofNG.MYulMyqVGBjBpUzXAQ!/b/dEIBAAAAAAAA&bo=ZgXoAQAAAAARF6o!&rf=viewer_4)
由图中信息可知，init/main.c的第515行,执行list命令后出现start_kernel附近的代码，执行

    (gdb) info registers
    
![picture4](http://m.qpic.cn/psb?/V10tKyT84U4n1M/EZ8XdwfbGYkCxmeqBoTBG7fP5DM92YDLWCmZU7PgWAQ!/b/dJUAAAAAAAAA&bo=bwXsAQAAAAARF6c!&rf=viewer_4)
通过阅读start_kernel源码

    asmlinkage void __init start_kernel(void) 
    { 
    char * command_line; 
    unsigned long mempages; 
    extern char saved_command_line[]; 
    lock_kernel(); 
    printk(linux_banner); 
    setup_arch(&command_line); //arm/kernel/setup.c 
    printk("Kernel command line: %s\n", saved_command_line); 
    parse_options(command_line); 
    ……
    
参考[Linux内核修炼之道 start_kernel函数](http://book.51cto.com/art/201007/213598.htm)中代码的注释，我们可知start_kernel()函数进行了一系列初始化函数，对内核进行大量的初始化操作。包括以下各项重要事件。

1. 关闭当前CPU的中断
2. 每一个中断都有一个中断描述符来进行描述，设置所有中断描述符的锁
3. 获取大内核锁，锁定整个内核。
4. 初始化页地址，使用链表将其链接起来
5. 每个CPU分配pre-cpu结构内存
6. 进程调度器初始化
7. 初始化RCU(Read-Copy Update)机制
8. 初始化hash表
9. 初始化定时器相关的数据结构
10. 对高精度时钟进行初始化
11. 对内核的profile功能进行初始化
12. 初始化控制台以显示printk的内容
13. 虚拟文件系统的初始化
14. slab初始化
15. 为有关的管理机制建立专用的slab缓存
16. 创建init进程

start_kernel函数中各初始化顺序不能随意调换，否则可能出现严重问题。

##### boot_cpu_init
将第二个断点设置在函数boot_cpu_init处，并使用list以及info register查看其附近的代码以及，运行到该处时各寄存器中的值，运行结果如下所示
![picture5](http://m.qpic.cn/psb?/V10tKyT84U4n1M/p2kMbgDy1P5U0u4qyE5ZwLAat7NC1FpeZp3VDgL3cXs!/b/dEIBAAAAAAAA&bo=QgXoAQAAAAARF44!&rf=viewer_4)
![picture6](http://m.qpic.cn/psb?/V10tKyT84U4n1M/WXqA26faG247wCa.Y1hOGNXyZVIhSUZ028StpN2QvG8!/b/dEMBAAAAAAAA&bo=OgXsAQAAAAARF*I!&rf=viewer_4)

这个函数是第一个CPU事件，主要作用是设置当前引导系统的CPU在物理上存在。内核中使用cpu_oresent_map位图表达有多少个CPU，使用cpu_online_map位图来表示那些CPU可以运行内核代码和接受中断处理，使用cpu_possible_map位图，表示最多可以使用多少个CPU。该函数即可一次设置三个位图标志，使得引导系统的CPU物理存在，初始化好最少需要运行的CPU

##### setup_arch
将第三个断点设置在函数setup_arch处，并使用list以及info register查看其附近的代码以及，运行到该处时各寄存器中的值，运行结果如下所示
![picture7](http://m.qpic.cn/psb?/V10tKyT84U4n1M/lNZfbBmMh4*mX8UcW6eBzUdHnxhPgM5Nk0QkM8XY4ko!/b/dF4BAAAAAAAA&bo=PAXqAQAAAAARF*I!&rf=viewer_4)
![picture8](http://m.qpic.cn/psb?/V10tKyT84U4n1M/c1JO74MBI2pP.rqMQdNObhKtMyOJ8rPUKrKDJClaXv0!/b/dPIAAAAAAAAA&bo=PgXqAQAAAAARF*A!&rf=viewer_4)
这个函数主要作用是对内核架构进行初始化。实现了对kernel中各全局变量的赋值，memory与cache的初始化及为之后的操作进行准备工作，请求后续的资源。


##### rest_init
将第四个断点设置在函数rest_init处，并使用list以及info register查看其附近的代码以及，运行到该处时各寄存器中的值，运行结果如下所示
![picture7](http://m.qpic.cn/psb?/V10tKyT84U4n1M/rvfqxO88h65Qm4n5dg6nKq7WrmKBo6tvmv5UC4AGW7Y!/b/dJUAAAAAAAAA&bo=PwXoAQAAAAARF*M!&rf=viewer_4)
![picture8](http://m.qpic.cn/psb?/V10tKyT84U4n1M/nq4*3aYhljmp.UOV4ez.5V4a8ohYTOGDwHCNLKnlU0w!/b/dJUAAAAAAAAA&bo=QwXoAQAAAAADF50!&rf=viewer_4)
主要任务为启动另外两个线程kernel_init()以及kthreadd()。执行完该函数后start_kernel()执行完毕，操作系统开始运行

## <span id="index2">实验小结</span>

本次实验难度较大，在本次实验前对Linux中各命令行的使用较为生疏，且对内核的启动过程知之甚少。通过本次实验，熟悉了命令行的使用方法，且对内核启动中关键事件有了进一步的了解，并提高了阅读源码的能力，虽然因时间原因且能力有限无法阅读所有源码，但结合注视及所查阅的资料，基本了解了start_kernel()及其中关键函数如set_arch(),rest_init()等的实现方式，为今后进一步理解内核启动奠定基础。