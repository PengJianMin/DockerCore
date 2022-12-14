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
# namespace资源隔离（进程是**描述资源**的单位，线程是**描述执行**的单位）
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
+ IPC namespace（Inter-Process Communication, IPC）:**进程间通信**，IPC资源包括常见的**信号量、消息队列和共享内存**
1. 在同一个IPC namespace下的**进程彼此可见**，不同IPC namespace下的进程则互相不可见
2. 申请IPC资源就申请了一个**全局唯一**的32位ID，所以IPC namespace中实际上包含了**系统IPC标识符**以及**实现POSIX消息队列的文件系统**
3. 宿主机上执行
    + `ipcmk -Q` 创建一个message queue
    + `ipcs -q` 看到已经开启的message queue
4. 在新建的**子进程**中调用的shell中执行ipcs -q查看message queu，子进程**找不到原先声明**的message queue了，**已经实现**了IPC的隔离
5. ```   #define _GNU_SOURCE
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
+ PID namespace：对进程PID**重新标号**，即两个**不同namespace下**的进程可以有**相同的PID**
1. **内核**为所有的PID namespace维护了一个**树状结构**
    + 最顶层的是**系统初始时**创建的，被称为 root namespace
    + root namespace创建的**新PID namespace**被称为child namespace（**树的子节点**）
    + **原先的**PID namespace就是新创建的PID namespace的parent namespace（**树的父节点**）
    + 父节点可以看到子节点中的进程，并可以通过信号等方式对子节点中的进程**产生影响**
    + 反过来，子节点却**不能看到**父节点PID namespace中的任何内容
2. **每个**PID namespace中的**第一个进程**“PID 1”，都会像**传统Linux中的init进程**一样拥有**特权**，起特殊作用
3. 一个namespace中的进程，不可能通过kill或ptrace影响**父节点**或者**兄弟节点**中的进程，因为其他节点的PID在这个namespace中**没有任何意义**
4. 如果你在新的PID namespace中**重新挂载**/proc文件系统，会发现其下只显示**同属一个PID namespace中**的**其他进程** `mount -t proc /proc`
5. 在**root namespace中**可以看到所有的进程，并且**递归包含**所有子节点中的进程（**父可以看子**）
    + Docker daemon 下的所有docker进程都是其namespace的子进程，所以可以查看
    + 在**外部监控**Docker中运行程序的方法了，就是监控**Docker daemon（server）** **所在的**PID namespace下的所有**进程及其子进程**，再进行筛选即可
6. ```   #define _GNU_SOURCE
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
7. 效果：`echo $$` 发现当前进程pid为1
    + 在**子进程**的shell中执行了ps aux/top之类的命令，发现还是可以看到**所有父进程的PID**，那是因为**还没有对文件系统挂载点进行隔离**，ps/top之类的命令调用的是真实系统下的/proc文件内容，看到的自然是所有的进程 `mount -t proc proc /proc`
8. PID namespace中的**init进程**（每一个**独立的**namespace中**都会有**）
    + 在**传统的Unix系统**中，PID为1的进程是init，地位非常特殊。它作为**所有进程**的父进程，维护一张**进程表**，不断检查进程的状态，一旦有某个子进程因为**父进程错误**成为了“孤儿”进程，init就会负责**收养这个子进程**并最终**回收资源**，**结束进程**
    + 容器中，启动的第一个进程（init进程）也需要实现类似init的功能，**维护**所有后续启动进程的运行状态
        + 当系统中存在树状嵌套结构的PID namespace时，若某个子进程成为孤儿进程，收养该子进程的责任就交给了该子进程**所属的PID namespace中的init进程**
        + 如果**确实需要**在**一个Docker容器中运行多个进程**，**最先启动的命令进程**应该是具有资源监控与回收等管理能力的，如**bash**
