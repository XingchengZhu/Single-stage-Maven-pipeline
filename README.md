# 单阶段 Maven 流水线（Podman + Rocky 9.6 + Java 8）使用说明

本仓库示例展示了**不依赖 Docker 守护进程**、仅用 **Podman + 单阶段 Dockerfile** 来构建并运行一个 Spring Boot（Java 8）应用镜像的完整流程，同时兼容 Jenkins 的**测试报告收集**。

---

## 目录

* [环境前置](#环境前置)
* [关键命令一览](#关键命令一览)
* [测试报告收集配置](#测试报告收集配置)
* [镜像构建与推送（Podman）](#镜像构建与推送podman)
* [Dockerfile（单阶段，JDK 1.8）](#dockerfile单阶段jdk-18)
* [pom.xml（Java 8）](#pomxmljava-8)
* [本地运行容器](#本地运行容器)
* [Jenkins Pipeline 参考](#jenkins-pipeline-参考)
* [常见问题排查](#常见问题排查)
* [可选：如何切换为多阶段以瘦身镜像](#可选如何切换为多阶段以瘦身镜像)

---

## 环境前置

* 构建机已安装：

  * **Podman**（支持 `--tls-verify=false` 与 HTTP/不安全私有仓库）
  * **Maven 3.6+**
  * 访问私有镜像仓库：`10.29.230.150:31381`
* 网络策略允许访问你指定的 Maven 中央仓库或公司 Nexus（如需）。

---

## 关键命令一览

### 1）编译并运行测试

（Jenkins 中建议使用，失败继续：`-Dmaven.test.failure.ignore=true`）

```bash
mvn -B -U -fae -DskipTests=false -Dmaven.test.failure.ignore=true clean test
```

### 2）测试报告收集的匹配模式（Jenkins `junit`）

```
**/target/surefire-reports/*.xml, **/target/failsafe-reports/*.xml
```

### 3）登录私有仓库、构建并推送镜像（Podman）

```bash
podman login --tls-verify=false 10.29.230.150:31381 -u admin -p Admin123
podman build --tls-verify=false -t 10.29.230.150:31381/library/testrepo:podman .
podman push  --tls-verify=false 10.29.230.150:31381/library/testrepo:podman
```

---

## 测试报告收集配置

* **Surefire**（单元测试）默认在 `target/surefire-reports/` 生成 `TEST-*.xml`
* **Failsafe**（集成测试，若使用）默认在 `target/failsafe-reports/` 生成 `*.xml`

在 Jenkins 的 `junit` 步骤填写：

```
**/target/surefire-reports/*.xml, **/target/failsafe-reports/*.xml
```

> 如果报告未被发现，请确认：
>
> * 流水线先执行了 `mvn test`
> * 工作目录正确（`pwd` 打印检查）
> * 报告路径是否被误写或被清理

---

## 镜像构建与推送（Podman）

1. 登录私有仓库（HTTP/自签名时使用 `--tls-verify=false`）
2. 直接在源码根目录执行 `podman build`
3. 推送到对应的命名空间/仓库/标签

> **注意**：本示例 Dockerfile 是**单阶段**，镜像较大但流程最简单。如果在意镜像体积，见文末多阶段示例思路。

---

## Dockerfile（单阶段，JDK 1.8）

将以下文件保存为项目根目录的 `Dockerfile`：

```dockerfile
# 单阶段：构建 + 运行都在同一个镜像里（简单但体积较大）
FROM 10.29.230.150:31381/library/m.daocloud.io/docker.io/rockylinux/rockylinux:9.6.20250531

# 基础工具 + OpenJDK 1.8 + Maven
RUN dnf clean all && \
    dnf -y --nobest --allowerasing update && \
    dnf -y --nobest --allowerasing install \
      java-1.8.0-openjdk-devel maven tzdata git which tar gzip && \
    dnf clean all

# 时区
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

WORKDIR /app

# 先拷 POM 利用依赖缓存
COPY pom.xml ./
RUN mvn -B -U -q -DskipTests=true dependency:go-offline

# 再拷源码并打包
COPY src ./src
RUN mvn -B -U -DskipTests=true clean package

EXPOSE 8080

# 直接运行 target 产物；用 sh -c 允许通配符匹配 jar 名
ENTRYPOINT ["sh","-c","exec java -jar /app/target/*.jar"]
```

**要点：**

* 使用你本地私库的 Rocky 9.6 基础镜像
* `dependency:go-offline` 尽量缓存依赖，加速后续构建
* 以 Java 8 编译并运行（与 `pom.xml` 一致）

---

## pom.xml（Java 8）

将以下内容保存为 `pom.xml`（或合并至现有 POM）：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.example</groupId>
  <artifactId>demo-jib</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <packaging>jar</packaging>
  <name>demo-jib</name>
  <description>Demo app built with Maven (Java 8)</description>

  <properties>
    <java.version>1.8</java.version>
    <spring-boot.version>2.7.18</spring-boot.version>
    <maven-compiler-plugin.version>3.10.1</maven-compiler-plugin.version>
    <maven-surefire-plugin.version>3.2.5</maven-surefire-plugin.version>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-dependencies</artifactId>
        <version>${spring-boot.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <!-- Java 8 编译目标 -->
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>${maven-compiler-plugin.version}</version>
        <configuration>
          <source>${java.version}</source>
          <target>${java.version}</target>
          <encoding>${project.build.sourceEncoding}</encoding>
        </configuration>
      </plugin>

      <!-- 单元测试（Jenkins 收集 surefire 报告用） -->
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-surefire-plugin</artifactId>
        <version>${maven-surefire-plugin.version}</version>
        <configuration>
          <failIfNoTests>false</failIfNoTests>
        </configuration>
      </plugin>

      <!-- Spring Boot 打包插件 -->
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
        <version>${spring-boot.version}</version>
        <executions>
          <execution>
            <goals><goal>repackage</goal></goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

---

## 本地运行容器

```bash
podman run --rm -p 8080:8080 10.29.230.150:31381/library/testrepo:podman
# 访问 http://localhost:8080
```

如应用使用了自定义端口，请在 `application.properties` 或 Dockerfile 中同步调整。

---

## Jenkins Pipeline 参考

```groovy
pipeline {
  agent any
  stages {
    stage('Test') {
      steps {
        sh 'mvn -B -U -fae -DskipTests=false -Dmaven.test.failure.ignore=true clean test'
      }
      post {
        always {
          junit '**/target/surefire-reports/*.xml, **/target/failsafe-reports/*.xml'
        }
      }
    }
    stage('Build & Push Image') {
      steps {
        sh '''
          podman login --tls-verify=false 10.29.230.150:31381 -u admin -p Admin123
          podman build --tls-verify=false -t 10.29.230.150:31381/library/testrepo:podman .
          podman push  --tls-verify=false 10.29.230.150:31381/library/testrepo:podman
        '''
      }
    }
  }
}
```

---

## 常见问题排查

* **Jenkins 显示 “No test report files were found”**

  * 确认 `mvn test` 已执行
  * 确认 `junit` 的通配写法：`**/target/surefire-reports/*.xml`
  * 在同一节点/容器内打印 `pwd` 和 `find` 验证路径
* **连接私库报 TLS 错误/HTTP 响应给 HTTPS 客户端**

  * 使用 `--tls-verify=false`（Podman 构建/推送都加）
  * 或配置守护进程/registry 为可信（企业环境按安全规范）
* **Jar 名称不一致导致容器启动失败**

  * 当前 Entrypoint 使用通配符 `/app/target/*.jar`；如需固定，建议在 `pom.xml` 中设置 `<finalName>` 并同步 Dockerfile
* **Java 版本冲突**

  * Dockerfile 使用 **java-1.8.0-openjdk-devel**；`pom.xml` 也使用 `1.8`。两者需一致
* **镜像过大**

  * 单阶段最简单但镜像体积较大，可参考多阶段思路
