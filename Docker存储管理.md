# Docker存储管理 
+ Docker提供了各种基于不同文件系统**实现的存储驱动（aufs、btrfs、devicemapper、vfs、overlay、zfs）**来管理**实际镜像文件**
# Docker镜像元数据管理
+ Docker镜像在设计上将镜像**元数据**与镜像文件**的存储**完全**隔离**开了
+ Docker在管理镜像层元数据时，采用的也正是从上至下repository、image、layer三个层次
1. repository与image这两类元数据**并无物理上的镜像文件**与之对应
2. layer这种元数据则**存在物理上的镜像层文件**与之对应
# `repository`元数据
1. 存放位置 **`/var/lib/docker/image/${Driver}/repositories. json`**
```
    /var/lib/docker/image/aufs# cat repositories.json | python -m json.tool
    {    
        "Repositories": {      
            "ubuntu":{
            "ubuntu:latest":
                    "sha256:ba6acccedd2923aee4c2acc6a23780b14ed4b8a5fa4e14e252a23b846df9b6c1",
            "ubuntu@sha256:626ffe58f6e7566e00254b638eb7e0f3b11d4da9675088f4781a50ae288f3322":
                    "sha256:ba6acccedd2923aee4c2acc6a23780b14ed4b8a5fa4e14e252a23b846df9b6c1"
             }
          }
     }
```
2. 存储了所有`repository`的名字
3. 每个`repository`下 **所有版本** 镜像的名字和tag（如ubuntu:latest）以及对应的镜像ID
4. 当前Docker默认采用SHA256算法根据镜像元数据配置文件**计算出镜像ID**
# image元数据