9. 信号与init进程
    + 如果init中**没有编写**处理某个信号的代码逻辑，那么与init在**同一个PID namespace下**的进程（**即使有超级权限**）发送给它的该信号都会被屏蔽。这个功能的主要作用是防止init进程被误杀（信号屏蔽）
    + **父节点中的进程**发送的信号，如果不是SIGKILL（销毁进程）或SIGSTOP（暂停进程）**也会被忽略**。
    + 一旦**init进程被销毁**，同一PID namespace中的**其他进程**也随之接收到SIGKILL信号而被销毁。
        + **理论上**，该PID namespace也不复存在了
        + 但是如果/proc/[pid]/ns/pid处于**被挂载**或者**打开状态**，namespace就会被**保留下来**
        + 然而，保留下来的namespace**无法通过setns()或者fork()创建进程**，所以实际上并**没有什么作用**
    + 当**一个容器内存在多个进程时**，**容器内的**init进程可以对信号进行捕获，当SIGTERM或SIGINT等信号到来时，对其子进程做信息保存、资源回收等处理工作
10. 挂载proc文件系统
    + `mount -t proc proc /proc`
    + 如果在新的PID namespace中使用ps命令查看，看到的**还是所有的进程**，因为与PID直接相关的/proc文件系统（procfs）没有挂载到一个**与原/proc不同**的位置
    + 此时并**没有进行mount namespace的隔离**，所以该操作实际上已经**影响了root namespace的文件系统**。当退出新建的PID namespace以后，再执行ps a时，就会发现出错，**再次执行** `mount -t proc proc /proc` 可以**修复错误**
11.  unshare()和setns() 在PID namespace中使用时**需要注意**：
    + 创建其他namespace时unshare()和setns()会**直接进入新的**namespace，而唯独PID namespace**例外**
    + unshare()允许用户在**原有进程中建立**命名空间进行隔离。但创建了PID namespace后，原先**unshare()调用者进程**并**不进入**新的**PID namespace**，**接下来创建的子进程**才会进入新的namespace，这个**子进程也就随之成为新namespace中的init进程**
    + 调用setns()创建新PID namespace时，**调用者进程**也不进入新的**PID namespace**，而是**随后创建的子进程进入**
    + 原因：
        + 调用getpid()函数得到的PID是**根据调用者所在的PID namespace而决定返回哪个PID**，进入新的PID namespace会导致PID产生变化。而对用户态的程序和库函数来说，它们都认为进程的PID是一个常量，PID的变化会引起这些**进程崩溃**
        + 一旦程序进程创建以后，那么它的PID namespace的**关系就确定下来**了，**进程不会变更它们对应的PID namespace**
        + **在Docker中**，**docker exec**会使用setns()函数**加入已经存在**的命名空间，但是**最终还是**会调用clone()函数，原因就在于此。
+ mount namespace：**特指mount命令**的**影响（作用、效果）范围**，**不是指文件内容**。通过隔离文件系统**挂载点**对隔离文件系统提供支持，不同mount namespace中的文件结构发生变化互不影响
1. 进程在创建mount namespace时，会把**当前的文件结构（指目录和文件等等）** 复制给新的namespace。新namespace中的所有mount操作都只影响**自身的文件系统**，对外界不会产生任何影响
2. 父节点namespace中的进程挂载了一张CD-ROM，这时子节点namespace复制的目录结构是**无法自动挂载**上这张CD-ROM的，因为这种操作会影响到父节点的文件系统
3. 挂载传播（mount propagation）：定义了挂载对象（mount object）之间的关系（也指**mount namesapce间的关系**），这样的关系包括共享关系和从属关系，系统用这些关系决定任何挂载对象中的挂载事件**如何传播到其他挂载对象**
    + 共享关系（share relationship）。如果两个挂载对象具有共享关系，那么一个挂载对象中的挂载事件会传播到另一个挂载对象，反之亦然
    + 从属关系（slave relationship）。如果两个挂载对象形成从属关系，那么一个挂载对象中的挂载事件会传播到另一个挂载对象，但是反之不行；在这种关系中，从属对象是事件的接收者。
