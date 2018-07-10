## 说明

```
此文主要介绍maven的安装使用已经常用的命令
```

## 安装

java的安装就不在这里说了。

```
下载地址：http://maven.apache.org/download.cgi
```

## 配置环境变量

```
这里编辑用户目录的.profile文件，编辑完成后执行 source .profile
PATH="$HOME/bin:$HOME/.local/bin:$PATH"
MAVEN_HOME=/opt/apache-maven-3.2.5
export PATH=${PATH}:${MAVEN_HOME}/bin
```

## 测试

```
mvn -v
Apache Maven 3.2.5 (12a6b3acb947671f09b81f49094c53f426d8cea1; 2014-12-15T01:29:23+08:00)
Maven home: /opt/apache-maven-3.2.5
Java version: 1.8.0_101, vendor: Oracle Corporation
Java home: /opt/jdk1.8.0_101/jre
Default locale: en_US, platform encoding: ANSI_X3.4-1968
OS name: "linux", version: "4.4.0-109-generic", arch: "amd64", family: "unix"
安装成功
```

## 命令

### 生成eclipse文件

```
mvn eclipse:eclipse
```

## 清除target下面的文件

```
mvn clean
```

## 编译项目

```
mvn compile
```

## 打包

```
mvn package
```

## 打包跳过测试代码

```
mvn clean install -Dmaven.test.skip=true
```

## 拷贝依赖jar包到指定目录

```
mvn dependency:copy-dependencies -DoutputDirectory=WebContent/WEB-INF/lib  -DincludeScope=runtime
```

## 安装jar包

```
mvn install:install-file -DgroupId=<groupId> -DartifactId=<artifactId> -Dversion=1.0.0 -Dpackaging=jar -Dfile=jar包地址
```

