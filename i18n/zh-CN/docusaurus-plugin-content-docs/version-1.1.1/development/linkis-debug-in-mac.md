---
title: 在Mac上调试Linkis
sidebar_position: 2
---

> 导语：本文详细记录了如何在IDEA中配置和启动Linkis的各个微服务，并实现JDBC、Python、Shell等脚本的提交和执行。

## 1. 代码调试环境

- Mac OS m1芯片，Linkis的`linkis-cg-engineplugin`和`linkis-cg-engineconnmanager`两个服务暂不支持在Windows上进行调试，可参考官网的开发文档进行远程调试。
- Zulu openjdk 1.8
- maven3.6.3
- Linkis 1.1.x ，本篇文章理论上可支持对Linkis1.0.3和以上版本的本地开发调试

## 2. 准备代码并编译

```shell
git clone git@github.com:apache/incubator-linkis.git
cd incubator-linkis
git checkout dev-1.2.0
```

克隆Linkis的源码到本地，并用IDEA打开，首次打开项目会从maven仓库中下载Linkis项目编译所需的依赖jar包。当依赖jar包加载完毕之后，运行如下编译打包命令。

```shell
mvn -N install
mvn clean Install
```

编译命令运行成功之后，在目录incubator-linkis/linkis-dist/target/下可找到编译好的安装包：apache-linkis-版本号-incubating-bin.tar.gz

## 3. 配置并启动服务

### 3.1 add mysql-connector-java到classpath中

服务启动过程中如果遇到mysql驱动类找不到的情况，可以把mysql-connector-java-版本号.jar添加到对应服务模块的classpath下，详细操作请参考3.5小节。

目前依赖mysql的服务有：

- linkis-mg-gateway
- linkis-ps-publicservice
- linkis-cg-linkismanage

### 3.2 调整log4j2.xml配置

在Linkis源码文件夹下，子目录linkis-dist/package/conf中，是Linkis的一些默认配置文件，首先对log4j2.xml文件进行编辑，在其中增加日志输出到控制台的配置。

![log4j2.xml](/Images/development/debug/log4j.png)

这里只贴出来需要新增的配置内容。

```xml
<configuration status="error" monitorInterval="30">
    <appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <ThresholdFilter level="INFO" onMatch="ACCEPT" onMismatch="DENY"/>
            <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%t] %logger{36} %L %M - %msg%xEx%n"/>
        </Console>
    </appenders>
    <loggers>
        <root level="INFO">
            <appender-ref ref="Console"/>
        </root>
    </loggers>
</configuration>
```

### 3.3 启动eureka服务

Linkis和DSS的服务都依赖Eureka，所以我们需要首先启动Eureka服务，Eureka服务可以在本地启动，也可以使用远程启动的服务。保证各个服务都能访问到Eureka的IP和端口之后，就可以开始着手启动其他微服务了。

在Linkis内部是通过-DserviceName参数设置应用名以及使用配置文件，所以-DserviceName是必须要指定的VM启动参数。

可以通过 “-Xbootclasspath/a:配置文件路径”命令，将配置文件追加到引导程序类的路径末尾，即将依赖的配置文件加到classpath中。

通过勾选Include dependencies with “Provided” scope ，可以在调试时，引入provided级别的依赖包。

![eureka](/Images/development/debug/eureka.png)

参数解释：

```shell
[service name]
linkis-mg-eureka

[Use classpath of module]
linkis-eureka

[Main Class]
org.apache.linkis.eureka.SpringCloudEurekaApplication

[VM Opitons]
-DserviceName=linkis-mg-eureka -Xbootclasspath/a:/Users/leojie/other_project/apache/linkis/incubator-linkis/linkis-dist/package/conf

[Program arguments]
--spring.profiles.active=eureka --eureka.instance.preferIpAddress=true
```

如果不想默认的20303端口可以修改端口配置：

```shell
文件路径：conf/application-eureka.yml
修改端口：
server:
  port: 8080 ##启动的端口
```