4. 一个挂载状态可能为以下一种：
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
5. ```   #define _GNU_SOURCE
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
6. 效果：子进程进行的**挂载与卸载操作**都将**只作用**于这个mount namespace。**子进程重新挂载**了/proc文件系统，当进程**退出后**，root mount namespace（主机）的/proc文件系统是**不会被破坏**的。
+ network namespace：提供了关于**网络资源**的隔离，包括网络设备、IPv4和IPv6协议栈、IP路由表、防火墙、/proc/net目录、/sys/class/net目录、套接字（socket）等
1. 一个物理的网络设备**最多存在于一个network namespace中（类似pid的namespace无法改变）**，可以通过创建**veth pair**（**虚拟网络设备对**：有两端，类似管道，如果数据从一端传入另一端也能接收到，反之亦然）在不同的network namespace间**创建通道**，以达到通信目的
2. 一般情况下，物理网络设备都分配在**最初的root namespace**（表示系统默认的namespace）中。但是如果有多块物理网卡，也可以把其中一块或多块**分配给新创建**的network namespace
3. 注意：当新创建的network namespace**被释放时**（所有内部的进程都终止**并且**namespace文件没有被挂载或打开），在这个namespace中的物理网卡会**返回到root namespace**，**而非创建该进程的父进程**所在的network namespace
4. veth pair（**虚拟网络设备对**）
    + 一端放置在新的namespace中，通常命名为**eth0**，一端放在原先的namespace中**连接物理网络设备**，再通过把多个**设备**接入**网桥**或者进行**路由转发**，来实现通信的目的
    + 在建立veth pair之前，新旧namespace该如何通信呢？答案是pipe（管道）。
        + 以Docker daemon启动容器的过程为例，假设容器内初始化的进程称为init。**Docker daemon在宿主机上负责创建**这个veth pair，把一端绑定到**docker0网桥上**，另一端接入新建的network namespace进程中。这个过程执行期间，**Docker daemon和init就通过pipe进行通信**。具体来说，就是在Docker daemon完成veth pair的创建之前，init在管道的另一端循环等待，直到管道另一端传来Docker daemon关于veth设备的信息，并关闭管道。init才结束等待的过程，并把**它的“eth0”** 启动起来
        + Docker网络示意图![Docker网络示意图](https://github.com/PengJianMin/DockerCore/blob/main/Docker%E7%BD%91%E7%BB%9C%E7%A4%BA%E6%84%8F%E5%9B%BE.jpg)
5. ```   #define _GNU_SOURCE
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
            int child_pid = clone(child_main,child_stack+STACK_SIZE,CLONE_NEWNET|CLONE_NEWNS|CLONE_NEWPID|CLONE_NEWIPC|CLONE_NEWUTS | SIGCHLD,NULL); //加入网络隔离的flag
            waitpid(child_pid,NULL,0);
            printf("Child pid is:%d\n",child_pid);
            printf("Already quit\n");
            return 0;
        }
