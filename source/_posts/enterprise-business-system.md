---
title: MAVEN 私服（Nexus）
date: 2020-12-23 14:35:00
tags: [Maven,Nexus,一方包,二方包,三方包]
---
## MAVEN 的作用

### 一、项目构建

#### （一）Lifecycle（生命周期）
- clean lifecycle  
![clean lifycycle](/images/maven-nexus/clean-lifecycle.png)
- default lifecycle  
    - validate
    - compile
    - test
    - package：打包
    - verify
    - install：打包并安装到本地仓库
    - deploy：打包并部署到远程仓库  
![default lifycycle](/images/maven-nexus/default-lifecycle.png)
- site lifecycle    
![site lifycycle](/images/maven-nexus/site-lifecycle.jpeg)

#### （二）操作指令
{% codeblock [清理Class文件] %}
mvn clean
{% endcodeblock %}

{% codeblock [正常打包] %}
# 顺序执行 validate compile test package
mvn package
{% endcodeblock %}

{% codeblock [跳过测试打包] %}
# 顺序执行 validate compile package
mvn package -Dmaven.test.skip=true
{% endcodeblock %}

{% codeblock [打包并安装至本地仓库] %}
# 顺序执行 validate compile test package verify install
mvn install
{% endcodeblock %}

{% codeblock [打包并部署至远程仓库] %}
# 顺序执行 validate compile test package verify install deploy
mvn deploy
{% endcodeblock %}

### 二、依赖管理
简而言之，对项目所依赖的jar包进行规范化管理
- 自动识别、下载、引入
{% codeblock [pom.xml] %}
<dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-autoconfigure</artifactId>
            <version>2.3.4.RELEASE</version>
            <scope>compile</scope>
            <classifier>sources</classifier>
            <type>jar.md5</type>
            <optional>true</optional>
        </dependency>
</dependencies>
{% endcodeblock %}

- 版本统一管理（版本冲突解决策略）  
    - jar包依赖策略：传递依赖  
    当A依赖了B、C，如果D依赖了A，则D实际上引入了A、B、C
    - 版本冲突默认策略
        - 1、最短路径优先：计算依赖传递深度，初始为当前项目
        - 2、最先申明优先：当依赖深度相同时，取决于pom.xml jar包引入先后顺序  
    - 指定移除设置  
    {% codeblock [pom.xml] %}
    <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-autoconfigure</artifactId>
                <version>2.3.4.RELEASE</version>
                <scope>compile</scope>
                <classifier>sources</classifier>
                <type>jar.md5</type>
                <optional>true</optional>
                <exclusions>
                        <exclusion>
                            <groupId>histo</groupId>
                            <artifactId>histo</artifactId>
                        </exclusion>
                    </exclusions>
            </dependency>
    </dependencies>
    {% endcodeblock %}

## MAVEN 环境配置

### 一、查看本地仓库
打开 IDEA 工具，查看 Preferences，搜索Maven，查看Local repository 路径。
![find local repository](/images/maven-nexus/find-local-maven-repository.png)
指令 mvn install 打包项目并将安装至本地仓库。
![local repository](/images/maven-nexus/local-maven-repository.png)

### 二、远程库环境配置
打开 IDEA 工具，查看 Preferences，搜索Maven，查看User setting file 路径。
![find setting file](/images/maven-nexus/find-setting-file.png)

#### （一）本地仓库 setting.xml 远程连接配置
{% codeblock [setting.xml] %}
<?xml version="1.0" encoding="UTF-8"?>

<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
  <!-- 远程服务连接配置 -->     
  <servers>
    <server>
      <id>histo-public</id>
      <username>**</username>
      <password>**</password>
    </server>
    <server>
      <id>histo-releases</id>
      <username>**</username>
      <password>**</password>
    </server>
    <server>
      <id>histo-snapshots</id>
      <username>**</username>
      <password>**</password>
    </server>
  </servers>
  <!-- maven 镜像仓库地址配置 -->
  <mirrors>
    <mirror>
     <id>histo-public</id>
     <mirrorOf>*</mirrorOf>
     <url>http://nexus.hengdaomed.com/repository/histo-public/</url>
    </mirror>
  </mirrors>
</settings>
{% endcodeblock %}

- servers 标签配置属性说明
    - server；一组远程服务连接信息
        - id：server连接唯一标识别（用于提供repository/mirror进行server验证信息连接）
        - username：访问用户
        - password：访问密码

- mirrors 标签配置属性说明
    - mirror：一组远程maven镜像仓库地址配置
        - id：mirror唯一身份标识，并在需要访问验证信息时需保持与server中id匹配
        - name：镜像别名
        - url：镜像基础地址
        - mirrorOf：指定镜像连接的具体库，*表示匹配所有
        
[更多资料](https://maven.apache.org/settings.html)

#### （二）项目 pom.xml 远程连接配置

{% codeblock [pom.xml] %}
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <!-- 远程镜像仓库连接配置 -->
    <repositories>
        <repository>
            <id>histo-public</id>
            <name>histo nexus</name>
            <url>http://nexus.hengdaomed.com/repository/histo-public/</url>
        </repository>
    </repositories>
    
    <!-- 远程仓库发布配置 -->
    <distributionManagement>
        <repository>
            <id>histo-releases</id>
            <name>Histo Release Nexus</name>
            <url>http://nexus.hengdaomed.com/repository/histo-releases/</url>
        </repository>
        <snapshotRepository>
            <id>histo-snapshots</id>
            <name>Histo Snapshot Nexus</name>
            <url>http://nexus.hengdaomed.com/repository/histo-snapshots/</url>
        </snapshotRepository>
    </distributionManagement>

    <!-- 打包构建配置，构建source文件 -->
    <build>
        <plugins>
            <plugin>
                <artifactId>maven-source-plugin</artifactId>
                <configuration>
                    <attach>true</attach>
                </configuration>
                <executions>
                    <execution>
                        <phase>compile</phase>
                        <goals>
                            <goal>jar</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>

{% endcodeblock %}

## Nexus 远程仓库服务介绍

### 一、私服的工作原理
![maven 私服工作原理](/images/maven-nexus/local-remote.jpeg)

"包引入分类"以及"Nexus仓库类型"在私服工作流程中的对应关系
### 二、包引入分类
- 一方包：本工程中的各模块的相互依赖
- 二方包：公司内部的依赖库，一般指公司内部的其他项目发布的jar包
- 三方包：公司之外的开源库，比如apache、alibaba、google等发布的依赖
    
![project modules](/images/maven-nexus/project-modules.png)

![项目包引入](/images/maven-nexus/jar-import.png)

### 三、Nexus 仓库类型
- group（组仓库）：使用时连接组仓库
- hosted（宿主仓库）：存放本公司开发jar包
    - Release 版本策略
    - Snapshot 版本策略
    - Mixed 版本策略
- proxy（代理仓库）：代理第三方中央仓库（其他大型开源库）
    
![Nexus Repository](/images/maven-nexus/nexus-repository.png)

![组仓库](/images/maven-nexus/histo-public-repository.png)

