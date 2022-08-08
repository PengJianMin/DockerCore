# DockerCore
+ 操作系统（内核）也是**一支程序**
+ 在**编写代码时**使用**系统调用**就是在**使用**这支程序的功能
+ 内核开发和应用开发是**不同层面**的东西，应用开发基于内核提供的功能，内核开发则是基于**硬件**提供的功能
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
                fpid = fork(); //调用fork()
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
+ UTS namespace（UNIX Time-sharing System）：提供了主机名和域名的隔离
1. 每个Docker容器就可以拥有**独立的主机名和域名**了，在**网络上**可以被视作一个**独立的节点**，**而非宿主机上的一个进程**，**虽然**本质上还是进程
2. ```  #define _GNU_SOURCE
        #include<sys/types.h>
        #include<sys/wait.h>
        #include<stdio.h>
        #include<sched.h>
        #include<signal.h>
        #include<unistd.h>

        #define STACK_SIZE (1024*1024)//中间有空格
        static char child_stack[STACK_SIZE];
        char * const child_args[] = { "/bin/bash",NULL}; //第二个参数只能是NULL

        int child_main(void* args){
            printf("now in Child process");
            sethostname("Elesev's New Namespace",100); // 效果[user@hostname]-->[root@Elesev's New elesev] 
            execv(child_args[0],child_args);
            return 1;
        }

        int main(){
            printf("Program start! \n");
            int child_pid = clone(child_main,child_stack+STACK_SIZE,CLONE_NEWUTS | SIGCHLD,NULL);//插入uts隔离的flag
            waitpid(child_pid,NULL,0);
            printf("Child pid is:%d\n",child_pid);
            printf("Already quit\n");
            return 0;
        }
3. IPC namespace（Inter-Process Communication, IPC）:**进程间通信**，IPC资源包括常见的**信号量、消息队列和共享内存**
    + 在同一个IPC namespace下的**进程彼此可见**，不同IPC namespace下的进程则互相不可见
    + 申请IPC资源就申请了一个**全局唯一**的32位ID，所以IPC namespace中实际上包含了**系统IPC标识符**以及**实现POSIX消息队列的文件系统**
    + 宿主机上执行
        + `ipcmk -Q` 创建一个message queue
        + `ipcs -q` 看到已经开启的message queue
    + 在新建的**子进程**中调用的shell中执行ipcs -q查看message queu，子进程**找不到原先声明**的message queue了，**已经实现**了IPC的隔离
    + ```   #define _GNU_SOURCE
            #include<sys/types.h>
            #include<sys/wait.h>
            #include<stdio.h>
            #include<sched.h>
            #include<signal.h>
            #include<unistd.h>

            #define STACK_SIZE (1024*1024)
            static char child_stack[STACK_SIZE];
            char * const child_args[] = { "/bin/bash",NULL};

            int child_main(void* args){
                printf("now in Child process");
                sethostname("Elesev's New Namespace",12);
                execv(child_args[0],child_args);
                return 1;
            }

            int main(){
                printf("Program start! \n");
                int child_pid = clone(child_main,child_stack+STACK_SIZE,CLONE_NEWIPC|CLONE_NEWUTS | SIGCHLD,NULL); // 加入IPC隔离的flag
                waitpid(child_pid,NULL,0);
                printf("Child pid is:%d\n",child_pid);
                printf("Already quit\n");
                return 0;
            }