上述设置完成之后，直接运行此Application，成功启动后可以通过http://localhost:20303/ 查看eureka服务列表。

![eureka-web](/Images/development/debug/eureka-web.png)

### 3.4 启动linkis-mg-gateway

linkis-mg-gateway是Linkis的服务网关，所有的请求都会经由gateway来转发到对应的服务上。

启动服务器前，首先需要编辑conf/linkis-mg-gateway.properties配置文件，增加管理员用户名和密码，用户名需要与你当前登录的mac用户名保持一致。

```properties
wds.linkis.admin.user=leojie
wds.linkis.admin.password=123456
```

设置 linkis-mg-gateway的启动Application

![gateway-app](/Images/development/debug/gateway.png)

参数解释：

```shell
[Service Name]
linkis-mg-gateway

[Use classpath of module]
linkis-gateway-server-support

[VM Opitons]
-DserviceName=linkis-mg-gateway -Xbootclasspath/a:/Users/leojie/other_project/apache/linkis/incubator-linkis/linkis-dist/package/conf

[main Class]
org.apache.linkis.gateway.springcloud.LinkisGatewayApplication
```

上述设置完成之后，可直接运行此Application。

### 3.5 启动linkis-ps-publicservice

publicservice是Linkis的公共增强服务，为其他微服务模块提供统一配置管理、上下文服务、物料库、数据源管理、微服务管理和历史任务查询等功能的模块。

设置linkis-ps-publicservice的启动Application

![publicservice-app](/Images/development/debug/publicservice.png)

参数解释：

```shell
[Service Name]
linkis-ps-publicservice

[Module Name]
linkis-public-enhancements

[VM Opitons]
-DserviceName=linkis-ps-publicservice -Xbootclasspath/a:/Users/leojie/other_project/apache/linkis/incubator-linkis/linkis-dist/package/conf 

[main Class]
org.apache.linkis.filesystem.LinkisPublicServiceApp

[Add provided scope to classpath]
通过勾选Include dependencies with “Provided” scope ，可以在调试时，引入provided级别的依赖包。
```

直接启动publicservice时，可能会遇到如下报错：

![publicservice-debug-error](/Images/development/debug/publicservice-debug-error.png)

需要把public-module模块加到linkis-public-enhancements模块的classpath下，详细步骤如下：

![step-1](/Images/development/debug/step-1.png)

![step-2](/Images/development/debug/step-2.png)

![step-3](/Images/development/debug/step-3.png)

![step-4](/Images/development/debug/step-4.png)

做完上述配置后重新启动publicservice的Application

### 3.6 启动linkis-ps-cs

启动ps-cs服务之前，需要保证publicservice服务成功启动。

![ps-cs-App](/Images/development/debug/ps-cs-App.png)

参数解释：

```shell
[Service Name]
linkis-ps-cs

[Use classpath of module]
linkis-cs-server

[VM Opitons]
-DserviceName=linkis-ps-cs -Xbootclasspath/a:/Users/leojie/other_project/apache/linkis/incubator-linkis/linkis-dist/package/conf 

[main Class]
org.apache.linkis.cs.server.LinkisCSApplication

[Add provided scope to classpath]
通过勾选Include dependencies with “Provided” scope ，可以在调试时，引入provided级别的依赖包。
```

### 3.7 启动linkis-cg-linkismanager

![cg-linkismanager-APP](/Images/development/debug/cg-linkismanager-APP.png)

参数解释：

```shell
[Service Name]
linkis-cg-linkismanager

[Use classpath of module]
linkis-application-manager

[VM Opitons]
-DserviceName=linkis-cg-linkismanager -Xbootclasspath/a:/Users/leojie/other_project/apache/linkis/incubator-linkis/linkis-dist/package/conf

[main Class]
org.apache.linkis.manager.am.LinkisManagerApplication

[Add provided scope to classpath]
通过勾选Include dependencies with “Provided” scope ，可以在调试时，引入provided级别的依赖包。
```

### 3.8 启动linkis-cg-entrance

