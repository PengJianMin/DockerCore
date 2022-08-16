# Docker存储管理 
+ Docker提供了各种基于不同文件系统**实现的存储驱动（aufs、overlay2等）**来管理**实际镜像文件**
# Docker镜像元数据管理
+ Docker镜像在设计上将镜像**元数据**与镜像文件**的存储**完全**隔离**开了
+ Docker在管理镜像层元数据时，采用的也正是从上至下repository、image、layer三个层次
1. repository与image这两类元数据**并无物理上的镜像文件**与之对应
2. layer这种元数据则**存在物理上的镜像层文件**与之对应
3. **repository元数据**
    + 存放位置 **`/var/lib/docker/image/${Driver}/repositories. json`**
```
    /var/lib/docker/image/aufs# cat repositories.json | python -m json.tool
    {    
        "Repositories": {      
            "ubuntu": {          
                "ubuntu:14.04":
                    "sha256:90d5884b1ee07f7f791f51bab92933943c87357bcd2fa6be0e82c48411bbb653"      
                 }    
            }
    }
```
