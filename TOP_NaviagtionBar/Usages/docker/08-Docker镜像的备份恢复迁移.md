　　在大家已经学会了如何构建镜像以后，为了备份该镜像，我们有以下几个选择：

- 我们可以将指定镜像保存成 tar 归档文件，需要使用时将 tar 包恢复为镜像即可；
- 登录 DockerHub 注册中心，将镜像推送至 DockerHub 仓库方便使用；
- 搭建私有镜像仓库，将镜像推送至私有镜像仓库方便使用。

　　接下来我们通过 tar 归档文件的方式实现镜像的备份恢复迁移。



## 镜像备份



　　使用 `docker save` 将指定镜像保存成 tar 归档文件。

```shell
docker save [OPTIONS] IMAGE [IMAGE...]
docker save -o /root/mycentos7.tar mycentos:7
```

- `-o`：镜像打包后的归档文件输出的目录。



## 镜像恢复



　　使用 `docker load` 导入 docker save 命令导出的镜像归档文件。

```shell
docker load [OPTIONS]
docker load -i mycentos7.tar
```

- `--input, -i`：指定导入的文件；
- `--quiet, -q`：精简输出信息。



## 镜像迁移



　　镜像迁移同时涉及到了上面两个操作，备份和恢复。

　　我们可以将任何一个 Docker 镜像从一台机器迁移到另一台机器。在迁移过程中，首先我们要把容器构建为 Docker 镜像。然后，该 Docker 镜像被作为 tar 包文件保存到本地。此时只需要拷贝或移动该镜像到我们想要的机器上，恢复该镜像并运行容器即可。

　　当然除了这种方式之外，我们还可以使用镜像仓库实现镜像的备份恢复迁移，接下来我们就学习一下如何使用 DockerHub 的镜像仓库。

