---
title: MAVEN 私服（Nexus）与 版本控制
date: 2020-12-23 14:35:00
tags: [Maven,Nexus,一方包,二方包,三方包,git-flow,版本控制]
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
    - <font color=blue>package</font>：打包
    - verify
    - <font color=blue>install</font>：打包并安装到本地仓库
    - <font color=blue>deploy</font>：打包并部署到远程仓库  
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
    - jar包依赖策略：<font color=blue>传递依赖</font>  
    当A依赖了B、C，如果D依赖了A，则D实际上引入了A、B、C
    - 版本冲突默认策略
        - <font color=blue>最短路径优先</font>：计算依赖传递深度，初始为当前项目
        - <font color=blue>最先申明优先</font>：当依赖深度相同时，取决于pom.xml jar包引入先后顺序  
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

## Nexus 私库版本发布规约

### 一、通用规约

- 版本号统一格式：<font color=blue><主版本>.<次版本>.<增量版本>-<代号></font>
- 版本迭代规则：
    - 主版本
        - 大版本可能不兼容小版本
        - 版本号从 1 开始，每次递增 1，且次版本、增量版本重置为 0，例如：由 1.1.3 迭代至 2.0.0
    - 次版本
        - 大版本兼容支持小版本，
        - 版本号从 0 开始，每次递增 1。且增量版本重置为 0，例如：由 1.0.13 迭代至 1.1.0
    - 增量版本
        - 大版本兼容支持小版本
        - 版本号从 0 开始，每次递增 1。例如：由 1.0.0 迭代至 1.0.1
    - 代号
        - SNAPSHOT：不稳定版本，处于开发阶段，代码可能随时变化，例如：1.1.12-SNAPSHOT
        - RCx：稳定版本，处于预发布阶段，代码实现了全部功能，并清除了大部分bug，接近正式发布。x 是数字，表示预发布版本迭代，初始为 1，例如：1.1.12-RC1、1.1.12-RC2
        - RELEASE/去除代号：稳定版本，为正式发布版本。我们采用去除代号方式，例如：用 1.1.12 而不用 1.1.12-RELEASE
- 版本可用性基本原则：
    - 开箱即用
        - pom.xml引入版本
        - application.yml/application.properties 文件加入配置信息
    - 嵌入工程项目
        - pom.xml引入版本
        - 完成工程配置，例如：过滤器配置
        - 开放接口处理

### 二、远程库推送规约
- 业务功能发布（一方包）
    - 推送至与业务相关的远程仓库
- 非业务（工具）功能发布（二方包）
    - 推送至公司公用远程仓库
