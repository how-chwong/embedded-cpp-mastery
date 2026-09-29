# 第1章：圣杯：倒排索引与 Mapping 强类型定制（ES篇）

作为 .NET 工程师，你极度熟悉 B+ 树索引（聚簇与非聚簇）。在传统 SQL 下，寻找一个包含“开发”字眼的名词，需要全表扫描。
但是在 ElasticSearch 中，它能于百亿级文本数据中，在 10 毫秒内瞬间拉起所有命中结果。

本章我们将：
1. 深入图解 ES 的核心灵魂 —— **倒排索引（Inverted Index）**。
2. 了解让 C# 程序员回归静态类型感觉的安全围栏 —— **Mapping（映射表结构）**。
3. 手把手将最强大的“IK 中文分词器（IK Analyzer）”装入我们的开发集群。

---

## 1. 深度图解：什么是“倒排索引 (Inverted Index)”？

假设我们在数据库有一张 `Articles` (文章表)。

### A. 正向索引 (Forward Index) —— 传统 SQL 的查找方式
我们要找到含有 "C#" 的行，必须逐行（Document）拆解，读取里面的 "Title" 内容，拿字符匹配去扫：

| Page ID (主键) | Title (正向存储)   | SQL 运作模式                          |
| :------------- | :----------------- | :------------------------------------ |
| **Doc 1**      | "C# 高并发编程"    | LIKE "%C#%" -> 全表扫描！             |
| **Doc 2**      | "Python AI 自动化" | LIKE "%C#%" -> 扫描对比...            |
| **Doc 3**      | "C# 入门到精通"    | LIKE "%C#%" -> 扫描对比，第三行命中。 |

### B. 倒排索引 (Inverted Index) —— ES 全文检索的核心
ES 在你将 JSON 录入的瞬间，会把文档**“打碎成一个一个独立的单词（Term）”**，并在后台显存里建立一幅反向对照地图。它连接了 “单词” 到底被哪些 “文档” 所引用：

```
 [ 词条 Term ]  ----------------------> [ 关联的文档 ID 列表 (Posting List) ]
   "C#"        -------------.---------> [ Doc 1, Doc 3 ]
   "高并发"     ------------.`---------> [ Doc 1 ]
   "Python"   ------------------------> [ Doc 2 ]
   "AI"        ------------------------> [ Doc 2 ]
