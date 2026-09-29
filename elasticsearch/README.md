# ElasticSearch 极速升级通关指南 (面向 .NET/C# 架构师)

欢迎来到 ElasticSearch (简称 ES) 的世界！

作为一名 .NET 工程师，你极度熟悉 SQL Server、MySQL 或者 PostgreSQL 等传统关系型数据库（RDBMS）。但是，当你的项目面临：
1. **千万级、亿级海量文本数据**的面面俱到、毫秒级模糊检索？
2. 类似于淘宝、京东商品名搜索那样的 **自动高亮（Highlighting）、纠错、拼音联想与分词匹配**？
3. 超大型分布式实时监控、服务器系统日志流聚合分析（ELK 架构）？

传统 SQL 的像 `charindex`、`LIKE "%keyword%"` 查询不仅会让数据库的 B+ 树索引彻底失效（全表扫描），更会导致服务器瞬时 CPU 飙升 100% 从而引发生产事故。 

这时候，**ElasticSearch — 基于倒排索引（Inverted Index）的高性能分布式全文搜索引擎** 便是终极救星。

---

## 🔍 核心心智模型对比：SQL 关系型 vs ES 全文检索

为了让你瞬间拉齐 ES 的术语和概念，我们拿你最熟悉的 **关系型数据库 (SQL)** 来做镜像对比：

| 维度 / 关系数据库 (RDBMS) | ElasticSearch (ES)                 | 核心概念与作用                                                                             |
| :------------------------ | :--------------------------------- | :----------------------------------------------------------------------------------------- |
| **Database** (数据库)     | **集群 / 索引 (Indexes)**          | ES 7.x 以后彻底取消了 Type(类) 的概念。一个 Index 可以直接对标 RDBMS 的一张独立 Table 表。 |
| **Table** (表)            | **Index** (索引)                   | 物理和逻辑上的文档合集。                                                                   |
| **Row** (数据行)          | **Document** (文档)                | ES 以 **JSON 规范数据包**作为最小存取单位。对标 C# 实体类转化后的 JSON 串。                |
| **Column** (列)           | **Field** (字段)                   | 文档属性定义。                                                                             |
| **Schema** (模式/表结构)  | **Mapping** (映射)                 | 定义字段数据类型（如 `keyword` 关键字, `text` 分词文本）。                                 |
| **SQL 查询**              | **DSL** (Domain Specific Language) | 基于 HTTP 提交的、结构化的 JSON 规范查询。                                                 |
| **主键 / 聚簇索引**       | **`_id`**                          | 每一条 Document 唯一的物理检索指纹。                                                       |

---

## 📖 章节导航

请按照以下章节顺序展开研读：

0. **[elasticsearch/ch0_setup.md](elasticsearch/ch0_setup.md)**：极简 Docker 容器环境搭建与 Kibana 测试沙盒。
   - 利用 Docker Compose 15 秒极速拉起 ES 8.x + Kibana（带密码/无密免密测试环境安装）、Kibana DevTools 极速指令。
1. **[elasticsearch/ch1_basics.md](elasticsearch/ch1_basics.md)**：核心分词与 REST APIs 经典操作。
   - 核心倒排索引（Inverted Index）底层图解、Mapping 解析与分词器（IK Analyzer 中文分词）；经典的文档 CRUD API。
2. **[elasticsearch/ch2_dsl_query.md](elasticsearch/ch2_dsl_query.md)**：高精密 DSL 搜索、过滤与聚合（对标 C# SQL 复杂查询）。
   - 全文匹配 (`match`) 对标分词查找、精确查询 (`term`) 对齐、布尔组合查询 (`bool` 包含 `must/should/filter`)，高亮字段展现。
3. **[elasticsearch/ch3_dotnet_nest.md](elasticsearch/ch3_dotnet_nest.md)**：C# (.NET Core / .NET 8) 高性能集成与 NEST 实战。
   - 利用最新官方客户端 `Elastic.Clients.Elasticsearch` 或 `NEST`，编写真实的 C# 数据索引添加、高精密搜索、对象高亮解析 DTO 终极实战项目。

---

现在，让我们从最首要的容器环境拉起开始。点击阅读 **[elasticsearch/ch0_setup.md](elasticsearch/ch0_setup.md)**！
