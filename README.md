# DockerCore

# Docker背后的内核知识
+ Docker实现方式
1. Docker容器**本质**上是**宿主机上的进程**
2. `namespace`：实现**资源"隔离"**
3. `cgroups`：实现**资源"限制"**
4. 写时复制（copy on write）：写时复制
+ namespace资源隔离（进程是**描述资源**的单位，线程是**描述执行**的单位）
1. Linux内核实现namespace的一个主要目的，就是实现轻量级虚拟化（容器）服务。在同一个namespace下的进程可以**感知彼此的变化**，而**对外界的进程一无所知**。这样就可以让容器中的进程产生错觉，仿佛自己置身于一个独立的系统环境中，以达到**独立和隔离**的目的
2. 完成了一个容器所需要做的**6项隔离**
    + namespace------系统调用参数---------隔离内容
    + UTS------------CLONE_NEWUTS--------主机名与域名
    + IPC------------CLONE_NEWIPC--------信号量、消息队列和共享内存
    + PID------------CLONE_NEWPID--------进程编号
    + NEWWORK--------CLONE_NEWNET--------网络设备、网络栈、端口等
    + MOUNT----------CLONE_NEWNS---------挂载点（文件系统）
    + USER-----------CLONE_NEWUSER-------用户和用户组
+ namespace API操作的4种方式（为了确定隔离的到底是哪6项namespace，在使用这些API时，通常需要指定**系统调用参数中**的**一个或多个**，通过|（位或）操作来实现。）
1. clone()：在创建**新进程**的**同时创建**namespace（**创建**）
    + `int clone(int(*child_func)(void *),void *chile_stack,int flags,void *args)`
        + child_func传入**子进程运行**的**程序主函数**
        + hild_stack传入子进程使用的**栈空间**
        + flags表示使用哪些CLONE_* 标志位（指上述6种）
        + args则可用于传入用户参数
2. /proc下的部分文件
    + 在/proc/[pid]/ns文件下看到**指向**不同**namespace号**的**文件**（**link文件**）
    + 如果两个进程指向的namespace编号**相同**，就说明它们在**同一个namespace**下，否则便在不同namespace里面
    + link文件一旦被打开，只要**文件描述符（fd）** 存在，就算该namespace下的所有**进程都已经结束**，这个namespace也会**一直存在**，**后续进程**也可以再**加入**进来
    + 在Docker中，通过文件描述符**定位和加入**一个**存在的**namespace是**最基本**的方式
    + **bind方式挂载**
        + 在**进程都结束**的情况下，也可以通过挂载的形式把namespace**保留**下来，保留namespace的目的是为以后有进程加入**做准备** 
        + `mount --bind /proc/27514/ns/uts ~/uts` 使用～/uts**文件**来代替/proc/27514/ns/uts
3. setns()：进程从**原先的**namespace**加入**某个**已经存在**的namespace（**加入**）
    + `int setns(int fd, int nstype)`
        + 参数fd表示要加入namespace的文件描述符。它是一个指向/proc/[pid]/ns目录的文件描述符，可以通过**直接打开**该目录下的**链接**或者打开一个**挂载**了该目录下链接的文件得到
        + 参数nstype让调用者可以检查fd指向的namespace类型是否符合实际要求。该参数为0表示不检查。
    + Docker中，使用**docker exec命令**在**已经运行**着的容器中执行一个**新的命令**，就需要用到该方法
    + 通常为了不影响**进程的调用者**，也为了使新加入的pid namespace**生效**，会**在setns()函数执行后使用clone()创建子进程**继续执行命令，让**原先的进程结束运行**
    + `execve()系列函数`：将新加入的namespace利用起来，执行用户命令 `execep("/bin/bash",null)`
4. unshare()：在**原先进程**上**进行**namespace**隔离**（**跳出**）
    + `int unshare(int flags)`
    + **不启动新进程**就可以起到隔离的效果，相当于**跳出原先**的namespace进行操作
    + Docker**目前并没有使用**这个系统调用
5. 拓展：fork()系统调用
    + 系统调用函数fork()并**不属于**namespace的API
    + 当程序调用fork()函数时，系统会创建新的进程，为其**分配资源**，例如**存储数据和代码的空间**，然后把原来进程的**所有值都复制**到新进程中，只有少量数值与原来的进程值不同，相当于**复制了本身**
    + 程序的后续代码逻辑要如何**区分自己是新进程还是父进程**呢？：fork()的神奇之处在于它**仅仅被调用一次**，却能够**返回两次**（**在父进程与子进程中各返回一次**），通过**返回值的不同**就可以区分父进程与子进程。它可能有以下3种不同的返回值：
        + 在父进程中，fork()返回**新创建子进程**的进程ID
        + 在子进程中，fork()**返回0**
        + 如果出现**错误**，fork()返回一个**负值**
        + ```#include<unistd.h>
             #include<stdio.h>
             int main(){
                pid_t fpid;
                int count = 0;
                fpid = fork();
                if(fpid<0){
                    printf("error in fork!");
                }else if(fpid==0){
                    printf("i am child, pid is %d\n",getpid());
                }else{
                    printf("i am parent, pid is %d\n",getpid());
                }
                return 0; 
             } 
        + `gcc -Wall fork_example.c && ./aout`            
 