6. 效果：`ifconfig` 不显示任何网络配置信息
+ user namespaces：主要隔离了安全相关的标识符（identifier）和属性（attribute），包括用户ID、用户组ID、root目录、key（指密钥）以及特殊权限
1. 一个**普通用户的进程**通过clone()创建的新进程在新user namespace中可以拥有**不同的用户和用户组**
2. 一个进程在容器外属于一个**没有特权**的普通用户，但是它创建的容器进程却属于**拥有所有权限**的**超级用户**，这个技术为容器提供了极大的自由
3. 用户在启动Docker daemon的时候指定了**userns-remap**，那么当用户运行容器时，**容器内部的root**用户并不等于宿主机内的root用户，而是**映射到宿主**上的**普通用户**
4. namespace映射图![namespace映射图](https://github.com/PengJianMin/DockerCore/blob/main/namespace%E6%98%A0%E5%B0%84%E5%9B%BE.jpg)
5. 用户**绑定映射**操作:通过在/proc/[pid]/uid_map和/proc/[pid]/gid_map两个文件中写入对应的绑定信息就可以实现这一点
    + 这两个文件只允许由拥有该user namespace中CAP_SETUID权限的**进程**写入**一次**，**不允许修改**（**通过进程写入**）
    + 写入的进程必须是该user namespace的父namespace或者子namespace
    + `0     1005   4294967295`
        + 第一个字段ID-inside-ns表示新建的user namespace中对应的user/group ID
        + 第二个字段ID-outside-ns表示namespace外部映射的user/group ID
        + 最后一个字段表示映射范围，通常填1，表示只映射一个，如果填大于1的值，则按顺序建立一一映射。
+ Docker不仅使用了user namespace，还使用了在user namespace中涉及的**Capabilities机制**。
1. Linux把原来和**超级用户**相关的**高级权限**划分为不同的单元，称为Capability。
2. 管理员可以独立对**特定的Capability**进行使用或禁止
3. Docker同时使用user namespace和Capability，这在很大程度上加强了容器的安全性。
+ 虽然namespace技术使用非常简单，但要真正把容器做到**安全易用**却并非易事
1. PID namespace中，需要实现一个完善的init进程来维护好所有进程
2. network namespace中，还有复杂的路由表和iptables规则没有配置
3. user namespace中还有许多权限问题需要考虑
4. 某些方面Docker已经做得不错，而有些方面才**刚刚起步**
# cgroups资源限制（重点在限制上）
+ cgroups不仅可以限制被namespace隔离起来的资源，还可以为资源设置权重、计算使用量、**操控任务（进程或线程）启停**等
+ cgoups官方定义：cgroups是Linux内核提供的一种机制，这种机制可以根据需求把一系列系统任务及其子任务整合（或分隔）到**按资源划分等级**的不同组内，从而为系统资源管理提供一个统一的框架。
+ cgroups可以限制、记录任务组所使用的**物理资源**（包括CPU、Memory、IO等），为容器实现虚拟化提供了基本保证，是构建Docker等一系列虚拟化管理工具的**基石**
+ 本质上来说，cgroups是内核附加在程序上的**一系列钩子（hook）**，通过程序运行时**对资源的调度**触发相应的钩子以达到**资源追踪和限制**的目的。
+  cgroups**实现方式**及**工作原理**简介
1. Linux中cgroup的实现形式表现为一个**文件系统** ,内核开发者们按照**Linux虚拟文件系统转换器（Virtual Filesystem Switch, VFS）** 接口实现了一套名为cgroup的文件系统，非常巧妙地用来表示cgroups的层级概念，把各个子系统的实现都**封装到文件系统的各项操作中**,把相应信息**写入配置文件**就可以生效
2. cgroups的实现本质上是给任务**挂上钩子**，当任务运行的过程中**涉及某种资源**时，就会触发钩子上所附带的**子系统进行检测**，根据资源类别的不同，使用对应的技术进行**资源限制**和**优先级分配**
3. cgroups如何判断资源超限及超出限额**之后的措施**
    + 对于**不同的系统资源**，cgroups提供了**统一的接口**对资源进行**控制和统计**，但限制的**具体方式则不尽相同**
    + **memory子系统**，会在描述内存状态的“mm_struct”结构体中记录它所属的cgroup，当进程需要申请更多内存时，就会触发cgroup用量检测，用量超过cgroup规定的限额，则拒绝用户的内存申请，否则就给予相应内存并在cgroup的统计信息中记录。实际实现要比以上描述**复杂得多**，不仅需考虑内存的分配与回收，还需考虑不同类型的内存如cache（缓存）和swap（交换区内存拓展）等。
    + 进程所需的内存超过它所属的cgroup最大限额以后，如果设置了**OOM Control（内存超限控制）** ，那么进程就会收到OOM信号并结束；否则进程就会**被挂起**，进入睡眠状态，直到cgroup中其他进程释放了足够的内存资源为止
    + **Docker**中**默认是开启**OOM Control的。
+ cgroups有如下4个特点：
1. cgroups的API以一个**伪文件系统**的方式实现，用户态的程序可以**通过文件操作**实现cgroups的组织管理
2. cgroups的组织管理操作单元可以细粒度到**线程级别**，另外用户可以创建和销毁cgroup，从而实现资源再分配和管理
3. 所有资源管理的功能都以子系统的方式实现，接口统一
4. 子任务创建之初与其**父任务**处于同一个cgroups的控制组
+ cgroups提供了以下四大功能：
1. **资源限制**：cgroups可以对任务使用的资源总额进行限制。如设定应用运行时使用**内存的上限**，一旦超过这个配额就发出**OOM（Out of Memory）提示**
2. **优先级分配**：通过分配的**CPU时间片数量及磁盘IO带宽大小**，实际上就相当于控制了任务运行的优先级
3. **资源统计**：cgroups可以统计系统的资源使用量，如CPU使用时长、内存用量等，这个功能**非常适用于计费**
4. **任务控制**：cgroups可以对任务执行**挂起、恢复**等操作
+ cgroups术语表
1. task（任务）：在cgroups的术语中，任务表示系统的一个**进程或线程**
2. cgroup（控制组）:cgroups中的**资源控制都以cgroup为单位实现**。cgroup表示按某种资源控制标准划分而成的**任务组**，包含一个或多个**子系统**。一个任务可以**加入**某个cgroup，也可以从某个cgroup**迁移**到另外一个cgroup
3. subsystem（子系统）:cgroups中的**子系统就是一个资源调度控制器**。比如CPU子系统可以控制CPU时间分配，内存子系统可以限制cgroup内存使用量
4. hierarchy（层级）：层级由一系列cgroup以一个**树状结构**排列而成，每个层级通过绑定对应的子系统进行资源控制。层级中的cgroup节点可以包含零或多个子节点，子节点**继承父节点挂载的子系统**。整个操作系统可以有多个层级。
+ cgroups组织结构
1. 传统的Unix任务管理，实际上是先启动init任务作为根节点，再由init节点创建子任务作为子节点，而每个子节点又可以创建新的子节点，如此往复，形成一个树状结构。（**单根**）
2. 系统中的多个cgroup构成的层级**并非单根**结构，可以允许存在多个。多个cgroup则是由多个层级构成的**森林**
3. 在Docker中，每个子系统**独自构成**一个层级，这样做非常易于管理。
+ cgroups、任务、子系统、层级四者间的**关系**及其**基本规则**
1. 同一个层级可以**附加**一个或多个子系统。如：CPU和Memory的子系统附加到了一个层级。![同一个层级可以附加一个或多个子系统](https://github.com/PengJianMin/DockerCore/blob/main/%E5%90%8C%E4%B8%80%E4%B8%AA%E5%B1%82%E7%BA%A7%E5%8F%AF%E4%BB%A5%E9%99%84%E5%8A%A0%E4%B8%80%E4%B8%AA%E6%88%96%E5%A4%9A%E4%B8%AA%E5%AD%90%E7%B3%BB%E7%BB%9F.jpg)
2. 一个子系统可以**附加到多个层级**，当且仅当目标层级**只有唯一一个子系统**时。下图小圈中的数字表示子系统附加的时间顺序，CPU子系统附加到层级A的同时不能再附加到层级B，因为层级B已经附加了内存子系统。**如果层级B没有附加过内存子系统**，那么CPU子系统同时附加到两个层级是允许的。![一个已经附加在某个层级上的子系统不能附加到其他含有别的子系统的层级上](https://github.com/PengJianMin/DockerCore/blob/main/%E4%B8%80%E4%B8%AA%E5%B7%B2%E7%BB%8F%E9%99%84%E5%8A%A0%E5%9C%A8%E6%9F%90%E4%B8%AA%E5%B1%82%E7%BA%A7%E4%B8%8A%E7%9A%84%E5%AD%90%E7%B3%BB%E7%BB%9F%E4%B8%8D%E8%83%BD%E9%99%84%E5%8A%A0%E5%88%B0%E5%85%B6%E4%BB%96%E5%90%AB%E6%9C%89%E5%88%AB%E7%9A%84%E5%AD%90%E7%B3%BB%E7%BB%9F%E7%9A%84%E5%B1%82%E7%BA%A7%E4%B8%8A.jpg)
3. 系统每次新建一个层级时，该系统上的所有任务**默认加入**这个新建层级的初始化cgroup，这个cgroup也被称为**root cgroup**。对于创建的每个层级，任务只能存在于其中一个cgroup中，即**一个任务不能存在于同一个层级的不同cgroup中**，但**一个任务可以存在于不同层级中的多个cgroup中**。如果操作时把一个任务添加到**同一个层级**中的另一个cgroup中，则会将它从第一个cgroup中**移除**。下图可以看到，httpd任务已经加入到层级A中的/cg1，而不能加入**同一个层级**中的/cg2，但是可以加入层级B中的/cg3。![一个任务不能属于同一个层级的不同cgroup](https://github.com/PengJianMin/DockerCore/blob/main/%E4%B8%80%E4%B8%AA%E4%BB%BB%E5%8A%A1%E4%B8%8D%E8%83%BD%E5%B1%9E%E4%BA%8E%E5%90%8C%E4%B8%80%E4%B8%AA%E5%B1%82%E7%BA%A7%E7%9A%84%E4%B8%8D%E5%90%8Ccgroup.jpg)
4. 任务在fork/clone自身时创建的子任务**默认与原任务**在同一个cgroup中，但是子任务**允许被移动**到不同的cgroup中。即fork/clone完成后，**父子任务间在cgroup方面是互不影响的**。下图中小圈中的数字表示任务出现的时间顺序，当httpd刚fork出另一个httpd时，两者在同一个层级中的同一个cgroup中。但是随后如果ID为4840的httpd需要移动到其他cgroup也是可以的，因为父子任务间已经独立。总结起来就是：**初始化时子任务与父任务在同一个cgroup，但是这种关系随后可以改变。**![刚fork_clone出的子任务在初始状态与其父任务处于同一个cgroup](https://github.com/PengJianMin/DockerCore/blob/main/%E5%88%9Afork_clone%E5%87%BA%E7%9A%84%E5%AD%90%E4%BB%BB%E5%8A%A1%E5%9C%A8%E5%88%9D%E5%A7%8B%E7%8A%B6%E6%80%81%E4%B8%8E%E5%85%B6%E7%88%B6%E4%BB%BB%E5%8A%A1%E5%A4%84%E4%BA%8E%E5%90%8C%E4%B8%80%E4%B8%AAcgroup.jpg)
+ 子系统简介
1. Docker并**没有对**cgroup本身做增强，**容器用户**一般也**不需要直接操作cgroup**
2. Docker使用如下9种子系统：
    + blkio：可以为**块设备**设定输入/输出限制，比如物理驱动设备（包括磁盘、固态硬盘、USB等）
    + cpu：使用调度程序控制任务**对CPU的使用**
    + cpuacct：自动生成cgroup中任务对CPU资源使用情况的**报告**
    + cpuset：可以为cgroup中的任务分配**独立的**CPU（此处针对多处理器系统）和内存
    + devices：可以开启或关闭cgroup中任务**对设备的访问**
    + freezer：可以挂起或恢复cgroup中的任务
    + memory：可以设定cgroup中任务对**内存**使用量的限定，并且自动生成这些任务对内存资源使用情况的报告
    + perf_event：使用后使cgroup中的任务可以进行统一的**性能测试**
    + net_cls:Docker**没有直接使用**它，它通过使用**等级识别符（classid）** 标记**网络数据包**，从而允许Linux**流量控制程序（Traffic Controller, TC）** 识别从具体cgroup中生成的数据包
3. cgroup简单实验
    + Linux中cgroup的**实现形式**表现为一个**文件系统**，因此需要mount这个文件系统才能够使用（也有可能已经mount好了），挂载成功后，就能看到各类子系统
    + `mount -t cgroup` 查看cgroup挂载情况
    + `ls -l /sys/fs/cgroup` 该目录下能看到各种支持的**子系统**
    + 在/sys/fs/cgroup的cpu子目录下创建控制组
        + `cd /sys/fs/cgroup`
        + `mkdir cg1`
        + `ls cg1`
            + ```   
                    cgroup.clone_children  cpuacct.usage_percpu       cpu.cfs_period_us  cpu.stat
                    cgroup.procs           cpuacct.usage_percpu_sys   cpu.cfs_quota_us   notify_on_release
                    cpuacct.stat           cpuacct.usage_percpu_user  cpu.rt_period_us   tasks
                    cpuacct.usage          cpuacct.usage_sys          cpu.rt_runtime_us
                    cpuacct.usage_all      cpuacct.usage_user         cpu.shares
     + `echo 18828 >> /sys/fs/cgroup/cpu/cg1/tasks` 限制18828进程
     + `echo 20000 > /sys/fs/cgroup/cpu/cg1/cpu.cfs_quota_us` 将CPU限制为最高使用20%
4. 在**Docker的实现**中，**Docker daemon**会在单独**挂载了每一个子系统的控制组目录**（比如/sys/fs/cgroup/cpu）下创建一个**名为docker的控制组**，然后在docker控制组里面，**再为**每个容器创建一个以**容器ID**为名称的**容器控制组**，这个容器里的**所有进程的进程号**都会写到该控制组tasks中，并且在**控制文件（比如cpu.cfs_quota_us）** 中写入预设的限制参数值。
5. docker组的层级结构如下
```
$ tree cgroup/cpu/docker
cgroup/cpu/docker/
├—— 0e8220dbfeac5cb96eb34c4b1a0d648e9358f205bec8c803bc1f7fc178cb8f78 #这是容器ID
|   ├—— cgroup.clone_children
|   ├—— cgroup.procs
|   ├—— cpu.cfs_period_us
|   ├—— cpu.cfs_quota_us #这里有值
|   ├—— cpu.rt_period_us
|   ├—— cpu.rt_runtime_us
|   ├—— cpu.shares
|   ├—— cpu.stat
|   ├—— notify_on_release
|   └—— tasks
├—— cgroup.clone_children
├—— cgroup.procs
├—— cpu.cfs_period_us
├—— cpu.cfs_quota_us
├—— cpu.rt_period_us
├—— cpu.rt_runtime_us
├—— cpu.shares
├—— cpu.stat
├—— notify_on_release
└—— tasks
```
+  `/sys/fs/cgroup/cpu/docker/<container-ID>`下文件的作用`
1. 一个cgroup创建完成，**不管绑定了何种子系统**，其目录下都会生成以下几个文件，用来描述cgroup的相应信息。同样，把相应信息写入这些配置文件就可以生效
    + tasks：这个文件中罗列了所有**在该cgroup中任务的TID，即所有进程或线程的ID**。该文件并不保证任务的TID有序，把**一个任务的TID写到这个文件中就意味着把这个任务加入这个cgroup**中，如果这个任务**所在的任务组**与其不在同一个cgroup，那么会在cgroup.procs文件里记录一个该任务所在任务组的TGID值，但是**该任务组的其他任务**并不受影响。
    + cgroup.procs：这个文件罗列所有在该cgroup中的TGID（线程组ID），即线程组中第一个进程的PID。该文件并不保证TGID有序和无重复。写一个TGID到这个文件就意味着把与其相关的线程都加到这个cgroup中。
    + notify_on_release：填0或1，表示是否在cgroup中**最后一个任务**退出时通知运行release agent，默认情况下是0，表示不运行。
    + release_agent：指定**release agent执行脚本**的文件路径（该文件在**最顶层cgroup目录中存在**），这个脚本通常用于**自动化卸载无用的cgroup**
+ cgroup**与任务之间**的关联关系
1. 实现上，cgroup与任务之间是**多对多的关系**，所以它们并不直接关联，而是通过一个**中间结构**把双向的关联信息记录起来。每个**任务结构体task_struct**中都包含了一个**指针**，可以查询到对应cgroup的情况，同时也可以查询到各个子系统的状态，这些子系统状态中也包含了**找到任务的指针**，不同类型的子系统按需定义**本身的控制信息结构体**，最终在自定义的结构体中把子系统状态指针包含进去，然后**内核**通过container_of（这个宏可以通过一个结构体的成员找到结构体自身）等宏定义来获取对应的结构体，关联到任务，以此达到资源限制的目的。同时，为了让cgroups便于用户理解和使用，也为了用精简的内核代码为cgroup提供熟悉的权限和命名空间管理，内核开发者们按照**Linux虚拟文件系统转换器（Virtual Filesystem Switch, VFS）** 接口实现了一套名为cgroup的文件系统，非常巧妙地用来表示cgroups的层级概念，把各个子系统的实现都**封装到文件系统的各项操作中**。
+ Docker在使用cgroup时的**注意事项**
1. Docker需要通过**挂载cgroup文件系统**新建一个层级结构，挂载时指定要绑定的子系统。把cgroup文件系统挂载上以后，就可以像**操作文件一样对cgroups的层级进行浏览和操作管理**（包括权限管理、子文件管理等）。
2. 除了cgroup文件系统以外，**内核**没有为cgroups的访问和操作添加**任何系统调用**
3. 目前无法将一个新的子系统绑定到**激活的层级**上，或者从一个激活的层级中解除某个子系统的绑定。
4. 当一个**顶层的cgroup文件系统**被卸载（unmount）时，如果其中创建过**深层次的后代cgroup目录**，那么就算上层的cgroup被卸载了，层级也是激活状态，其后代cgroup中的配置依旧有效。只有**递归式地卸载层级**中的所有cgroup，那个层级才会被真正删除。
