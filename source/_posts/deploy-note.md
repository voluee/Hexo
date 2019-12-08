---
title: Deploy note
date: 2019-2-1 12:00:00
top: false
cover: true
toc: true
summary: 记录WIndows和Linux还有MacOS系统下的开发工具的安装、部署、配置环境、命令等
categories: DevOps
tags:
  - Windows
  - Linux

---

# 部署日记

>  Author：[Voluee](https://github.com/Voluee) 
>
> 记录WIndows和Linux还有MacOS系统下的开发工具的安装、部署、配置环境、命令等
# Windows

## 软件目录

|                       开发工具                       |                          开发工具                           |                           开发工具                           |                           开发工具                           |
| :--------------------------------------------------: | :---------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|       [Typora](https://www.typora.io/#windows)       |     [Notepad++](https://notepad-plus-plus.org/download)     |            [VSC](https://code.visualstudio.com/)             |       [IDEA](https://www.jetbrains.com/idea/download)        |
|                        `Java`                        |         [Erlang](https://www.erlang.org/downloads)          |        [Node.js](https://nodejs.org/zh-cn/download/)         | [Elasticsearch](https://www.elastic.co/cn/downloads/elasticsearch) |
|    [Mysql](https://dev.mysql.com/downloads/mysql)    | [Redis](https://github.com/MicrosoftArchive/redis/releases) | [Mongodb](https://www.mongodb.com/download-center/community) | [Kibana](https://www.elastic.co/cn/downloads/elasticsearch)  |
|                       Navicat                        |           [R-Desktop](https://redisdesktop.com/)            | [Robo3t](https://download.robomongo.org/1.2.1/windows/robo3t-1.2.1-windows-x86_64-3e50a65.zip) |             [Tomcat](https://tomcat.apache.org/)             |
|   [Postman](https://www.getpostman.com/downloads/)   |     [Rabbitmq](https://www.rabbitmq.com/download.html)      |        [Maven](https://maven.apache.org/download.cgi)        |   [Jmeter](https://jmeter.apache.org/download_jmeter.cgi)    |
|         [Git](https://git-scm.com/downloads)         |        [Sourcetree](https://www.sourcetreeapp.com/)         |       [Postman](https://www.getpostman.com/downloads/)       |          [Jekins](https://jenkins.io/zh/download/)           |
| [Wireshark](https://www.wireshark.org/download.html) |                                                             |                                                              |                                                              |

## 常用端口

- RabbitMQ：http://localhost:15672
- Kibana(es)：http://localhost:5601
- MongoDB：http://localhost:27017
- Redis：http://localhost:6379

## CMD命令行代理设置

- 先设置好VPN的本地代理ip和端口

    ```bash
    set http_proxy=http://127.0.0.1:1080
    set https_proxy=http://127.0.0.1:1080
    set http_proxy_user=123
    set http_proxy_pass=123
    ```

- npm设置淘宝镜像

  ```bash
  命令行临时使用指定镜像（淘宝）
  npm --registry https://registry.npm.taobao.org install express
  使用cnpm代替npm
  npm install -g cnpm --registry=https://registry.npm.taobao.org
  命令行永久更改使用指定镜像
  npm config set registry https://registry.npm.taobao.org
  查看目前使用的npm镜像的方法
  npm config get registry
  ```

## Mysql 5.7

- 修改环境变量

  ```yaml
  变量名: MYSQL_HOME
  变量值: C:\developer\env\mysql-5.7.25-winx64 
  path里添加: %MYSQL_HOME%\bin
  ```

- 创建`my.ini`

  ```ini
  [mysqld]
  character-set-server=utf8
  port = 3306
  sql_mode="STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION"
  default_storage_engine=innodb
  innodb_buffer_pool_size=1000M
  innodb_log_file_size=50M
  basedir=C:\developer\env\mysql-5.7.25-winx64
  datadir=C:\developer\env\mysql-5.7.25-winx64\data
  max_connections=200
  [mysql]
  default-character-set=utf8
  [mysql.server]
  default-character-set=utf8
  [mysql_safe]
  default-character-set=utf8
  [client]
  port = 3306
  plugin-dir=C:\developer\env\mysql-5.7.25-winx64\lib\plugin
  ```

- 进入`bin`目录，cmd，执行以下命令

  ```bash
  mysqld --initialize
  在data目录下生成文件，xxx.err文件里说明了root账户的临时密码
  mysqld –install MySQL57
  net start MySQL
  mysql -u root -p
  ALTER USER 'root'@'localhost' IDENTIFIED BY 'qweasd';
  ```

## IDEA

- 下载地址：https://www.jetbrains.com/idea/download
- 安装插件：lombok、jrebel、ali guide、mybatis log plugin

## Redis

- 下载Redis（工具包），下载地址：https://github.com/MicrosoftArchive/redis/releases

- 在目录栏输入cmd后，执行redis的启动命令：redis-server.exe redis.windows.conf

## Elasticsearch

- 下载Elasticsearch6.2.2的zip包（工具包），下载地址：https://www.elastic.co/cn/downloads/past-releases/elasticsearch-6-2-2

- 安装中文分词插件，在elasticsearch-6.2.2\bin目录下执行以下命令：

    ```bash
    elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.2.2/elasticsearch-analysis-ik-6.2.2.zip
    ```

- 运行bin目录下的elasticsearch.bat启动Elasticsearch

- 下载Kibana,作为访问Elasticsearch的客户端，请下载6.2.2版本的zip包（工具包），下载地址：https://artifacts.elastic.co/downloads/kibana/kibana-6.2.2-windows-x86_64.zip

- 运行bin目录下的kibana.bat，启动Kibana的服务端

- 访问http://localhost:5601 打开Kibana的用户界面，选择`Dev Tools`标签，通过输入es的DSL语句来操作es

## Mongodb

- 下载Mongodb安装包，下载地址：https://www.mongodb.com/download-center/community

- 选择安装路径进行安装，打开bin目录，有`mongo.exe(客户端)`和`mongod.exe(服务端)`

- 在安装路径下创建`data\db`和`data\log`两个文件夹

- 在安装路径下创建`mongod.cfg`配置文件

    ```yaml
    systemLog:
        destination: file
        path: c:\developer\env\MongoDB\data\log\mongod.log
    storage: 
        dbPath: c:\developer\env\MongoDB\data\db
    ```

- 安装为服务（运行命令需要用管理员权限）

    ```bash
    C:\developer\env\MongoDB\bin\mongod.exe --config 			  "C:\developer\env\MongoDB\mongod.cfg" --install
    ```

- 服务相关命令

    ```bash
    启动服务：net start MongoDB
    关闭服务：net stop MongoDB
    移除服务：C:\developer\env\MongoDB\bin\mongod.exe --remove
    ```

- 下载robo3t客户端程序（工具包）：https://download.robomongo.org/1.2.1/windows/robo3t-1.2.1-windows-x86_64-3e50a65.zip

- 解压到指定目录，打开robo3t.exe并连接到localhost:27017

## RabbitMQ

- 安装Erlang（工具包），下载地址：http://erlang.org/download/otpwin6421.3.exe

- 安装RabbitMQ（工具包），下载地址：https://dl.bintray.com/rabbitmq/all/rabbitmq-server/3.7.14/rabbitmq-server-3.7.14.exe

- 安装完成后，进入RabbitMQ安装目录下的sbin目录

- 在地址栏输入cmd并回车启动命令行，然后输入以下命令启动管理功能：

    ```bash
    rabbitmq-plugins enable rabbitmq_management
    ```

- 访问地址查看是否安装成功：http://localhost:15672/

- 输入账号密码并登录：guest guest

- 创建帐号并设置其角色为管理员



# Linux

## Docker环境安装

- 安装yum-utils：

    ```bash
    yum install -y yum-utils device-mapper-persistent-data lvm2
    ```

- 为yum源添加docker仓库位置：

    ```bash
    yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
    ```

- 安装docker：

    ```bash
    yum install docker-ce
    ```

- 启动docker：

    ```bash
    systemctl start docker
    ```

## Mysql安装

- 下载mysql5.7的docker镜像：

    ```bash
    docker pull mysql:5.7
    ```

- 使用docker命令启动：

    ```bash
    docker run -p 3306:3306 --name mysql \-v /mydata/mysql/log:/var/log/mysql \-v /mydata/mysql/data:/var/lib/mysql \-v /mydata/mysql/conf:/etc/mysql \-e MYSQL_ROOT_PASSWORD=root  \-d mysql:5.7
    ```

- 参数说明

    ```basic
    -p 3306:3306：将容器的3306端口映射到主机的3306端口
    -v /mydata/mysql/conf:/etc/mysql：将配置文件夹挂在到主机
    -v /mydata/mysql/log:/var/log/mysql：将日志文件夹挂载到主机
    -v /mydata/mysql/data:/var/lib/mysql/：将数据文件夹挂载到主机
    -e MYSQLROOTPASSWORD=root：初始化root用户的密码
    ```

- 进入运行mysql的docker容器：

    ```bash
    docker exec -it mysql /bin/bash
    ```

- 使用mysql命令打开客户端：

    ```bash
    mysql -uroot -proot --default-character-set=utf8
    ```

- 创建mall数据库：

    ```bash
    create database mall character set utf8
    ```

- 安装上传下载插件，并将docment/sql/mall.sql上传到Linux服务器上：

    ```bash
    yum -y install lrzsz
    ```

- 将mall.sql文件拷贝到mysql容器的/目录下：

    ```bash
    docker cp /mydata/mall.sql mysql:/
    ```

- 将sql文件导入到数据库：

    ```bash
    use mall;source /mall.sql;
    ```

- 创建一个reader帐号并修改权限，使得任何ip都能访问：

    ```bash
    grant all privileges on *.* to 'reader' @'%' identified by '123456';
    ```

## Redis安装

- 下载redis3.2的docker镜像：

    ```bash
    docker pull redis:3.2
    ```

- 使用docker命令启动：

    ```bash
    docker run -p 6379:6379 --name redis \-v /mydata/redis/data:/data \-d redis:3.2 redis-server --appendonly yes
    ```

- 进入redis容器使用redis-cli命令进行连接：

    ```bash
    docker exec -it redis redis-cli
    ```

## Nginx安装

### 下载nginx1.10的docker镜像：

```bash
docker pull nginx:1.10
```

### 从容器中拷贝nginx配置

- 先运行一次容器（为了拷贝配置文件）：

    ```bash
    docker run -p 80:80 --name nginx \-v /mydata/nginx/html:/usr/share/nginx/html \-v /mydata/nginx/logs:/var/log/nginx  \-d nginx:1.17.1
    ```

- 将容器内的配置文件拷贝到指定目录：

    ```bash
    docker container cp nginx:/etc/nginx /mydata/nginx/
    ```

- 修改文件名称：

    ```bash
    mv nginx conf
    ```

- 终止并删除容器：

    ```bash
    docker stop nginx
    docker rm nginx
    ```

### 使用docker命令启动：


```bash
docker run -p 80:80 --name nginx \-v /mydata/nginx/html:/usr/share/nginx/html \-v /mydata/nginx/logs:/var/log/nginx  \-v /mydata/nginx/conf:/etc/nginx \-d nginx:1.17.1
```


## RabbitMQ安装

- 下载rabbitmq3.7.15的docker镜像：

    ```bash
    docker pull rabbitmq:3.7.15
    ```

- 使用docker命令启动：

    ```bash
    docker run -d --name rabbitmq \--publish 5671:5671 --publish 5672:5672 --publish 4369:4369 \--publish 25672:25672 --publish 15671:15671 --publish 15672:15672 \rabbitmq:3.7.15
    ```

- 进入容器并开启管理功能：

    ```bash
    docker exec -it rabbitmq /bin/bash
    rabbitmq-plugins enable rabbitmq_management
    ```

- 开启防火墙：

    ```bash
    systemctl start firewalld
    systemctl stop firewalld
    firewall-cmd --zone=public --add-port=15672/tcp --permanent
    firewall-cmd --reload
    ```

## Elasticsearch安装

- 下载elasticsearch6.4.0的docker镜像：

    ```bash
    docker pull elasticsearch:6.4.0
    ```

- 修改虚拟内存区域大小，否则会因为过小而无法启动:

    ```bash
    sysctl -w vm.max_map_count=262144
    ```

- 使用docker命令启动：

    ```bash
    docker run -p 9200:9200 -p 9300:9300 --name elasticsearch \-e "discovery.type=single-node" \-e "cluster.name=elasticsearch" \-v /mydata/elasticsearch/plugins:/usr/share/elasticsearch/plugins \-v /mydata/elasticsearch/data:/usr/share/elasticsearch/data \-d elasticsearch:6.4.0
    ```

- 启动时会发现/usr/share/elasticsearch/data目录没有访问权限，只需要修改/mydata/elasticsearch/data目录的权限，再重新启动。

    ```bash
    chmod 777 /mydata/elasticsearch/data/
    ```

- 安装中文分词器IKAnalyzer，并重新启动：

    ```bash
    docker exec -it elasticsearch /bin/bash#此命令需要在容器中运行
    elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.4.0/elasticsearch-analysis-ik-6.4.0.zip 
    docker restart elasticsearch
    ```

- 开启防火墙：

    ```bash
    firewall-cmd --zone=public --add-port=9200/tcp --permanent
    firewall-cmd --reload
    ```

- 访问会返回版本信息：http://192.168.3.101:9200/ 

## kibana安装

- 下载kibana6.4.0的docker镜像：

    ```bash
    docker pull kibana:6.4.0
    ```

- 使用docker命令启动：

    ```bash
    docker run --name kibana -p 5601:5601 \--link elasticsearch:es \-e "elasticsearch.hosts=http://es:9200" \-d kibana:6.4.0
    ```

- 开启防火墙：

    ```bash
    firewall-cmd --zone=public --add-port=5601/tcp --permanentfirewall-cmd --reload
    ```

- 访问地址进行测试：http://192.168.3.101:5601 

## Mongodb安装

- 下载mongo3.2的docker镜像：

    ```bash
    docker pull mongo:3.2
    ```

- 使用docker命令启动：

    ```bash
    docker run -p 27017:27017 --name mongo \-v /mydata/mongo/db:/data/db \-d mongo:3.2
    ```