![cg-entrance-APP](/Images/development/debug/cg-entrance-APP.png)

参数解释：

```shell
[Service Name]
linkis-cg-entrance

[Use classpath of module]
linkis-entrance

[VM Opitons]
-DserviceName=linkis-cg-entrance -Xbootclasspath/a:D:\yourDir\incubator-linkis\assembly-combined-package\assembly-combined\conf

[main Class]
org.apache.linkis.entrance.LinkisEntranceApplication

[Add provided scope to classpath]
通过勾选Include dependencies with “Provided” scope ，可以在调试时，引入provided级别的依赖包。
```

### 3.9 启动cg-engineconnmanager

![engineconnmanager-app](/Images/development/debug/engineconnmanager-app.png)

参数解释：

```shell
[Service Name]
linkis-cg-engineconnmanager

[Use classpath of module]
linkis-engineconn-manager-server

[VM Opitons]
-DserviceName=linkis-cg-engineconnmanager -Xbootclasspath/a:/Users/leojie/other_project/apache/linkis/incubator-linkis/linkis-dist/package/conf -DJAVA_HOME=/Library/Java/JavaVirtualMachines/zulu-8.jdk/Contents/Home/

[main Class]
org.apache.linkis.ecm.server.LinkisECMApplication

[Add provided scope to classpath]
通过勾选Include dependencies with “Provided” scope ，可以在调试时，引入provided级别的依赖包。
```

-DJAVA_HOME是为了指定ecm启动引擎时所使用的java命令所在的路径

### 3.10 启动linkis-cg-engineplugin

![engineplugin-app](/Images/development/debug/engineplugin-app.png)

参数解释：

```shell
[Service Name]
linkis-cg-engineplugin

[Use classpath of module]
linkis-engineconn-plugin-server

[VM Opitons]
-DserviceName=linkis-cg-engineplugin -Xbootclasspath/a:/Users/leojie/other_project/apache/linkis/incubator-linkis/linkis-dist/package/conf

[main Class]
org.apache.linkis.engineplugin.server.LinkisEngineConnPluginServer

[Add provided scope to classpath]
通过勾选Include dependencies with “Provided” scope ，可以在调试时，引入provided级别的依赖包。
```

启动engineplugin的时候可能会遇到如下报错：

![engineplugin-debug-error](/Images/development/debug/engineplugin-debug-error.png)

解决办法是把public-module模块加到linkis-engineconn-plugin-server模块的classpath下，参考3.5小节

### 3.11 关键配置修改

以上操作只是完成了对Linkis各个微服务启动Application的配置，除此之外，Linkis服务启动时所加载的配置文件中，有些关键配置也需要做针对性地修改，否则启动服务或脚本执行的过程中会遇到一些报错。关键配置的修改归纳如下：

####  3.11.1 conf/linkis.properties

```properties
# linkis底层数据库连接参数配置
wds.linkis.server.mybatis.datasource.url=jdbc:mysql://yourip:3306/linkis?characterEncoding=UTF-8
wds.linkis.server.mybatis.datasource.username=your username
wds.linkis.server.mybatis.datasource.password=your password

# 设置bml物料存储路径不为hdfs
wds.linkis.bml.is.hdfs=false
wds.linkis.bml.local.prefix=/Users/leojie/software/linkis/data/bml

wds.linkis.home=/Users/leojie/software/linkis

# 设置管理员用户名，你的本机用户名
wds.linkis.governance.station.admin=leojie
```

在配置linkis底层数据库连接参数之前，请创建linkis数据库，并运行linkis-dist/package/db/linkis_ddl.sql和linkis-dist/package/db/linkis_dml.sql来初始化所有表和数据。

其中wds.linkis.home=/Users/leojie/software/linkis的目录结构如下，里面只放置了lib目录和conf目录。引擎进程启动时会把wds.linkis.home中的conf和lib路径，加到classpath下，如果wds.linkis.home不指定，可能会遇到目录找不到的异常。

![linkis-home](/Images/development/debug/linkis-home.png)

