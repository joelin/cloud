# rancher 内网安装指南

由于内网无法访问互联网，需要把rancher依赖的信息都预先下载下来。由于rancher使用docker方式进行安装和部署，这里只需要把其依赖的镜像下载下来即可。

## 环境准备
* 安装docker 设置 selinux 如下 :

```
SELINUX=disabled
SELINUXTYPE=targeted
```

* 安装 no SSL registry

> docker run -it -p 5000:5000 --restart=always --name registry  -v  /root/docker_data:/var/lib/registry   registry:2.5.0


* 安装ui

```
docker run \
 -d \
 -e ENV_DOCKER_REGISTRY_HOST=registry \
 -e ENV_DOCKER_REGISTRY_PORT=5000 \
 -p 8888:80 \
 --link registry \
 konradkleine/docker-registry-frontend:v2

```


* 配置安装节点，指向内部的registry  no ssl 配置docker文件如下
使用内部注册中心为默认搜索源，排在docker.io之前,redhat centos带来的好处

```
ADD_REGISTRY='--add-registry 172.17.138.101:5000'

INSECURE_REGISTRY='--insecure-registry 172.17.138.101:5000'
```

## rancher catic集群
rancher 自身的组件,这里使用v1.1.2版本  https://github.com/rancher/rancher/releases/tag/v1.1.2

```
rancher/server:v1.1.2
rancher/agent:v1.0.2
rancher/agent-instance:v0.8.3
rancher-compose-v0.8.6
```

### run server
内置数据库
```
docker run -d --restart=always -p 8080:8080 \
    -e CATTLE_BOOTSTRAP_REQUIRED_IMAGE=172.17.138.101:5000/rancher/agent:v1.0.2 \
    -e CATTLE_AGENT_INSTANCE_IMAGE=172.17.138.101:5000/rancher/agent-instance:v0.8.3 \
    172.17.138.101:5000/rancher/server:v1.1.2

```
外置数据库
```
docker run -d --restart=always -p 8080:8080 \
    -e CATTLE_BOOTSTRAP_REQUIRED_IMAGE=172.17.138.101:5000/rancher/agent:v1.0.2 \
    -e CATTLE_AGENT_INSTANCE_IMAGE=172.17.138.101:5000/rancher/agent-instance:v0.8.3 \
    -e CATTLE_DB_CATTLE_MYSQL_HOST=172.17.138.101 \
    -e CATTLE_DB_CATTLE_MYSQL_PORT=3306 \
    -e CATTLE_DB_CATTLE_MYSQL_NAME=cattle \
    -e CATTLE_DB_CATTLE_USERNAME=cattle \
    -e CATTLE_DB_CATTLE_PASSWORD=cattle \
    rancher/server:v1.1.2
```

### add hosts
根据界面的命令进行添加

## 在cattle里进行k8s集群安装
### 镜像准备
首先需要把k8s所需的镜像拉取下来，因为没有文档，所以自己分析rancher-catalog项目里的源码，得到对应cattle版本的镜像如下，下载下来后，推到内部的注册中心。
```
 busybox
 rancher/etcd:v2.3.6-4
 rancher/ingress-controller:v0.1.3
 rancher/k8s:v1.2.4-rancher9
 rancher/kubectld:v0.2.1
 rancher/kubernetes-agent:v0.2.1

```
### 集群安装
根据界面提示，新建一个k8s集群环境即可。建议至少3个host来搭建k8s集群。

## 总结
* rancher的文档在开源界应该是仅次于k8s，但是基本都是基于互联网环境的文档；无互联网环境下的文档基本不可用，需要自己摸索
* catalog使用git来管理和维护，有很多不方便，在cattle镜像里面，写死了github的地址，有提供env改写这个地址，不过没有进行过测试
* 开始时碰到一些坑，节点没有加成功；界面删掉后，数据库里记录还在，在后续k8s环境搭建时卡在6/9服务部署阶段，解决办法重新搭建环境
* k8s测试时还有一些问题，由于k8s服务启动是还需要从gcr.io里拉取google的镜像，需要手动把这些镜像导入到内部环境，具体有哪些没有统计，后续测试也没有继续进行下去
* 在分析cattle需要哪些镜像来搭建k8s时，发现源码里的一个bug，第二天已被修复，pom.xml里少了agent模块，导致编译不过。 https://github.com/rancher/cattle/blob/v0.168.2/pom.xml
* rancher自身搭建起来非常快，把catalog应用的镜像拉下来后，也能很快的部署实例
