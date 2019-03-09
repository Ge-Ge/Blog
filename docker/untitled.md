# 还为重复安装开发环境而烦吗? 这或许是更好的解决方案 —— docker

#### 工欲善其事必先利其器

开始进行web开发之前，都需要搭建好基本的开发环境. 个人用到的有nginx、redis、mysql、node.js.

**搭建环境不同的方式**

* 使用apt\(ubuntu\)、brew\(mac os\)一个个安装
* 脚本: LNMP一键安装包
* 源码编译

  上面的解决方案都有一个共同的缺点

* 一旦系统重装，需要重新安装、配置（有多台电脑时，开发环境版本容易不一致）
* 没有版本控制系统，软件配置维护麻烦

#### 更好的解决方案 —— docker

基于 [docker](https://www.docker.com/products/docker-desktop)（18.03以上）搭建[nginx](https://hub.docker.com/_/nginx)、 [redis](https://hub.docker.com/_/redis) 、[mysql](https://hub.docker.com/_/mysql) 服务。

#### 项目结构

```text
.
├── .env            # 默认为dev的环境变量
├── .gitignore
├── README.md
├── container       # 不同容器的配置文件
│   ├── mysql
│   │   └── docker-compose.yml
│   ├── nginx
│   │   ├── conf
│   │   ├── docker-compose.prod.yml
│   │   └── docker-compose.yml
│   └── redis
│       └── docker-compose.yml
└── prod           # prod的环境变量
    └── .env
```

docker-compose 在运行时会使用当前目录下的.env文件， 并且不支持指定env文件，所以需要多个不同环境时，只能在对应文件夹下建立.env文件

#### 项目内容

通过.env文件配置整个项目所需要的环境变量

```text
# file .env
# 项目名称
COMPOSE_PROJECT_NAME=site
# compose文件
COMPOSE_FILE=container/nginx/docker-compose.yml:container/mysql/docker-compose.yml:container/redis/docker-compose.yml
# mysql config
MYSQL_ROOT_PASSWORD=123456
MYSQL_DATABASE=demo
# redis config
REDIS_PASSWORD=123456
# 自定义环境变量 本地服务器 IP
SITE_IP=host.docker.internal # host.docker.internal需要18.03以上版本
```

 **以nginx的 docker-compose.yml 文件为例:**  ${SITE\_IP}将被替换成host.docker.internal, $${SITE\_IP}将不会被替换

```text
version: "3"
services:
  nginx:
    image: nginx
    volumes:
      - ./conf/dev.template:/etc/nginx/conf.d/dev.template
    ports:
      - "80:80"
    environment:
      - SITE_IP=${SITE_IP}
    command: /bin/bash -c "envsubst '$${SITE_IP}'< /etc/nginx/conf.d/dev.template > /etc/nginx/conf.d/dev.conf &&  exec nginx -g 'daemon off;'"
    networks:
      - default
      - network_site
networks:
  network_site:
    driver: bridge
```

其他镜像的配置可以从dockerhub查看redis、mysql

#### 启动全部

```text
// dev模式
docker-compose up

// prod模式,使用 prod下的.env文件
cd ./prod && docker-compose up
```

#### 单独启动

```text
docker-compose up nginx
docker-compose up mysql
docker-compose up redis
```

#### 停止

```text
# 停止某个服务
docker-compose stop nginx 
# 停止全部
docker-compose stop
```

具体配置请从[github仓库](https://github.com/Ge-Ge/site)查看 通过使用docker,我们只需要一个repository存放配置, 便可以在多台电脑上迅速安装环境.

