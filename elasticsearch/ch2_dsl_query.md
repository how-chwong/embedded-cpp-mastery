# 第2章：高精密 DSL 搜索、过滤与聚合（ES篇）

在 C# 或者是 SQL 架构中，当面临复杂的组合模糊筛选，比如：
> “我们要寻找商品名中带有‘华为’或‘Pura’，并且价格在 5000 到 10000 之间，但是商品的产地不能是‘美国’，最后，凡是带有‘华为’二字的匹配，返回给前端的段落里还要自动带上高亮的色彩红字 `<em style='color:red;'>` 样式。”

要在传统 SQL 或者是 LINQ 里面去拼凑这段逻辑，你会写出海量令团队不忍多看第二眼的类似 `if (req.price != null) query = query.Where(...)` 和各种拼接字符串的代码。

但在 SQL 的世界外，ES 准备了一套极其精密、纯 JSON 结构的查询系统 —— **DSL (Domain Specific Language，域特定查询语言)**。
本章，我们将像战术指挥官一样，攻下高级 DSL 搜索、拼装和聚合这一核心要塞。

---

## 1. 核心实战：基础演示数据录入

立刻打开你的 **Kibana DevTools** 窗口，我们创建一套干净的 `articles` 索引并录入 3 条基础数据。

```http
# 1. 建立具有中文 IK 分词支持的 articles Mapping
PUT /articles
{
  "mappings": {
    "properties": {
      "id": { "type": "long" },
      "title": { "type": "text", "analyzer": "ik_max_word" },
      "tag": { "type": "keyword" }, // 不分词
      "price": { "type": "double" },
      "views": { "type": "integer" }
    }
  }
}

# 2. 批量录入示范数据
POST /_bulk
{ "index" : { "_index" : "articles", "_id" : "1" } }
{ "id": 1, "title": "C# 高并发编程与高性能微服务", "tag": "DotNet", "price": 99.0, "views": 1500 }
{ "index" : { "_index" : "articles", "_id" : "2" } }
{ "id": 2, "title": "Python AI 机器学习与机器视觉极速起步", "tag": "Python", "price": 120.0, "views": 8000 }
{ "index" : { "_index" : "articles", "_id" : "3" } }
{ "id": 3, "title": "C# 极速通关与 SQL Server 高级开发", "tag": "DotNet", "price": 49.0, "views": 600 }
```

---

## 2. 传统精确比对 `term` 与 模糊模糊匹配 `match`（大坑！）

习惯了 SQL `=` 符号的 .NET 程序员，经常会在 ES 里犯下写错查询的低级失误。你必须分清：

### A. `match` (模糊匹配 / 全文检索)
* **本质**：会把用户输入的东西**先也拿去进行一次分词切碎**，然后再去倒排索引里算重叠匹配率。
* **对标 RDBMS**：`LIKE "%C#%"` 或是模糊查询。
* **示例**：
  ```json
  // 会把 "C#微服务" 分割成 "C#" 和 "微服务"，然后检索两个词重叠的所有文章！
  "match": {
    "title": "C#微服务"
  }
  ```

### B. `term` (精确查询 / 值比对)
* **本质**：**直接拿着你的原始输入**，不加任何分词转换，直接去倒排索引库里做 $100\%$ 的字对字等值对齐。
* **对标 RDBMS**：`WHERE tag = 'DotNet'`。
* **示例**：
  ```json
  // 只会比对完全是 "DotNet" 这一串的文章（tag 字段在 Mapping 中定义是 keyword 不分词材质）
  "term": {
    "tag": "DotNet"
  }
  ```

---

## 3. 完美整合：布尔过滤器 `bool` (大合体查询)