4. PID namespace：对进程PID**重新标号**，即两个**不同namespace下**的进程可以有**相同的PID**
    + **内核**为所有的PID namespace维护了一个**树状结构**
        + 最顶层的是**系统初始时**创建的，被称为 root namespace
        + root namespace创建的**新PID namespace**被称为child namespace（**树的子节点**）
        + **原先的**PID namespace就是新创建的PID namespace的parent namespace（**树的父节点**）
        + 父节点可以看到子节点中的进程，并可以通过信号等方式对子节点中的进程**产生影响**
        + 反过来，子节点却**不能看到**父节点PID namespace中的任何内容
    + **每个**PID namespace中的**第一个进程**“PID 1”，都会像**传统Linux中的init进程**一样拥有**特权**，起特殊作用
    + 一个namespace中的进程，不可能通过kill或ptrace影响**父节点**或者**兄弟节点**中的进程，因为其他节点的PID在这个namespace中**没有任何意义**
    + 如果你在新的PID namespace中**重新挂载**/proc文件系统，会发现其下只显示**同属一个PID namespace中**的**其他进程** `mount -t proc /proc`
    + 在**root namespace中**可以看到所有的进程，并且**递归包含**所有子节点中的进程（**父可以看子**）
        + Docker daemon 下的所有docker进程都是其namespace的子进程，所以可以查看
        + 在**外部监控**Docker中运行程序的方法了，就是监控**Docker daemon（server）** **所在的**PID namespace下的所有**进程及其子进程**，再进行筛选即可
    + ```   #define _GNU_SOURCE
            #include<sys/types.h>
            #include<sys/wait.h>
            #include<stdio.h>
            #include<sched.h>
            #include<signal.h>
            #include<unistd.h>

            #define STACK_SIZE (1024*1024)
            static char child_stack[STACK_SIZE];
            char * const child_args[] = { "/bin/bash",NULL};

            int child_main(void* args){
                printf("now in Child process");
                sethostname("Elesev's New Namespace",12);
                execv(child_args[0],child_args);
                return 1;
            }

            int main(){
                printf("Program start! \n");
                int child_pid = clone(child_main,child_stack+STACK_SIZE,CLONE_NEWPID|CLONE_NEWIPC|CLONE_NEWUTS | SIGCHLD,NULL);//加入pid隔离的flag
                waitpid(child_pid,NULL,0);
                printf("Child pid is:%d\n",child_pid);
                printf("Already quit\n");
                return 0;
            }
    + 效果：`echo $$` 发现当前进程pid为1
        + 在**子进程**的shell中执行了ps aux/top之类的命令，发现还是可以看到**所有父进程的PID**，那是因为**还没有对文件系统挂载点进行隔离**，ps/top之类的命令调用的是真实系统下的/proc文件内容，看到的自然是所有的进程 `mount -t proc proc /proc`
    + PID namespace中的**init进程**（每一个**独立的**namespace中**都会有**）
        + 在**传统的Unix系统**中，PID为1的进程是init，地位非常特殊。它作为**所有进程**的父进程，维护一张**进程表**，不断检查进程的状态，一旦有某个子进程因为**父进程错误**成为了“孤儿”进程，init就会负责**收养这个子进程**并最终**回收资源**，**结束进程**
        + 容器中，启动的第一个进程（init进程）也需要实现类似init的功能，**维护**所有后续启动进程的运行状态
            + 当系统中存在树状嵌套结构的PID namespace时，若某个子进程成为孤儿进程，收养该子进程的责任就交给了该子进程**所属的PID namespace中的init进程**
            + 如果**确实需要**在**一个Docker容器中运行多个进程**，**最先启动的命令进程**应该是具有资源监控与回收等管理能力的，如**bash**
    + 信号与init进程
        + 如果init中**没有编写**处理某个信号的代码逻辑，那么与init在**同一个PID namespace下**的进程（**即使有超级权限**）发送给它的该信号都会被屏蔽。这个功能的主要作用是防止init进程被误杀（信号屏蔽）
        + **父节点中的进程**发送的信号，如果不是SIGKILL（销毁进程）或SIGSTOP（暂停进程）**也会被忽略**。
        + 一旦**init进程被销毁**，同一PID namespace中的**其他进程**也随之接收到SIGKILL信号而被销毁。
            + **理论上**，该PID namespace也不复存在了
            + 但是如果/proc/[pid]/ns/pid处于**被挂载**或者**打开状态**，namespace就会被**保留下来**
            + 然而，保留下来的namespace**无法通过setns()或者fork()创建进程**，所以实际上并**没有什么作用**
        + 当**一个容器内存在多个进程时**，**容器内的**init进程可以对信号进行捕获，当SIGTERM或SIGINT等信号到来时，对其子进程做信息保存、资源回收等处理工作
    + 挂载proc文件系统
        + `mount -t proc proc /proc`
        + 如果在新的PID namespace中使用ps命令查看，看到的**还是所有的进程**，因为与PID直接相关的/proc文件系统（procfs）没有挂载到一个**与原/proc不同**的位置
        + 此时并**没有进行mount namespace的隔离**，所以该操作实际上已经**影响了root namespace的文件系统**。当退出新建的PID namespace以后，再执行ps a时，就会发现出错，**再次执行** `mount -t proc proc /proc` 可以**修复错误**
    +  unshare()和setns() 在PID namespace中使用时**需要注意**：
        + 创建其他namespace时unshare()和setns()会**直接进入新的**namespace，而唯独PID namespace**例外**
        + unshare()允许用户在**原有进程中建立**命名空间进行隔离。但创建了PID namespace后，原先**unshare()调用者进程**并**不进入**新的**PID namespace**，**接下来创建的子进程**才会进入新的namespace，这个**子进程也就随之成为新namespace中的init进程**
        + 调用setns()创建新PID namespace时，**调用者进程**也不进入新的**PID namespace**，而是**随后创建的子进程进入**
        + 原因：
            + 调用getpid()函数得到的PID是**根据调用者所在的PID namespace而决定返回哪个PID**，进入新的PID namespace会导致PID产生变化。而对用户态的程序和库函数来说，它们都认为进程的PID是一个常量，PID的变化会引起这些**进程崩溃**
            + 一旦程序进程创建以后，那么它的PID namespace的**关系就确定下来**了，**进程不会变更它们对应的PID namespace**
            + **在Docker中**，**docker exec**会使用setns()函数**加入已经存在**的命名空间，但是**最终还是**会调用clone()函数，原因就在于此。
5. mount namespace：**特指mount命令**的**影响（作用、效果）范围**，**不是指文件内容**。通过隔离文件系统**挂载点**对隔离文件系统提供支持，不同mount namespace中的文件结构发生变化互不影响
    + 进程在创建mount namespace时，会把**当前的文件结构（指目录和文件等等）** 复制给新的namespace。新namespace中的所有mount操作都只影响**自身的文件系统**，对外界不会产生任何影响
    + 父节点namespace中的进程挂载了一张CD-ROM，这时子节点namespace复制的目录结构是**无法自动挂载**上这张CD-ROM的，因为这种操作会影响到父节点的文件系统
    + 挂载传播（mount propagation）：定义了挂载对象（mount object）之间的关系（也指**mount namesapce间的关系**），这样的关系包括共享关系和从属关系，系统用这些关系决定任何挂载对象中的挂载事件**如何传播到其他挂载对象**
        + 共享关系（share relationship）。如果两个挂载对象具有共享关系，那么一个挂载对象中的挂载事件会传播到另一个挂载对象，反之亦然
        + 从属关系（slave relationship）。如果两个挂载对象形成从属关系，那么一个挂载对象中的挂载事件会传播到另一个挂载对象，但是反之不行；在这种关系中，从属对象是事件的接收者。
    + 一个挂载状态可能为以下一种：
        + 共享挂载（share）：传播事件的挂载对象称为共享挂载
            + **/lib目录**使用完全的共享挂载传播，**各namespace之间发生的变化**都会互相影响
            + `mount --make-shared <mount-object>`
        + 从属挂载（slave）：接收传播事件的挂载对象称为从属挂载：
            + 上层的mount namespace下的 **/bin目录**与child namespace通过master slave方式进行挂载传播，当mount namespace中的/bin目录发生变化时，发生的挂载事件能够**自动传播**到child namespace中的**/bin目录**
            + `mount --make-slave <shared-mount-object>`
        + 共享/从属挂载（shared and slave）：同时兼有前述两者特征的挂载对象称为共享/从属挂载
            + `mount --make-share <slave-mount-object>`
        + 私有挂载（private）：既不传播也不接收传播事件的挂载对象称为私有挂载
            + **/proc目录**使用私有挂载传播的方式，各mount namespace之间互相隔离
            + `mount --make-private <mount-object>`
        + 不可绑定挂载（unbindable）：它们与私有挂载相似，但是不允许执行绑定挂载，即创建mount namespace时这块文件对象不可被复制
            + **/root目录**一般都是管理员所有，不能让其他mount namespace挂载绑定
            + `mount --make-unbindable <mount-object>`
    + ```   #define _GNU_SOURCE
            #include<sys/types.h>
            #include<sys/wait.h>
            #include<stdio.h>
            #include<sched.h>
            #include<signal.h>
            #include<unistd.h>

            #define STACK_SIZE (1024*1024)
            static char child_stack[STACK_SIZE];
            char * const child_args[] = { "/bin/bash",NULL};

            int child_main(void* args){
                printf("now in Child process");
                sethostname("Elesev's New Namespace",12);
                execv(child_args[0],child_args);
                return 1;
            }

            int main(){
                printf("Program start! \n");
                int child_pid = clone(child_main,child_stack+STACK_SIZE,CLONE_NEWNS|CLONE_NEWPID|CLONE_NEWIPC|CLONE_NEWUTS | SIGCHLD,NULL);//加入mount隔离的flag
                waitpid(child_pid,NULL,0);
                printf("Child pid is:%d\n",child_pid);
                printf("Already quit\n");
                return 0;
            }
    + 效果：子进程进行的**挂载与卸载操作**都将**只作用**于这个mount namespace。**子进程重新挂载**了/proc文件系统，当进程**退出后**，root mount namespace（主机）的/proc文件系统是**不会被破坏**的。
6. network namespace