- [<font color=blue>Nexus 私库与项目连接关系示意图</font>](https://www.processon.com/view/link/5fc44bf4079129329897956f)
![Nexus 私库与项目连接关系示意图](/images/maven-nexus/central-repository-connection.png)  

## git-flow流程与版本控制

⚠️<font color=BurlyWood>图中版本号为错误示例</font>  

![git-flow流程与版本控制](/images/maven-nexus/git-flow.webp)

- 两个长期分支
    - 主分支（master）：对外发布分支，包含稳定的发布版本
    - 开发分支（develop）：基于 master 分支克隆，包含最新开发版本、最近待发布至下一个release的代码
- 三种短期分支
    - 功能分支（feature/xxx）：基于 develop 分支克隆，用于开发某种特定功能，开发完成后，需要并入develop
    - 补丁分支（hotfix/xxx）：基于 master 分支克隆，进行生产缺陷修复，修复完成后，并入 master 与 develop 分支
    - 预发分支（release/xxx）：基于 develop 分支克隆，用于在并入 master 分支，进行正式版发布前，提供预发布（RC）版本进行功能测试。该分支对代码进行了封版，禁止在该分支上进行新功能开发，只允许修复测试出现的bug
- 标签（tag）：预发布以及稳定版本发布需要添加对应版本标识。例如：1.0.0-RC1、1.0.0-RC2 或 1.0.0  

### 一、版本开发、测试、发布标准流程

假定当前封版背景：

- 线上发布版本 1.0.1
- <font color=red>预发布运行版本 1.0.0-RC3</font>
- 长期分支
    - master：项目版本号：1.0.1
    - develop：项目版本号：1.1.0-SNAPSHOT
- 短期分支
    - release/1.0.0
    - hotfix/1.0.1

#### （一）功能开发
- 无未并入 feature 分支
    - 基于 develop Cherry Pick 出 feature/xxx 分支
    - 项目版本号延用 develop：1.1.0-SNAPSHOT
    - 新启 Sprint 功能开发基于 feature/xxx 进行code pr & review
- 存在未并入 feature/xxx1 分支 （1个并行示例，版本号：1.1.0-SNAPSHOT）
    - 基于 develop Cherry Pick 出 feature/xxx2 分支
    - 项目版本号延用 develop：1.1.0-SNAPSHOT
        - 若 feature/xxx2 预估上线时间晚于 feature/xxx1，视为未来发布功能
        - 若 feature/xxx2 预估上线时间早于 feature/xxx1, 视为下版本主要功能
    - 新启 Sprint 功能开发基于 feature/xxx2 进行code pr & review

⚠️<font color=BurlyWood>测试环境（多）持续集成监听对应 feature 分支，用于开发验证</font>  

#### （二）预发布

⚠️<font color=BurlyWood>预发布环境类似线上环境，同时间点仅存在唯一运行版本。即如果存在多个版本并行开发场景，需排队加入预发布（如果条件允许，可考虑合并版本）</font>  

- 基于 develop 创建 release/1.1.0 分支
- 功能开发分支 feature/xxx 代码合并到 release/1.1.0 分支
- <font color=blue>调整 release/1.1.0 分支项目版本号为 1.1.0-RC1</font>
- 添加 标签（tag）：1.1.0-RC1
- 基于 1.1.0-RC1 tag 进行预发布环境发布
- 预发布环境功能测试验收
    - <font color=green>测试验收通过，准备发布上线</font>
    - 测试验收存在bug
        - <font color=blue>调整 release/1.1.0 分支 项目版本号为：1.1.0-RC2</font>
        - 修复内容并入 release/1.1.0 分支
            - 基于 feature/xxx 功能开发分支进行 bug修复
            - 修复完成后并入 release/1.1.0 分支
        - 进行标签添加（版本号递增，例如：1.1.0-RC2）、预发布环境更新、测试验收，直至通过
    - <font color=red>存在功能性问题（功能不可用、功能缺失等），终止并回退预发布，重新进入功能开发阶段</font>  

#### （三）上线发布

- <font color=blue>调整 release/1.1.0 分支项目版本号为：1.1.0</font>
- 将 release/1.1.0 分支代码并入 master 分支
    - 进行标签（tag）添加 1.1.0
    - 基于 master 分支（或者 tag 1.1.0）完成上线发布
- 将 release/1.1.0 分支代码并入 develop 分支
- <font color=blue>调整 develop 分支项目版本号为下个主要功能的版本号：1.2.0-SNAPSHOT</font>  

⚠️<font color=BurlyWood>develop 分支项目版本号总是指向下一个主要功能版本，基于以上示例，思考下个版本号为 1.2.0-SNAPSHOT 的含义？</font>  

### 二、线上Bug紧急修复处理（打补丁）

⚠️<font color=BurlyWood>打补丁：对正式版本（线上运行版本）进行bug修复。并非每个线上bug都需要打补丁，对于不紧急bug，可以随下个功能开发版本修复发布</font> 

假定需要对正式版本 1.1.0 打补丁：  
- 基于 master 分支，新建修复补丁分支 hotfix/1.1.1
- <font color=blue>调整 hotfix/1.1.1 分支项目版本号为 1.1.1-SNAPSHOT</font>
- 基于 hotfix/1.1.1 完成生产缺陷修复
- 修复完成后, <font color=blue>调整 hotfix/1.1.1 分支项目版本号为 1.1.1</font>，并入 master 分支，添加 tag 1.1.1 以及上线发布
- 合并 hotfix/1.1.1 分支代码至 develop（此时必定产生项目版本号冲突——处理策略：采用 develop 项目版本号）

⚠️<font color=BurlyWood>feature 功能分支需保持与 develop 分支项目代码更新同步</font>  