在真实的业务大屏、中后台筛选中，你离不开并列组合。在 ES 中，所有条件通过 **`bool` (布尔查询)** 实现：
- **`must`**: **必须满足**。会强制过滤结果，并且参与计算“相关评分（`_score`：即该文章有多符合用户的口味，分越高排在最越前面）”。（对标 SQL `AND`）
- **`must_not`**: **绝对不能满足**。排查不相关的干扰项，不计算相关评分。（对标 SQL `AND NOT`）
- **`should`**: **应该/或者满足**。只要有其中一项命中就算符合，能极大提升相关评分。**（对标 SQL `OR`）**
- **`filter`**: **高能过滤器（重点性能提速绝招）**。
  - **对齐 C#**：类似于 `Where` 过滤，但它**完全不参与计算评分（不烧 CPU）**！而且 ES 具备强大的机制将 filter 扫描到的主键缓存进后台内存虚机。
  - **避坑法门**：**凡是跟模糊检索无关的条件（例如限制非零库存、限制价格区间、限制商品状态），统统都必须写入 `filter` 作用域，以此来大幅提升性能！**

---

### 实战演训：全功能大合体 DSL 检索（在 DevTools 运行测试）

```http
POST /articles/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "title": "C#"
          }
        }
      ],
      "filter": [
        {
          "range": {
            "price": {
              "gte": 40,
              "lte": 150
            }
          }
        }
      ],
      "must_not": [
        {
          "term": {
            "tag": "Python"
          }
        }
      ]
    }
  },
  "highlight": {
    "fields": {
      "title": {}
    },
    "pre_tags": ["<span style='color:red;'>"],
    "post_tags": ["</span>"]
  }
}
```

#### 右侧极有高能量的极速返回包（截取核心部分）：
```json
{
  "hits": {
    "total": { "value": 2 }, 
    "max_score": 0.44106528,
    "hits": [
      {
        "_id": "1",
        "_source": {
          "title": "C# 高并发编程与高性能微服务",
          "price": 99.0
        },
        "highlight": {
          "title": [
            "<span style='color:red;'>C#</span> 高并发编程与高性能微服务"
          ]
        }
      }
    ]
  }
}
```

---

## 4. 强大的实时数据聚合：Metric & Bucket（对标 C# `GroupBy`）

有些时候，我们不想单纯拿数据，我们要统计海量销售单，算出：
- **今天总共有多少个不同大类的商品（对标 SQL `COUNT(DISTINCT tag)`）**
- **每个大类下的文章总浏览量是多少？（对标 SQL `SUM(views) ... GROUP BY tag`）**

在 ES 中，这属于**高精密实时聚合系统 (Aggregations，简称 aggs)**：
- **`Bucket` (桶聚合)**：负责根据指定维度**分门别类**，好比把产品根据 tag 分成一个个透明物理桶。
- **`Metric` (度量聚合)**：负责在分好类的每一个桶上面，计算**求和（SUM）、均值（AVG）、最大值（MAX）、最小值（MIN）**。

### 实战：一键分析不同标签（Tag）下的文章总数与浏览总和

```http
POST /articles/_search
{
  "size": 0, 
  "aggs": {
    "tag_group": {
      "terms": {
        "field": "tag"
      },
      "aggs": {
        "sum_of_views": {
          "sum": {
            "field": "views"
          }
        }
      }
    }
  }
}
```

#### 精致的返回报文结构（数据大屏的最爱）：
```json
{
  "aggregations": {
    "tag_group": {
      "buckets": [
        {
          "key": "DotNet", 
          "doc_count": 2, 
          "sum_of_views": {
            "value": 2100.0
          }
        },
        {
          "key": "Python", 
          "doc_count": 1, 
          "sum_of_views": {
            "value": 8000.0
          }
        }
      ]
    }
  }
}
```

---

现在，最硬核、最强大的 DSL 查询命令逻辑已全盘收入囊中。点击进入 **[elasticsearch/ch3_dotnet_nest.md](elasticsearch/ch3_dotnet_nest.md)**，我们将回到 C# 的大本营，在真实的 ASP.NET / 控制台代码中接入官方 NEST，完成强类型高能全功能的完整应用实战！
