---
layout: "post"
title: "使用 docker 部署 spring-boot 项目"
date: "2018-06-14"
categories: docker
---

# 前言

最近有个想法，就是在单机上面部署双实例来实现高可用，当一个实例修改之后重新启动部署的时候，另一个实例是可用的。于是，准备试试 docker。

# Docker 简介

Docker 是一个开源的应用容器引擎，基于 Go 语言 并遵从 Apache2.0 协议开源。

Docker 可以让开发者打包他们的应用以及依赖包到一个轻量级、可移植的容器中，然后发布到任何流行的 Linux 机器上，也可以实现虚拟化。
<!-- more -->

# 简单流程

## 构建一个简单的 spring-boot 应用

使用 idea 快速的搭建一个 spring-boot 应用，添加 web 依赖。

新增一个 controller 如下：

```java
@RestController
public class DockerController {	
    @RequestMapping("/")
    public String index() {
        return "Hello Docker!";
    }
}
```

## 增添 Docker 支持

在`pom.xml`文件中增添如下：

```xml
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <!-- Docker maven plugin -->
            <plugin>
                <groupId>com.spotify</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <version>1.0.0</version>
                <configuration>
                    <imageName>image_name</imageName>
                    <dockerDirectory>src/main/docker</dockerDirectory>
                    <resources>
                        <resource>
                            <targetPath>/</targetPath>
                            <directory>${project.build.directory}</directory>
                            <include>${project.build.finalName}.jar</include>
                        </resource>
                    </resources>
                </configuration>
            </plugin>
            <!-- Docker maven plugin -->
        </plugins>
    </build>
```

在目录`src/main/docker`下创建 Dockerfile 文件，Dockerfile 文件用来说明如何来构建镜像。

```dockerfile
FROM openjdk:8-jdk-alpine
VOLUME /tmp
ADD docker-demo-0.0.1-SNAPSHOT.jar app.jar
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```

这个 Dockerfile 文件很简单，构建 Jdk 基础环境，添加 Jar 到镜像中，简单解释一下：
- FROM ，表示使用 Jdk8 环境 为基础镜像，如果镜像不是本地的会从 DockerHub 进行下载
- VOLUME ，VOLUME 指向了一个/tmp 的目录，由于 Spring Boot 使用内置的 Tomcat 容器，Tomcat 默认使用/tmp 作为工作目录。这个命令的效果是：在宿主机的/var/lib/docker 目录下创建一个临时文件并把它链接到容器中的/tmp 目录
- ADD ，拷贝文件并且重命名
- ENTRYPOINT ，为了缩短 Tomcat 的启动时间，添加 java.security.egd 的系统属性指向/dev/urandom 作为 ENTRYPOINT

这样 Spring Boot 项目添加 Docker 依赖就完成了。

# 安装 Docker 环境

## Linux 下安装 Docker 环境

Linux 环境下安装 Docker 较为简单，一句命令行即可搞定

```bash
yum install docker-io
```

## Windows 下安装 Docker 环境

首先登录官网下载，可能会需要你注册登录一下。

下载下来安装好之后，可能需要重启一下，因为需要开启 hyper-V 支持，这是一种虚拟化支持。

重启好之后会发现 Docker 已经开始运行了，任务栏通知区域有一只小鲸鱼。

需要进行一下设置，在`General`选项卡里面勾选 `Expose daemon on tcp://localhost:2357 without TLS`，如果不勾选的话，等一下打包镜像的时候会打包失败。

然后就是还有一个可能会经常失败的就是下载镜像，处理办法就是更换一下国内的镜像源，在`C:\ProgramData\Docker\config`中新建`daemon.json`文件，若没有这个文件夹，可自己新建，文件内容如下：

```json
{
    "registry-mirrors": ["https://registry.docker-cn.com"],
    "live-restore": true
}
```

然后重启 Docker，打开 powerShell 命令行，输入 `docker version` 可查看相关版本信息。

# 使用 Docker 部署 Spring-boot 项目

对项目进行打包，运行测试，运行正常即可开始用 Docker 构建。

```bash
#打包
mvn package
#启动
java -jar docker-demo-0.0.1-SNAPSHOT.jar
```

看到 Spring-boot 的启动日志后表明环境配置没有问题，接下来我们使用 DockerFile 构建镜像。

```bash
mvn package docker:build
```

构建成功之后，即可使用相关命令启动镜像了：

```bash
 docker run -d -p 8080:8080 image_name
```

启动镜像之后，可以使用`docker ps`来查看运行中的容器。每一次执行`run`命令都会生成一个容器，可以对容器执行`start`、`stop`等操作。

使用浏览器访问`http://localhost:8080`，发现返回：

```txt
Hello Docker!
```

说明使用 Docker 部署 spring-boot 项目成功！

# 附录：常用命令

```bash
#运行镜像 -d 指后台运行，-p 指定映射端口，本机端口映射容器端口
docker run -d -p 8088:8080 [image]

docker images #查看构建好的镜像

docker ps -a #查看所有容器

docker start [container] #启动容器

docker stop [container] #停止容器

docker rm [container] #删除容器

docker rmi [image] #删除镜像
```