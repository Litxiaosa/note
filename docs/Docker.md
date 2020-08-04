### **Docker**

####  **1: 安装docker**

```shell
#卸载旧的 docker
 sudo yum remove docker \
                   docker-client \
                   docker-client-latest \
                   docker-common \
                   docker-latest \
                   docker-latest-logrotate \
                   docker-logrotate \
                   docker-engine
                   
 #安装yum-utils软件包（提供yum-config-manager 实用程序）
 1:sudo yum install -y yum-utils
 
 #设置 docker 镜像
 2:sudo yum-config-manager \
     --add-repo \ 
     #https://download.docker.com/linux/centos/docker-ce.repo  #添加docker镜像仓库，我们一般修改为阿里镜像（快）
     http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
     
 #更新索引（可选）
 3：yum makecache fast
  
 #安装 docker 引擎
 4：sudo yum install docker-ce docker-ce-cli containerd.io
 
 #启动 docker
 5：sudo systemctl start docker
```

#### **2: 配置阿里云镜像加速**

```shell
1: sudo mkdir -p /etc/docker

2: sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://a0z24fmr.mirror.aliyuncs.com"]
}
EOF

3: sudo systemctl daemon-reload

4:sudo systemctl restart docker
```

#### **3: 卸载docker**

```shell
#卸载Docker Engine，CLI和Containerd软件包：
 sudo yum remove docker-ce docker-ce-cli containerd.io
 
 #主机上的映像，容器，卷或自定义配置文件不会自动删除。要删除所有图像，容器和卷：
 sudo rm -rf /var/lib/docker
 
 #必须再手动删除所有已编辑的配置文件
```



#### **4:镜像操作**

```shell
 #镜像下载
 docker pull 镜像名
 
 #镜像搜索
 docker search 镜像名
 
 #删除镜像
 docker rmi 镜像ID
 
 #删除全部镜像
 docker rmi $(docker images -q)
```



#### **5: 容器操作**

```shell
#新建并启动容器，参数：-i  以交互模式运行容器；-t  为容器重新分配一个伪输入终端；--name  为容器指定一个名称
 docker run -it --name 容器名
 
 #在容器中打开新的交互模式终端，可以启动新进程，参数：-i  即使没有附加也保持STDIN 打开；-t  分配一个伪终端
 docker exec -it 容器名 /bin/bash
 
 #后台启动容器，参数：-d  已守护方式启动容器  -p 设置映射端口号   -e 修改配置，比如内存大小，账号密码等
 docker run -d -p 宿主机端口号：容器端口号 -e MYSQL_ROOT_PASSWORD=123456 容器名
 
 #启动容器
 docker start 容器名
 
 #重启容器
 docker restart 容器名
 
 #列出容器中运行进程
 docker top 容器名
 
 #查看容器日志，默认参数
 docker logs 容器名
 
 #使用run方式在创建时进入
 docker run -it 容器名 /bin/bash
 
 #关闭容器并退出
 exit
 
 #仅退出容器，不关闭
 快捷键：Ctrl + P + Q
 
 #查看正在运行的容器
 docker ps
 
 #停止所有的容器
 docker stop $(docker ps -a -q)
 
 #删除所有容器
 docker rm $(docker ps -a -q)
 
 ##删除容器
 docker rm 容器名
 
 #查看镜像和容器的详细信息
 docker inspect 容器名字
```



#### **6: 数据卷**

```shell
#关键字  -v  -P：大写 P 表示随机分配端口号 --name:别名
 docker run -it -P --name 自定义容器名字 -v xiaosa:容器的路径 容器名  #如果冒号前面有 表示具名挂载（常用）
 docker run -it -P --name 自定义容器名字 -v 容器的路径 容器名  #如果冒号前面没有 表示匿名挂载
 docker run -it -P --name 自定义容器名字 -v 宿主机路径：容器的路径 容器名  #如果冒号前面是绝对路径 表示自己分配路径
 docker run -it -d -p 3306:3306 -v /etc/mysql/conf.d -v /var/lib/mysql -e -e MYSQL_ROOT_PASSWORD=123456 --name mysql_01 mysql
 
 #volumes-from mysql_01 容器数据卷，可以理解为 以mysql_01为蓝本复制一份
 docker run -it -d -p 3307:3306 -v /etc/mysql/conf.d -v /var/lib/mysql -e -e MYSQL_ROOT_PASSWORD=123456 --name mysql_02 volumes-from mysql_01  mysql
 #创建一个 dockerfile 文件，名字可以所以，建议Dockerfile 
 #文件的内容
 FROM 镜像名
 VOLUME ["挂载的文件名字","可以多个"]  #匿名挂载 
 CMD echo "-------可以打印------"
 CMD /bin/bash
```



#### **7: DockerFile**

```shell
FROM        #基础镜像，一切从这里构建
MAINTAINER  #作者，镜像是谁写的，一般姓名+邮箱 
RUN         #镜像构建的时候，需要运行的命令
ADD         #添加内容，比如：mysql 压缩包，tomcat 压缩包
WORKDIR     #镜像的工作目录
VOLUME      #挂载主机的目录
EXPOSE      #暴露的端口
CMD         #指定这个容器启动的时候要运行的命令.每个dockerfile中只能有一个CMD如果有多个那么只执行最后一个(替换)
ENTRYPOINT  #指定这个容器启动的时候要运行的命令.可以追加命令
ENV         #设置环境内环境变量 例如：ENV MYSQL_ROOT_PASSWORD 123456，  ENV JAVA_HOME /usr/local/jdk1.8.0_45
#1:新建文件
touch my_centos

#2：编写文件
FROM centos
MAINTAINER  xiaosa <416082509@qq.com>

ENV MYPATH /usr/local
WORKDIR $MYPATH

RUN yum -y install vim
RUN yum -y install net-tools

EXPOSE 80

CMD /bin/bash


#3：通过build构建镜像 my_centos：dockerfile文件路径， mycentos:0.1：镜像名字和版本
docker build -f my_centos -t mycentos:0.1 .
```