```

- **高能奇效**：当用户输入 "C#" 时，ES **根本不用去扫原先的 10 亿行原始文本**。它只需像翻阅大英百科全书最后的索引附录一样，瞬间在哈希词条地图里抓出 `"C#"`，直接发现后面躺着的就是 `[Doc 1, Doc 3]`！这是一种常数级 $O(1)$ 的无敌瞬间响应。

---

## 2. 让 C# 程序员感到舒适的 Mapping（类型映射定义）

在 C# 里面，我们不喜欢没有 Schema 的自由数据（比如 dynamic），我们讲究强类型安全性（`Entity` 里的 `int id`、`string title`）。
在 ES 中，决定索引内部每个字段性质的强表结构就叫做 **Mapping**。

### A. 两个绝对核心的文本字段区分（高亮警惕！）
当你给一个属性标注为“文本”时，你必须要在 `text` 和 `keyword` 之间做出非此即彼的选择：

1. **`text` (全文分词型)**：
   - 扔进去的字符串，会被 ES **自动打碎**（如 "My Computer" 碎成 "My"、"Computer"）并分别塞入倒排索引。
   - **作用**：适合做大段文本的**全文检索、模糊检索**（如新闻正文、商品标题）。
   - **代价**：**绝对无法用于等值精确查找、排序（Order By）与聚合（Group By）！**
2. **`keyword` (精确不分词型)**：
   - 扔进去的文本，ES 会把它**当成一具雕像整体看待**，不加任何打碎。("My Computer" 索引里就只有这一个长词)。
   - **作用**：适合做高精密度的**等值比对**（例如：订单号、手机号、地区、邮箱、商品状态码）。
   - **特性**：可以高效用于 **排序（Sort）、Filter 过滤、以及高能聚合计算**。

### B. 经典 Mapping 实践演示（在 Kibana DevTools 写入）
我们模拟 C# 的 ProductDapper 实体：
```csharp
public class Product {
    public int Id;
    public string Name; // 需要做中文模糊搜索
    public string SerialNum; // 精密唯一序列号，需要绝对对齐查找
    public decimal Price;
}
```

在 Kibana DevTools 中，一键显式创建它的 Mapping 表：

```http
PUT /products
{
  "mappings": {
    "properties": {
      "product_id": {
        "type": "long"
      },
      "name": {
        "type": "text", // 1. 全文分词：允许用户模糊输入“华为”搜索到整个商品
        "analyzer": "ik_max_word" // 2. 核心大妙：指定加载 IK 中文最细粒度分词器！
      },
      "serial_num": {
        "type": "keyword" // 3. 唯一字段：不需分词，主要用作等值过滤与排序
      },
      "price": {
        "type": "double"
      },
      "published_at": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss || yyyy-MM-dd"
      }
    }
  }
}
```

---

## 3. IK Analyzer：为什么必须为 ES 注入中文分词器？

### 【中文面临的致命大怪兽】
ES 默认自带的英文分词器（Standard）是基于“空格”来区分单词的。比如 "I am a dotnet coder" 会被拆成 "I", "am", "a", "dotnet", "coder"。
但中文句子 **没有空格**！
* 句子："我是微软开发工程师"
* 默认英文分词器拆成：`"我"`、`"是"`、`"微"`、`"软"`、`"开"`、`"发"`、`"工"`、`"程"`、`"师"`！
* **致命灾难**：当用户输入 "微软" 或者是 "开发" 时，ES 需要去倒排索引里拼装，这导致分词检索和高亮功能彻底报废！

为了解决中文分词难题，全球最著名的中文开源分词插件便是 **IK Analyzer**。

---

### 🛠️ 15秒一键装配 IK 中文分词器（Docker 沙盒版）

感谢我们刚才配置的 Docker 平台。我们无需繁琐地去 Github 翻源码、拷贝 zip。直接在命令行终端，对我们的 `elasticsearch` 容器，执行一击必杀命令：

```bash
# 在本地终端中运行以下三步，一键下载并自我装载 IK 插件：

# 1. 指引 ES 容器去下载与其底层版本 8.11.1 完美匹配的 IK 中文分词 zip 包并安装：
docker exec -it elasticsearch bin/elasticsearch-plugin install https://release.infinilabs.com/elasticsearch/plugins/analysis-ik/8.11.1/analysis-ik-8.11.1.zip

# 2. 回车执行，看到控制台输出 -> Installed analysis-ik 证明装填成功！

# 3. 优雅重启你的 ES 集群，使其重新初始化并加载 IK：
docker-compose restart
```

---

### 验证 IK 中文分词的奇迹效果
立刻返回 Kibana 的 **DevTools** 面板，我们用 `_analyze` HTTP 方法对分词进行测试（这是极佳的排产工具）：

```http
# 测试 IK 分词效果（1. ik_smart 为智能粗粒度切分；2. ik_max_word 为最精细、大矩阵切分）
POST /_analyze
{
  "analyzer": "ik_max_word",
  "text": "大微软开发工程师"
}
```

#### 运行后的惊叹返回结果：
右侧会瞬间飞起一段精致的 JSON：
```json
{
  "tokens": [
    { "token": "大" },
    { "token": "微软" }, // 独立拆出：词意完美命中！
    { "token": "开发" }, // 独立拆出：词意完美命中！
    { "token": "工程师" },
    { "token": "工程" },
    { "token": "导师" }
  ]
}
```
此时，倒排索引才算真正学会了人类中文的奥术语法！

---

现在，倒排索引与中文分词、 Mapping 黄金围栏已完全备齐。点击进入 **[elasticsearch/ch2_dsl_query.md](elasticsearch/ch2_dsl_query.md)**，我们将直接开始学习 ES 最强最可怕的兵器 —— 极其强大的 **复杂 DSL（结构化查询）**！