#### 3.11.2 conf/linkis-cg-entrance.properties

```properties
# entrance服务执行任务的日志目录
wds.linkis.entrance.config.log.path=file:///Users/leojie/software/linkis/data/entranceConfigLog

# 结果集保存目录，本机用户需要读写权限
wds.linkis.resultSet.store.path=file:///Users/leojie/software/linkis/data/resultSetDir
```

#### 3.11.3 conf/linkis-cg-engineconnmanager.properties

```properties
wds.linkis.engineconn.root.dir=/Users/leojie/software/linkis/data/engineconnRootDir
```

不修改可能会遇到路径不存在异常。

#### 3.11.4 conf/linkis-cg-engineplugin.properties

```properties
wds.linkis.engineconn.home=/Users/leojie/other_project/apache/linkis/incubator-linkis/linkis-engineconn-plugins/shell/target/out

wds.linkis.engineconn.plugin.loader.store.path=/Users/leojie/other_project/apache/linkis/incubator-linkis/linkis-engineconn-plugins/shell/target/out
```

这里两个配置主要为了指定引擎存储的根目录，指定为target/out的主要目的是，引擎相关代码或配置改动后可以直接重启engineplugin服务后生效。

### 3.12 为当前用户设置sudo免密

引擎拉起时需要使用sudo来执行启动引擎进程的shell命令，mac上当前用户使用sudo时一般都需要输入密码，因此，需要为当前用户设置sudo免密，设置方法如下：

```shell
sudo chmod u-w /etc/sudoers
sudo visudo
将#%admin ALL=(ALL) AL替换为 %admin ALL=(ALL) NOPASSWD: ALL
保存文件退出
```

### 3.13 服务测试

保证上述服务都是成功启动状态，然后在postman中测试提交运行shell脚本作业。

首先访问登录接口来生成Cookie:

![login](/Images/development/debug/login.png)

然后提交执行shell代码

POST: http://127.0.0.1:9001/api/rest_j/v1/entrance/submit

body参数：

```json
{
  "executionContent": {
    "code": "echo 'hello'",
    "runType": "shell"
  },
  "params": {
    "variable": {
      "testvar": "hello"
    },
    "configuration": {
      "runtime": {},
      "startup": {}
    }
  },
  "source": {
    "scriptPath": "file:///tmp/hadoop/test.sql"
  },
  "labels": {
    "engineType": "shell-1",
    "userCreator": "leojie-IDE"
  }
}
```

执行结果：

```json
{
    "method": "/api/entrance/submit",
    "status": 0,
    "message": "OK",
    "data": {
        "taskID": 1,
        "execID": "exec_id018017linkis-cg-entrance192.168.3.13:9104IDE_leojie_shell_0"
    }
}
```

最后检查任务运行状态和获取运行结果集：

GET http://127.0.0.1:9001/api/rest_j/v1/entrance/exec_id018017linkis-cg-entrance192.168.3.13:9104IDE_leojie_shell_0/progress

```json
{
    "method": "/api/entrance/exec_id018017linkis-cg-entrance192.168.3.13:9104IDE_leojie_shell_0/progress",
    "status": 0,
    "message": "OK",
    "data": {
        "progress": 1,
        "progressInfo": [],
        "execID": "exec_id018017linkis-cg-entrance192.168.3.13:9104IDE_leojie_shell_0"
    }
}
```

GET http://127.0.0.1:9001/api/rest_j/v1/jobhistory/1/get

GET http://127.0.0.1:9001/api/rest_j/v1/filesystem/openFile?path=file:///Users/leojie/software/linkis/data/resultSetDir/leojie/linkis/2022-07-16/214859/IDE/1/1_0.dolphin

```json
{
    "method": "/api/filesystem/openFile",
    "status": 0,
    "message": "OK",
    "data": {
        "metadata": "NULL",
        "totalPage": 0,
        "totalLine": 1,
        "page": 1,
        "type": "1",
        "fileContent": [
            [
                "hello"
            ]
        ]
    }
}
```