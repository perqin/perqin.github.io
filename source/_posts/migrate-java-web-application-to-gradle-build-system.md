---
title: 移植Java Web应用到Gradle构建系统
date: 2017-02-27 18:07:33
tags: [java, gradle]
---

最近我参与了一个Java Web Application项目，但由于历史遗留问题，这个项目并没有使用任何构建系统（哪怕是Ant），而是完全依赖IDE的配置，这给项目带来了很多限制，比如对于习惯使用其他IDE的开发者，比如我，就需要花费不少时间专门配置IDE。不仅如此，缺少构建系统使得整个开发流程也趋于僵化、不灵活，比如独立部署、打包、测试等流程都需要专门的人为执行而不能自动化。基于以上考虑，以及对Gradle的兴趣，我尝试将项目结构改为了Gradle构建系统的Java Web应用。

## Gradle准备

Gradle是一种构建工具。在一个Gradle项目中，可以编写构建脚本`build.gradle`，在其中指定各个开发生命周期中涉及的任务，包括但不仅限于编译、打包成WAR、部署到容器和单元测试。

Gradle的运行需要Java环境，这里不再赘述，JDK环境的配置读者可以直接通过搜索引擎找到。

Gradle的运行方式有两种：Gradle和Gradle Wrapper，其中前者就是直接下载Gradle到本地机器并配置好环境变量，方法也很简单，前往Gradle的官方下载地址[http://services.gradle.org/distributions](http://services.gradle.org/distributions)下载即可。目前（2017/02/27）最新的版本是3.4，其中`gradle-3.4-bin.zip`是可执行包，一般下载这个就可以了，后缀src是源码，后缀all的则是最全的压缩包。

下载解压到自己选择的目录之后，将其中的`bin`目录放入环境变量中即可，比如我的`bin`路径为：

```
/opt/gradle/gradle-3.4/bin
```

验证安装成功：

```
perqin@PQNACRUBT:~$ gradle -v

------------------------------------------------------------
Gradle 3.4
------------------------------------------------------------

Build time:   2017-02-20 14:49:26 UTC
Revision:     73f32d68824582945f5ac1810600e8d87794c3d4

Groovy:       2.4.7
Ant:          Apache Ant(TM) version 1.9.6 compiled on June 29 2015
JVM:          1.8.0_121 (Oracle Corporation 25.121-b13)
OS:           Linux 4.4.0-64-generic amd64
```

然而，有时候我们的项目所需要的可能是其他指定版本的Gradle，甚至有时候让所有开发者都事先安装好指定的Gradle版本十分困难，因此Gradle提供了另一种叫做Gradle Wrapper的小脚本。把`gradle`替换成项目根目录里的`gradlew`（Windows平台下对应的是`gradlew.bat`），该脚本会判断项目需要的Gradle是否已经安装到了本机，如果没有则会自动从官方网站下载对应版本到本地的缓存目录，并将命令传递给缓存的对应版本的Gradle。

获取Wrapper的方式这里不讲，因为一般在Gradle项目中都会有，也可以通过`gradle init -type <type>`初始化Gradle项目环境的时候生成。

## 调整目录结构

因为时间紧迫，我也是现学现用，因此以下的内容也许不是业界最佳实践，但是我认为比原来更好的方案。

在迁移到Gradle之前，项目的目录结构如下：

```
project
+ doc
+ resources
+ src                                   // 源代码
  + cn                                  // cn包，其中包含.java文件，亦包含一些xml文件
  + com
  + ...
  + res                                 // 部分非.java文件
    application-context.xml             // 相关上下文xml文件
    applicationContext-webservice.xml
    applicatoinContext-....xml
    jdbc.properties                     // 相关java配置文件
    ....properties
    ....xml                             // 其他相关xml文件
+ src_jar
+ test                                  // 测试类代码
  + cn
  + com
  + ...
+ WebContent                            // 网站内容根目录
  .gitignore
```

上面的目录虽然比较规整但并不算十分规范，比如Mybatis里的很多mapper的xml文件就被放到了对应的java文件的同一个目录。

首先进入项目根目录，将Gradle环境集成好：

```
$ cd project
$ gradle init -type java-application
```

这时，你会发现目录中多了以下内容：

```
+ gradle
  + wrapper
      gradle-wrapper.jar
      gradle-wrapper.properties
  build.gradle
  gradlew
  gradlew.bat
  settings.gradle
```

其中gradle目录里就是Gradle Wrapper了，而build.gradle是构建脚本，settings.gradle包含构建设置。除此以外，构建过程中还会生成`.gradle`目录和`build`目录，需要把它们加入到`.gitignore`中。

接下来，我对项目的目录进行了调整，新的结构如下：

```
project
+ .gradle                           // 使用Gradle产生的临时目录
+ build                             // 构建过程中生成的临时目录
+ doc
+ gradle
+ libs                              // jar包
+ resources
+ src
  + main
    + java                          // 仅包含java文件，原来的包结构不变
      + cn
      + com
      + ...
    + resources                     // 所有在打包之后需要和java文件放在一起，但非.java的文件
      + META-INF
          MANIFEST.MF
      + cn
        + persistence
          + mapper
              ...Mapper.xml         // 即原来在cn.persistence.mapper包里的xml文件
        application-context.xml     // 即原来的application-context.xml
        ....xml
    + webapp                        // 即原来的WebContent
  + test                            // 测试目录
+ src_jar
  .gitignore
  build.gradle                      // 构建脚本
  gradlew
  gradlew.bat
  settings.gradle                   // 配置脚本
```

需要注意的是，所有和java文件放在一起的非java文件都需要根据原有目录结构放到`src/main/resources/`中，否则无法被Gradle正确打包。

上面的目录结构将java文件和其他文件分开，并且把主代码和测试代码分了源码集（source set），这对大部分的构建工具和IDE都更加友好。后面读者会看到Gradle构建大大减少了项目对IDE的依赖。

## 配置构建脚本

在`build.gradle`中，给出了整个项目的构建任务。经过修改的最终版本如下：

```
buildscript {
    repositories {
        jcenter()
    }

    dependencies {
        // Add Gradle Cargo plugin for deployment
    	classpath 'com.bmuschko:gradle-cargo-plugin:2.2.3'
    }
}

// This is a WAR project!
apply plugin: 'war'

apply plugin: 'com.bmuschko.cargo'

sourceCompatibility = 1.7

// In this section you declare where to find the dependencies of your project
repositories {
    // We use Mavel Central instead of jcenter
    mavenCentral()
}

tasks.withType(JavaCompile) {
	options.warnings = false
}

dependencies {
    // Add jar files in libs/ into dependencies
	compile fileTree(dir: 'libs', include: ['*.jar'])
	// List your dependencies here
	// compile 'group-id:artifact-id:version'
	compile 'javax.activation:activation:1.1'
	// ......
	// Use JUnit test framework
    testCompile 'junit:junit:4.11'
}

cargo {
    // Deploy to Tomcat 7
    containerId = 'tomcat7x'

    deployable {
        context = '/'
    }

    local {
        // JVM args to speed up container performance!
    	jvmArgs = '-XX:PermSize=256M -XX:MaxPermSize=512m -Xms256m -Xmx1024m'
    	// Set environment variable TOMCAT_HOME to the home of your local tomcat
        homeDir = file(System.env.TOMCAT_HOME)
        // Log and output files
        outputFile = file('build/output.txt')
        logFile = file('build/log.txt')
        logLevel = 'high'

        containerProperties {
            property 'cargo.tomcat.ajp.port', 9099
        }
    }
}

// Ensure that the WAR is up-to-date before deployment
cargoRunLocal.dependsOn assemble
```

可以看到，Gradle的依赖管理比添加jar包更加轻松，只需要一行代码，就可以自动从仓库下载对应版本的依赖，这很方便管理项目的依赖，也不需要把一大堆jar包也加入VCS里了。当然，Maven Central中没有的jar包也可以直接放入`libs/`目录。

现在，只要运行：

```
$ gradle assemble
```

或

```
$ ./gradlew assemble
```

，就会自动下载依赖、编译、打包了。接下来，我们通过cargo插件，增加部署到本地Tomcat的任务。参照上面的脚本，在cargo域里面可以配置很多东西，而这些东西在其他开发者刚用IDE打开项目之后，都不需要对着IDE专门配置一遍！

不过必须注意的是，**必须配置环境变量TOMCAT_HOME，指向本地Tomcat的安装路径。**

接下来，运行任务`cargoRunLocal`就可以启动容器：

```
perqin@PQNACRUBT:~/Workspace/project$ ./gradlew cargoRunLocal
Starting a Gradle Daemon (subsequent builds will be faster)
:compileJava UP-TO-DATE
:processResources UP-TO-DATE
:classes UP-TO-DATE
:war UP-TO-DATE
:assemble UP-TO-DATE
:cargoRunLocal
Press Ctrl-C to stop the container...
> Building 83% > :cargoRunLocal
```

可以看到日志也被输出到目录了：

```
perqin@PQNACRUBT:~/Workspace/project/build$ ls -l | grep txt
-rw-rw-r-- 1 perqin perqin     6509 2月  27 22:10 log.txt
-rw-rw-r-- 1 perqin perqin 34139380 2月  27 22:12 output.txt
```

至此，一个项目的Gradle构建就配置好了，即使是命令行，也只需要一两条语句就能完成构建到部署。在IDE中，也能通过简单的双击完成Gradle任务，比如构建、测试等。