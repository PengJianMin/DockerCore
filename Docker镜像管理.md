# Docker镜像管理
+ libcontainer可以快速构建起应用的运行环境。但是当需要进行**容器迁移**，对容器的运行环境进行**全盘打包**时，libcontainer就束手无策了。
+ “镜像”技术作为Docker管理**文件系统**以及**运行环境**的强有力补充
# 什么是Docker镜像
+ Docker镜像是一个**只读**的Docker**容器模板**，含有启动Docker容器所需的**文件系统结构及其内容**
+ Docker镜像是Docker容器的**静态视角**，Docker容器是Docker镜像的**运行状态**
+ Docker镜像的**文件内容**以及一些运行Docker**容器的配置文件**组成了Docker容器的**静态文件系统运行环境——rootfs**
+ rootfs：Docker容器的**根目录**
1. rootfs是Docker容器在启动时**内部进程可见**的文件系统
2. rootfs通常包含一个操作系统运行所需的文件系统，例如可能包含典型的**类Unix操作系统中的目录系统**，**如/dev、/proc、/bin、/etc、/lib、/usr、/tmp及运行Docker容器所需的配置文件、工具等**
3. 在**传统的Linux操作系统内核启动**时，首先挂载一个**只读（read-only）的**rootfs，当系统检测其完整性之后，再将其切换为**读写（read-write）模式**
4. 在Docker架构中，当Docker daemon为Docker容器挂载rootfs时，**沿用了**Linux内核启动时的方法，即将rootfs设为只读模式。在挂载完毕之后，利用**联合挂载（union mount）技术**在已有的只读rootfs上**再挂载一个读写层**。这样，可读写层处于Docker容器文件系统的最顶层，其下**可能联合挂载多个只读层**，只有在Docker容器运行过程中**文件系统发生变化**时，才会把变化的文件内容写到可读写层，并**隐藏只读层中的老版本文件**
+ Docker镜像的主要**结构特点**
1. **分层**（Docker镜像**如此轻量**的重要原因） **镜像层、只读层、读写层**
    + Docker镜像是采用**分层的方式构建**的，每个镜像都由**一系列的“镜像层”** 组成
    + 当需要**修改**容器镜像内的某个文件时，只对处于最上方的**读写层进行变动**，不覆写下层已有文件系统的内容，已有文件**在只读层中的原始版本**仍然存在，但会被读写层中的新版文件所**隐藏**
    + 当使用**docker commit提交**这个**修改过的容器文件系统**为一个**新的镜像**时，保存的内容**仅为最上层**读写文件系统中**被更新过的文件**。
    + 分层达到了在**不同镜像之间共享镜像层**的效果
2. **写时复制（copy-on-write）**  **镜像层、容器层**
    + 在**多个容器之间**共享镜像，每个容器在启动的时候并不需要单独复制一份镜像文件，而是将**所有镜像层**以**只读的方式**挂载到一个挂载点，再在上面**覆盖**一个可读写的**容器层**。
    + 在未更改文件内容时，所有容器**共享同一份数据**
    + 只有在Docker容器运行过程中文件系统**发生变化**时，才会把变化的文件内容写到可读写层，并**隐藏**只读层中的**老版本文件**。
    + 写时复制配合分层机制减少了镜像对**磁盘空间的占用**和**容器启动时间**
3. **内容寻址存储（content-addressable storage）**
    + 根据文件内容来**索引**镜像和镜像层
    + 之前版本对每一个**镜像层**随机生成一个UUID（**已淘汰**），**新模型**对镜像层的**内容**计算校验和，生成一个**内容哈希值**，并以此哈希值**代替之前的UUID**作为**镜像层的唯一标志**
    + 该机制主要提高了**镜像的安全性**，并在**pull、push、load和save操作**后检测数据的完整性
    + 另外，基于内容哈希来索引镜像层，在一定程度上**减少了ID的冲突**并且**增强了镜像层的共享**，对于来自**不同构建**的镜像层，只要**拥有相同的内容哈希**，也能**被不同的镜像共享**
