# Docker使用

## Docker基础

#### Docker的运行

##### 启动docker

```
 systemctl start docker.service
```

##### 重启docker

```
systemctl restart docker
```

**守护进程重启**  

```  
sudo systemctl daemon-reload
```

##### 关闭docker 

```
sudo systemctl stop docker
```

#### Docker换源

```
sudo vim /etc/docker/daemon.json
```

输入：
```
{
  "registry-mirrors": [
    "https://docker.hpcloud.cloud",
    "https://docker.m.daocloud.io",
    "https://docker.unsee.tech",
    "https://docker.1panel.live",
    "http://mirrors.ustc.edu.cn",
    "https://docker.chenby.cn",
    "http://mirror.azure.cn",
    "https://dockerpull.org",
    "https://dockerhub.icu",
    "https://hub.rat.dev"
  ]
}
```

json格式要对不然会重启会报错。

重启：

```
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## Docker基本命令

#### `pull` 拉取镜像

```bash
docker pull ubuntu 
```

#### `images` 列出已安装镜像

```
docker images
```

#### `ps` 列出容器

```
docker ps 
```

##### 其他参数

- **`-a, --all`**: 显示所有容器，包括停止的容器。
- **`-q, --quiet`**: 只显示容器 ID。
- **`-l, --latest`**: 显示最近创建的一个容器，包括所有状态。
- **`-n`**: 显示最近创建的 n 个容器，包括所有状态。
- **`--no-trunc`**: 不截断输出。
- **`-s, --size`**: 显示容器的大小。
- **`--filter, -f`**: 根据条件过滤显示的容器。
- **`--format`**: 格式化输出

#### `rmi` 删除镜像

```bash
docker rmi [ID]
```

#### `rm` 删除容器

```
docker rm [ID]
```

#### `run` 运行容器

```bash
docker run --name [name] -it [IMAGE ID] /bin/sh
```

#### `start` 启动容器

```
docker start [ID]
```

#### `restart` 重启容器

```
docker restart [ID]
```

#### `stop`关闭容器

```
docker stop [ID]
```

#### `kill` 强行关闭容器 

```
docker kill [ID]
```

#### `exec` 执行命令

```bash
docker exec [ID] [command] #运行在容器中执行命令
```

```bash
docker exec -it [ID] /bin/bash #启动一个交互式的 shell 会话连接正在运行的 Docker 容器中
```

`-it` : 启动一个交互式的 shell 会话进入正在运行的 Docker 容器中

 #### `rename` 更改容器名称

```bash
docker rename [旧名称] [新名称]
```

#### `logs`查看容器运行日志

```
docker logs [ID]
```

#### `top`查看容器中进程信息

```
docker top [ID]
```

#### cp 给容器传文件

##### 容器给宿主机传

```
docker cp id:<容器内的路径> <本地保存文件的路径>
```

##### 宿主机给容器传

```
docker cp <本地文件的路径> id:<容器内的路径>
```

## Docker进阶

### Docker Compose使用

#### Docker Compose安装

```
apt install docker-compose
```

#### 镜像安装

```
docker-compose pull
```

**注意 ：** 这个操作需要在有 .yaml配置文件的目录执行。

#### 启动容器

```
docker-compose up
```

- 默认情况，`docker-compose up` 启动的容器都在前台，控制台将会同时打印所有容器的输出信息，可以很方便进行调试。
- 使用 `docker-compose up -d`，将会在后台启动并运行所有的容器。一般推荐生产环境下使用该选项。
- 如果服务容器已经存在，`docker-compose up` 将会尝试停止容器，然后重新创建（保持使用 `volumes-from` 挂载的卷），以保证新启动的服务匹配 `docker-compose.yml` 文件的最新内容。

#### 停止容器

```
docker-compose down
```

* 此命令将会停止 `up` 命令所启动的容器，并移除网络，但不会移除数据卷。

#### 列出容器

```
docker-compose ps
```

* -q 只打印容器的 ID 信息。

#### 重新启动

```
docker-compose restart
```

#### 停止容器

```
docker-compose stop
```

#### 删除容器

```
docker-compose rm 
```

推荐先暂停，再删除。