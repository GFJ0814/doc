maven发布jar包到远程maven仓库遇到的问题整理

背景：

​	一个项目分为以下几个module，分别是不同的服务，结构如下：

​	sampling

​		client

​		service

​	sampling pom的内容如下：

```java
<groupId>com.platform.sampling</groupId>
<artifactId>sampling</artifactId>
<version>0.0.4</version>
<modules>
        <module>service</module>
        <module>client</module>
 </modules>
```

service的pom的内容大致如下：

```
<parent>
    <artifactId>sampling</artifactId>
    <groupId>com.platform.sampling</groupId>
    <version>0.0.4</version>
</parent>
<artifactId>service</artifactId>
```

client的pom内容大致如下，client会依赖service中的内容：

```
<parent>
    <artifactId>sampling</artifactId>
    <groupId>com.platform.sampling</groupId>
    <version>0.0.4</version>
</parent>
<groupId>com.platform.sampling</groupId>
<artifactId>client</artifactId>
<version>0.0.4</version>
<packaging>jar</packaging>
 <dependencies>
        <dependency>
            <groupId>com.platform.sampling</groupId>
            <artifactId>service</artifactId>
            <version>0.0.4</version>
        </dependency>
    </dependencies>
```

我们的目的：将client打包发布到中央仓库提供依赖。

一、开发测试阶段：

​	此时，如果是本地开发测试，那么只需要将整个项目install到本地maven库，然后，新建一个maven项目，在项目中添加client的依赖，就可以调试将被发布的client的jar包功能。

二、deploy到远程maven仓库

操作一：在client模块下执行deploy命令

报错1：

[ERROR] Failed to execute goal org.apache.maven.plugins:maven-compiler-plugin:3.1:compile (default-compile) on project service: Fatal error compiling: invalid target release: 1.8 -> [Help 1]
[ERROR] 
[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
[ERROR] Re-run Maven using the -X switch to enable full debug logging.
[ERROR] 
[ERROR] For more information about the errors and possible solutions, please read the following articles:
[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/MojoExecutionException

编译环境没有jdk1.8的环境，解决之后，继续操作一

解决参考：<https://blog.csdn.net/blueheart20/article/details/51580510>

报错2：

​[ERROR] Failed to execute goal org.apache.maven.plugins:maven-deploy-plugin:2.8.2:deploy (default-deploy) on project service: Deployment failed: repository element was not specified in the POM inside distributionManagement element or in -DaltDeploymentRepository=id::layout::url parameter -> [Help 1]
[ERROR] 
[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
[ERROR] Re-run Maven using the -X switch to enable full debug logging.
[ERROR] 
[ERROR] For more information about the errors and possible solutions, please read the following articles:
[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/MojoExecutionException	

意思是，没有在指定要将包打到哪个地方，解决办法要么在pom文件中设置distributionManagement元素的值，要么使用指定发布仓库的命令： -DaltDeploymentRepository=id::layout::url

解决办法：

​	在父pom中添加：

```
<distributionManagement>
    <repository>
        <id>releases</id>
        <url>releases仓库地址</url>
    </repository>
    <snapshotRepository>
        <id>snapshots</id>
        <url>snapshots仓库地址</url>
    </snapshotRepository>
</distributionManagement>
```

继续操作一

还是发布失败！！！！！！！！！

问题原因一：

​	因为client依赖service，如果是打包client到远程的话，首先应该将service打包发布到中央仓库。

OK，那么，在service模块下执行deploy命令，此时，service可以被打包到远程，此时再打包发布client。

因为service依赖parent sampling的pom，但是sampling的pom并没有发布到远程maven库。这就导致在打包client的时候，无法正常的依赖service，导致client的打包失败。

原因总结：在发布子module时，如果依赖parent的pom，那在deploy到仓库时，要将parent pom也发布到仓库，否则会因为找不到parent pom而无法打包。

解决这个问题有两种办法：

1.在parent下执行deploy，这样就会把父pom和子module都发布到maven仓库

2.或者将子module的parent去掉，不依赖不在仓库的pom，将子module提升为一个独立的maven项目发布。

这里选择第一种方式：直接在parent的pom下执行deploy。如果sampling下有其他子模块，如果其他子模块不想打包到远程的话，只需要在对应的子模块的pom中加上：	

```
<properties>
    <maven.deploy.skip>true</maven.deploy.skip>
</properties>
```

​	

​	