4. **联合挂载、联合文件系统（union filesystem）** 
    + 联合挂载技术可以在**一个挂载点（目录）** 同时挂载 **多个文件系统（分层）**，将挂载点的原目录与被挂载内容进行整合，使得最终可见的文件系统将会包含**整合之后的各层**的文件和目录
    + 联合挂载是用于**将多个镜像层的文件系统**挂载到**一个挂载点来**实现一个**统一文件系统视图**的途径，是下层存储驱动（如aufs、overlay等）实现**分层合并**的方式
    + 所以严格来说，联合挂载并**不是Docker镜像的必需技术**，比如我们在使用Device Mapper存储驱动时，其实是使用了快照技术来达到分层的效果，没有联合挂载这一概念
    + Ubuntu:14.04镜像后容器中的**aufs文件**系统为例
      + 由于初始挂载时**读写层为空**，所以从**用户的角度**看，该**容器的文件系统**与**底层的rootfs**没有差别
      + 从**内核的角度**来看，则是**显式区分**开来的两个层次。当需要修改镜像内的某个文件时，只对处于**最上方的读写层**进行了变动，**不覆写下层**已有文件系统的内容，已有文件在只读层中的**原始版本仍然存在**，但会被读写层中的**新版文件所隐藏**
      + 当**docker commit**这个修改过的**容器文件系统**为一个新的镜像时，保存的内容**仅为最上层读写**文件系统中被**更新过的文件**
      + ![aufs挂载Ubuntu 14.04文件系统示意图](https://github.com/PengJianMin/DockerCore/blob/main/aufs%E6%8C%82%E8%BD%BDUbuntu%2014.04%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%E7%A4%BA%E6%84%8F%E5%9B%BE.jpg)
5. Docker镜像的**存储组织方式**
    + ![ Docker容器文件系统的全局视图](https://github.com/PengJianMin/DockerCore/blob/main/Docker%E5%AE%B9%E5%99%A8%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%E7%9A%84%E5%85%A8%E5%B1%80%E8%A7%86%E5%9B%BE.jpg)
    + 除了echo hello进程所在的cgroups和namespace环境之外，容器文件系统其实是一个**相对独立的组织**
    + **可读写部分（read-write layer以及volumes）、init-layer、只读层（read-only layer）** 这3部分结构共同组成了一个容器所需的下层文件系统，它们通过联合挂载的方式**巧妙地表现为一层**，使得**容器进程**对这些层的存在**一点都不知道**
# Docker镜像关键概念
+  **registry** 
1. registry用以**保存Docker镜像**，其中还包括**镜像层次结构**和关于**镜像的元数据**
2. 用户可以在自己的数据中心搭建**私有的registry**，也可以使用Docker官方的**公用registry服务**，即**Docker Hub**
    + Docker Hub中有两种类型的仓库，即**用户仓库（user repository）** 与**顶层仓库（top-level repository）**
        + 用户仓库由**普通的**Docker Hub用户创建
        + 顶层仓库则由**Docker公司**负责维护，提供官方版本镜像。理论上，顶层仓库中的镜像经过Docker公司验证，被认为是**架构良好且安全的**
+  **repository**
1. repository即由具有某个功能的Docker镜像的**所有迭代版本**构成的**镜像组**
2. **registry由一系列经过命名的repository组成**，repository通过**命名规范**对用户仓库和顶层仓库进行组织
    + 用户仓库的命名由用户名和repository名两部分组成，中间以“/”隔开，即username/repository_name的形式，repository名通常表示镜像所具有的功能，如ansible/ubuntu14.04-ansible；而顶层仓库则**只包含repository名的部分**，如ubuntu。
3. repository是一个**镜像集合**，其中包含了**多个不同版本的镜像**，使用标签进行版本区分，如ubuntu:14.04、ubuntu:12.04等，它们均属于**ubuntu这个repository**
4. **registry是repository的集合，repository是镜像的集合**
+ **manifest（描述文件）**
1. manifest（描述文件）主要存在于registry中作为Docker镜像的**元数据文件**，在pull、push、save和load中作为**镜像结构和基础信息的描述文件**。在镜像被pull或者load到Docker宿主机时，manifest被**转化为本地的镜像配置文件config**
2. **manifest list**可以组合**不同架构**实现**同名Docker镜像的manifest**，用以支持**多架构Docker镜像**
+ **image**
1. Docker内部的image概念是用来存储一组**镜像相关的元数据信息**，主要包括镜像的**架构（如amd64）**、镜像**默认配置信息**、构建镜像的**容器配置信息**、包含所有镜像层信息的rootfs。
2. Docker利用rootfs中的**diff_id**计算出内容寻址的索引（chainID）来获取**layer相关信息**，进而获取每一个镜像层的文件内容。
+ **layer（镜像层）**
1. layer（镜像层）是一个Docker用来**管理镜像层的中间概念**
2. 镜像是由镜像层组成的，而单个镜像层可能被**多个镜像共享**，所以Docker将layer与image的**概念分离**。
3. Docker镜像管理中的layer主要存放了镜像层的diff_id、size、cache-id和parent等内容，**实际的文件内容**则是由**存储驱动**来管理，并可以通过**cache-id**在**本地**索引到。
+ **Dockerfile**
1. Dockerfile是在通过docker build命令**构建自己的Docker镜像**时需要使用到的**定义文件**
2. 它允许用户使用基本的**DSL语法**来定义Docker镜像，每一条指令**描述了构建镜像的步骤**
# Docker镜像构建操作
+ 用户并不能“无中生有”地创建一个镜像，无论是启动一个容器或者构建一个镜像，都是在**其他镜像的基础上**进行的，Docker有一系列镜像称为**基础镜像**（如基础Ubuntu镜像ubuntu、基础Fedora镜像fedora等），**基础镜像便是镜像构建的起点**。
+ docker commit是**将容器**提交为一个镜像，也就是从容器更新或者构建镜像
+ docker build是**在一个镜像的基础上**构建镜像
+ **commit镜像**
1. docker commit命令**只提交**容器镜像**发生变更了的部分**，即修改后的容器镜像与**当前仓库中**对应镜像之间的**差异部分**，这使得该操作**实际需要提交的文件往往并不多**
2. **Docker daemon**接收到对应的HTTP请求后，需要执行的步骤如下:
    + 根据用户输入pause参数的设置确定**是否暂停**该Docker容器的运行
    + 将容器的**可读写层导出打包**，该读写层代表了当前运行容器的文件系统与当初启动该容器的镜像之间的**差异**
    + 在**层存储（layerStore）**中注册可读写层**差异包**
    + 更新镜像历史信息和rootfs，并据此在**镜像存储（imageStore）** 中创建一个**新的镜像**，记录其元数据
    + 如果指定了repository信息，则给上述镜像添加**tag信息**
+ **build构建镜像**
1. 用户主要使用**Dockerfile和docker build命令**来完成一个新镜像的构建。 
    + `Usage: docker build [OPTIONS] PATH | URL | -`
        + PATH或URL所指向的文件称为**context（上下文）**, context包含build Docker镜像过程中**需要的Dockerfile以及其他的资源文件**
2. 当**Docker client**接收到用户命令，首先解析**命令行参数**
    + 情况1：第一个参数为“-”，即
        + ```
            #从STDIN中读入Dockerfile，没有context。
            $ sudo docker build - < Dockerfile
            
            #从STDIN中读入压缩的context。
            $ sudo docker build - < context.tar.gz
          ```
        + 此时，则根据命令行输入参数对Dockerfile和context进行设置。
    + 情况2：第一个参数为URL，且是**git repository URL**，如
        + ```
            $ sudo docker build github.com/creack/docker-firefox
          ```
        + 调用git clone ——depth 1——recursive命令**克隆该GitHub repository**，该操作会在**本地的一个临时目录中**进行，命令成功之后**该目录将作为context**传给Docker daemon，该**目录中的Dockerfile**会被用来进行后续构建Docker镜像
    + 情况3：第一个参数为URL，**且不是git repository URL**，则从该URL**下载context**，并将其封装为一个**io流——io.Reader**，后面的处理与情况1相同，只是将STDIN换为了io.Reader
    + 情况4：其他情况，即context为本地文件或目录的情况
        + ```
            #使用了当前文件夹作为context
            $ sudo docker build -t vieux/apache:2.0 .
            
            #使用/home/me/myapp/dockerfiles/debug作为Dockerfile，并且使用/home/me/myapp作为context
            $ cd /home/me/myapp/some/dir/really/deep
            $ sudo docker build -f /home/me/myapp/dockerfiles/debug /home/me/myapp
          ```
3. Docker server端
    + 创建一个**临时目录**，并将context指定的文件系统**解压到该目录**下
    + 读取并**解析Dockerfile**
    + 根据解析出的Dockerfile**遍历其中**的所有指令，并**分发到不同的模块**去执行。Dockerfile每条指令的格式均为INSTRUCTION arguments,INSTRUCTION是一些特定的关键词，包括FROM、RUN、USER等，都会**映射到不同的parser**进行处理
    + parser为上述每一个指令创建一个对应的**临时容器**，在临时容器中执行当前指令，然后通过**commit使用此容器生成一个镜像层**
    + Dockerfile中**所有的指令对应的层的集合**，就是此次build后的结果
    + 如果指定了**tag参数**，便给镜像打上对应的tag
    + **最后一次commit**生成的镜像ID就会作为**最终的镜像ID**返回
# Docker镜像的分发方法
+ `docker export`与`docker import`：“在某台**机器**上**导出**一个Docker**容器**并且在另外一台机器上**导入**”
+ `docker push`和`docker pull`，或者`docker save`和`docker load`命令进行**镜像的分发**
1. docker push通过**线上Docker Hub**的方式迁移
2. docker save则是通过**线下包**分发的方式迁移
+ 直接**对容器进行**持久化和**使用镜像进行**持久化的区别在于以下两点：
1. 两者**应用的对象**有所不同，docker export用于**持久化容器**，而docker push和docker save用于**持久化镜像**
2. 将容器**导出后再导入（exported-imported）** 后的**容器**会**丢失所有的历史**，而**保存后再加载（saved-loaded）** 的**镜像**则没有丢失历史和层，这意味着后者可以通过docker tag命令实现**历史层回滚**，而前者不行
+ **pull镜像**
1. Docker的server端收到**用户发起的pull请求**后，需要做的主要工作如下：
    + 根据用户命令行参数解析出其希望拉取的repository信息，这里repository可能为**tag格式**，也可能为**digest格式**
    + 将repository信息解析为ReposotryInfo并验证其是否合法
    + 根据待拉取的repository是否为**official版本**以及用户没有配置Docker Mirrors获取**endpoint列表**，并遍历endpoint，**向该endpoint指定的registry发起会话**。endpoint**偏好顺序**为API版本v2＞v1，**协议https＞http**
    + 如果待拉取的repository为official版本，或者endpoint的API版本为v2,Docker便不再尝试对v1 endpoint发起会话，直接向v2 registry拉取镜像。
        + 获取v2 registry的endpoint
        + 由endpoint和待拉取镜像名**创建HTTP会话**、获取拉取指定镜像的认证信息并验证API版本
        + 如果tag值为空，即没有指定标签，则获取v2 registry中repository的tag list，然后对于tag list中的每一个标签，都执行一次pullV2Tag方法。该方法的功能分成两大部分：
            + 一是验证用户请求
            + 二是当且仅当**某一层**不在本地时进行拉取这一层文件**到本地**
        + 如果tag值不为空，则只对**指定标签**的镜像进行上述工作
    + 如果向v2 registry拉取镜像失败，则尝试从v1 registry拉取
2. Docker client端在pull镜像时如果用户没有指定tag，则client会**默认使用latest作为tag**，即Docker server端会收到latest这个tag，所以并不会执行以上描述的过程。但如果用户在client端没有指定tag，而是指定了下载同一个repository**所有tag镜像的flag**，即 **-a**，那么传给server的tag**仍然保持空**，这时候才会执行以上描述的过程。
+ **push镜像**
1. 当用户制作了自己的镜像后，希望将它**上传至仓库**，此时可以通过docker push命令完成该操作
2. Docker server接收到**用户的push请求**后的关键步骤如下:
    + 解析出repository信息
    + 获取所有**非Docker Mirrors**的endpoint列表，并验证repository在**本地是否存在**。遍历endpoint，然后发起同registry的会话。如果确认会话对方API版本是v2，则不再对v1 endpoint发起会话
    + 如果endpoint对应版本为v2 registry，则验证被推registry的访问权限，创建**V2Pusher**，调用**pushV2 Repository方法**。这个方法会判断用户输入的repository名字是否含有tag，如果含有，则在本地repository中获取对应镜像的ID，调用pushV2Tag方法；如果不含有tag，则会在本地repository中查询对应所有同名repository，对其中每一个获取镜像ID，执行**pushV2Tag方法**。这个方法会首先验证用户指定的镜像ID在本地ImageStore中是否存在。接下来，该方法会对从顶向下逐个构建一个描述结构体，上传这些镜像层。将这些镜像内容上传完毕后，再将一份**描述文件manifest上传到**registry。
    + 如果镜像不属于上述情况，则Docker会调用pushRepository方法来推送镜像到v1 registry，并根据待推送的repository和tag信息保证当且仅当某layer在enpoint上不存在时，才上传该layer。
+ **docker export命令导出容器**
1. daemon实例调用ContainerExport方法来进行具体的操作，这个过程的主要步骤如下：
    + 根据命令行参数（容器名称）找到**待导出**的容器
    + 对该容器调用**containerExport()函数**导出容器中的所有数据包括：
        + ❏ **挂载**待导出容器的文件系统；
        + ❏ 打包该容器**basefs（即graphdriver上的挂载点）**下的所有文件。以aufs为例，basefs对应的是aufs/mnt下对应容器ID的目录；
        + ❏ 返回打包文档的结果并卸载该容器的文件系统。
    + 将导出的数据回写到HTTP请求应答中。
+ **docker save命令保存镜像**
1. save函数会创建一个**临时文件夹**用于保存**镜像json文件**。
2. 然后循环遍历所有**待导出的镜像**，对每一个镜像执行saveImage函数来导出该镜像。
3. 另外，为了与老版本repository兼容，还会将被导出的repository的名称、标签及ID信息以JSON格式写入到名为repositories的文件中。
4. 而新版本中被导出的镜像配置文件名、repository的名称、标签以及镜像层描述信息则是写入到**名为manifest.json的文件**中。最后执行文件压缩并写入到输出流。
