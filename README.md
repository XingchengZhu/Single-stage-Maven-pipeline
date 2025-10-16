# Single-stage Maven Pipeline (Rocky 9.6 + Java 8 + Podman)

> 单阶段构建（构建 + 运行在同一镜像）示例。使用 **Podman** 构建并推送镜像，Jenkins 收集测试报告。

## 快速索引

* 基础镜像与构建流程：见仓库根目录

  * **Dockerfile（单阶段）** → [`./Dockerfile`](./Dockerfile)
  * **pom.xml** → [`./pom.xml`](./pom.xml)
* 端口：默认 `8080`
* 运行时 JDK：Java 8（见 `pom.xml` 与 Dockerfile）

---

## 前置条件

* 构建机具备：

  * **Podman**（可执行 `podman login/build/push/run`）
  * **Maven 3.6+**
* 可访问私有镜像仓库：`10.29.230.150:31381`
* （可选）内网 Maven 仓库 / 代理

---

## CI 流程命令（Jenkins Stage）

```bash
# 1) 单元测试（失败不阻断后续阶段）
mvn -B -U -fae -DskipTests=false -Dmaven.test.failure.ignore=true clean test
```

Jenkins 收集测试报告时使用以下通配（在 “JUnit test result” 里）：

```
**/target/surefire-reports/*.xml, **/target/failsafe-reports/*.xml
```

构建并推送镜像（Podman）：

```bash
podman login --tls-verify=false 10.29.230.150:31381 -u admin -p Admin123
podman build --tls-verify=false -t 10.29.230.150:31381/library/testrepo:podman .
podman push  --tls-verify=false 10.29.230.150:31381/library/testrepo:podman
```

> 说明
>
> * `--tls-verify=false` 用于兼容 HTTP 私库或自签名证书场景。
> * 切换为 HTTPS 且证书可信后，可移除此参数。

---

## 本地快速验证

```bash
# 打包（如需跳过测试）
mvn -B -U -DskipTests=true clean package

# 构建镜像并运行
podman build --tls-verify=false -t testrepo:local .
podman run --rm -p 8080:8080 testrepo:local
# 打开 http://localhost:8080
```

---

## 测试报告排查

若 Jenkins 提示 “No test report files were found”，请在同一节点打印路径：

```bash
pwd
find . -maxdepth 4 \( -name 'TEST-*.xml' -o -path '*/surefire-reports/*.txt' -o -name 'failsafe-summary.xml' \) -type f -print
```

确认 `target/surefire-reports/*.xml` 是否存在。

---

## 常见问题

**Q1：推送镜像报 “server gave HTTP response to HTTPS client”**
A：加上 `--tls-verify=false`（login/build/push 都加）。

**Q2：镜像偏大**
A：单阶段镜像包含 JDK 与 Maven 工具链，简单但体积大。可改为多阶段（Builder + Runtime）瘦身：Builder 负责 `mvn package`，Runtime 仅携带 JRE 并复制 `target/*.jar`。

**Q3：端口或 Jar 名变更**
A：端口请在应用配置/`Dockerfile` 中保持一致；Jar 名建议在 `pom.xml` 里通过 `<finalName>` 固定，`ENTRYPOINT` 里相应调整。

---

## 文件说明

* **Dockerfile（单阶段）**：[`./Dockerfile`](./Dockerfile)

  * Rocky 9.6 基础镜像
  * 安装 `java-1.8.0-openjdk-devel` 与 `maven`
  * `dependency:go-offline` 充分利用层缓存
  * 打包后以 `java -jar /app/target/*.jar` 启动

* **pom.xml**：[`./pom.xml`](./pom.xml)

  * Java 8、Spring Boot 2.7.x
  * Surefire 报告开启，`failIfNoTests=false`
  * 可选：设置 `<finalName>` 固定产物名，便于 ENTRYPOINT 精确指定
