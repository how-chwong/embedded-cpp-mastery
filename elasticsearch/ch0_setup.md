# 第0章：极简 Docker 容器环境搭建与 Kibana 工具链（ES篇）

让一个 .NET 程序员在 Windows 上直接下载 ES 的 zip 包并在 CMD 中运行，往往不是一个上策。因为 ES 极为依赖 Java 虚拟机环境（JVM）、复杂的安全证书凭证（8.x 默认强力开启 SSL ），并且会因为缺少图形管理化控制台而感到迷茫。

因此，今天我们采用全球微服务标准的一键拉起大招 —— **Docker Compose**。我们将仅用 15 秒拉起一个包含：**单节点 ElasticSearch 8.11 虚机** + **可视化全能控制台 Kibana** 的完美开发沙盒，并且是**特制的测试无密码调试模式（极其省心！）**。

---

## 1. 黄金利器：什么是 Kibana 及其 DevTools？

作为 .NET 程序员，你在写 SQL 时，一定会使用 Microsoft 的 **SQL Server Management Studio (SSMS)** 开窗口、写脚本、调试语句。
在 ElasticSearch 生态中，提供 100% 同级别地位的可视化大屏控制台就叫 **Kibana**。

Kibana 内部自带一个专门给程序员调试 ES 语句的神器面板： **"Dev Tools" (开发中心工具)**。
* 支持绝佳的**语法自动补全**。
* 支持单侧写 HTTP 语句，右侧秒出 ES 返回的 JSON 报文。
* **我们所有的 CRUD、高级检索、 Mapping 调试百分之百都在此面板中完成！**

---

## 2. 15 秒急速开箱：通过 Docker-Compose 启动开发环境

确保你的电脑上已经安装好 [Docker Desktop](https://www.docker.com/)（如果不清楚如何安装，请参考 .NET 容器化指南：极简下载一个安装即可）。

### 🛠️ 跟着执行：
1. 随意选择一个干净文件夹，例如命名为 `d:\Workspace\es_docker`。
2. 在该文件夹下，新建一个名为 **`docker-compose.yml`** 的物理文件（Docker 的标准配置格式）：

```yaml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.1
    container_name: elasticsearch
    environment:
      - node.name=es-node01
      - discovery.type=single-node   # 单实例开发测试环境（免集群选举）
      - bootstrap.memory_lock=true # 解锁内存限制，避免频繁发生 JVM 内存崩溃
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m" # 限制内存最大最小 512M（研发机福利，防止吃空你的笔记本！）
      - xpack.security.enabled=false # 【高能避坑】：关闭 8.x 繁复的安全账号体系，开发、测试无密码裸连！
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata:/usr/share/elasticsearch/data
    ports:
      - "9200:9200" # ES 数据引擎侦听 HTTP 外部接口

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.1
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200 # 对接 ES 容器域名
      - I18N_LOCALE=zh-CN # 【福利】：Kibana 一键自动转为纯中文汉化界面！
    ports:
      - "5601:5601" # Kibana 控制台页面侦听端口
    depends_on:
      - elasticsearch # 必须等 ES 成功跑起来，才会拉起 Kibana 

volumes:
  esdata:
    driver: local
```

3. 用终端在此文件夹下，一键爆气启动：
   ```bash
   docker-compose up -d
   ```
4. **验证启动结果**：
   - 第一次会自动下载约几百兆的官配镜像。耐心等待一两分钟。
   - 看到控制台输出绿勾 `Started` 后。用浏览器输入：[http://localhost:9200/](http://localhost:9200/)。
   - 如果页面上返回了一段带有 `"cluster_name"` 及一句名言 `"You Know, for Search"` 的 JSON 文本，说明 ES 已安全落地，开始提供高性能服务。

---

## 3. 进入 Kibana 调试大本营

大局已定！
1. 用浏览器访问你的可视化控制台地址：[http://localhost:5601/](http://localhost:5601/)。
2. 页面会自动进入 Kibana 完美汉化的中文主控大屏。
3. **如何找到 DevTools**：
   - 点击左上角的“汉堡菜单”（三横线收缩栏）。
   - 向下滑动，找到并点击 **`开发工具` (Dev Tools)**。
4. 在右侧的大编译框里，清空默认行，输入我们第一句简短测试指令：
   ```http
   GET _cluster/health
   ```
   光标停在这行，点击那行旁边出现的 **“绿色三角形运行按钮”**，右侧极富极高含量的 JSON 返回值便呈现出了当前的健康状况（`status: green`）。

---

## 4. 调试沙盒基础练习（在 Dev Tools 中试玩 CRUD）

ES 天生是 **RESTful API** 的最强代言人。它所有的操作全部是原汁原味的 HTTP 方法映射（对标 C# REST API 设计行为）：
* `PUT` / `POST` -> 创建/更新
* `GET` -> 读取
* `DELETE` -> 物理删除

```http
# 1. 往名为 users 的索引（相当于 C# Users 表）中，创建一号文档（_id = 1）
PUT /users/_doc/1
{
  "name": "Solomon",
  "age": 30,
  "role": "DotNet Developer"
}

# 2. 读取一号文档数据
GET /users/_doc/1

# 3. 完美更新一号文档
POST /users/_update/1
{
  "doc": {
    "age": 31
  }
}

# 4. 彻底物理删除一号文档
DELETE /users/_doc/1
```

你可以在 Kibana 的左侧输入这些语句，点三角形运行，它们无需任何多余的网络组件，就会像你平常在数据库内做增删改查一样，直观、清晰、高速地被执行。

---

现在，你的开发环境、高性能沙盒 Kibana 已完美对接并调试通过。点击进入 **[elasticsearch/ch1_basics.md](elasticsearch/ch1_basics.md)**，我们将了解让 ES 傲视群雄的圣杯底层逻辑 —— **倒排索引机制（Inverted Index）**，以及核心中文分词与数据结构 Mapping 的高级配制！
