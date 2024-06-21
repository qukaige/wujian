### 部署环境
- 操作系统：CentOS 7.6
- 数据库：Postgres12.x
- Docker：20.x
- docker-mendix-buildpack：v5.0.3

### 部署步骤
#### 1. 安装 Docker
   参考官网安装
#### 2. 构建Mendix应用镜像
```bash
# 我这里测试5.0.3版本
git clone --branch v5.0.3 https://github.com/mendix/docker-mendix-buildpack.git
cd docker-mendix-buildpack
# 构建2个镜像, 镜像会在外网下载资源, 国内网络可能会失败
docker build -t mendix-rootfs:app -f rootfs-app.dockerfile .
docker build -t mendix-rootfs:builder -f rootfs-builder.dockerfile .
# 执行完可以看到构建完成的镜像
[root@VM-4-8-centos docker-mendix-buildpack]# docker images
REPOSITORY                                                 TAG       IMAGE ID       CREATED         SIZE                                             
mendix-rootfs                                              builder   d4298b29fce9   9 seconds ago   1.08GB
mendix-rootfs                                              app       95c4f942bc62   3 minutes ago   470MB

# 如果不行构建也可以通过网络下载镜像
docker pull wallaceferreirasetta/mendix-rootfs:app
docker pull wallaceferreirasetta/mendix-rootfs:builder

# 修改Dockerfile文件, 将这两个参数改成你自己构建的镜像, 保存退出
ARG ROOTFS_IMAGE=wallaceferreirasetta/mendix-rootfs:app
ARG BUILDER_ROOTFS_IMAGE=wallaceferreirasetta/mendix-rootfs:builder

# 新建文件夹
mkdir build
# 将应用mpk包复制到build目录, 通过unzip解压
# 注意 如果是10.x新版需要手动把 vendorlib 中的jar 复制到userlib中, 因为打包时不会包含vendorlib中的文件
ll build
total 30132
-rw-r--r--  1 root root 19470571 Jun 14 11:27 EchartsDemo.mpk
-rw-r--r--  1 root root 11304960 Jun 14 11:27 EchartsDemo.mpr
drwxr-xr-x  5 root root     4096 Jun 11 16:37 javascriptsource
drwxr-xr-x  8 root root     4096 Jun 11 16:37 javasource
-rw-r--r--  1 root root    50836 Jun 14 11:27 package.xml
drwxr-xr-x  4 root root     4096 Jun 11 16:38 theme
drwxr-xr-x  3 root root     4096 Jun 11 16:38 theme-cache
drwxr-xr-x 10 root root     4096 Jun 11 16:37 themesource
drwxr-xr-x  2 root root     4096 Jun 14 11:30 widgets
# 回到 docker-mendix-buildpack 目录,构建应用镜像
docker build \
--build-arg BUILD_PATH=build \
--tag mendix/echartsdemo:v1 .
# 至此已完成镜像构建
```
#### 3. 安装 Postgres
```bash
# 这里使用docker快速启动
# 创建一个网络
docker network create mendix
# 挂载数据目录
mkdir -p /data/postgres/data
# -e POSTGRES_PASSWORD 设置密码
docker run --name postgres12.5 --network=mendix \
-e POSTGRES_PASSWORD=bWBdG10XzFS4JPR \
-e PGDATA=/var/lib/postgresql/data/pgdata \
-v /data/postgres/data:/var/lib/postgresql/data \
-p 5432:5432 -dit postgres:12.5
```
#### 4. 启动Mendix应用
```bash
docker run -dit --name=echartsdemo -p 8080:8080 --network=mendix \
-e ADMIN_PASSWORD=Password1! \
-e DATABASE_URL=postgres://postgres:bWBdG10XzFS4JPR@1.1.1.1:5432/postgres \
-v /root/mendix/data/files:/opt/mendix/build/data/files \
-e LICENSE_ID=license配置, 没有可以不写 \
-e LICENSE_KEY=license配置, 没有可以不写 \
mendix/echartsdemo:v1

# /root/mendix/data/files 是mendix默认文件存储路径, 根据需要是否要进行外部挂载
```