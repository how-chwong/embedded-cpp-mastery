# 第3章：C# (.NET 8) 宿主极速集成与高级 NEST 实战（ES篇）

在研究完 ES 各种在 Kibana REST 接口下的 DSL 数据交互后，作为 .NET 工程师，我们最紧要的落地任务莫过于：**如何在 C# 业务工程里，以前后端分离的强类型姿势轻松把 ES 用起来？**

本章，我们将：
1. 梳理 .NET 生态中官方客户端包的升级背景。
2. 动手组装一个高标的 C# 控制台，配置支持 **SSL/非SSL** 链接的 `ElasticsearchClient`。
3. 实战编写：**向 ES 批量插入 C# 实体**、以及**用流畅 C# 强类型语法构造 Complex DSL（布尔组合 + 聚合高亮）**并完美解析字段 DTO 返回的终极代码。

---

## 1. 宿主驱动背景：NEST 的进化

在以往（ES 7.x 及以前），.NET 开发者使用的 NuGet 老包叫做 **`NEST`**（对应低级驱动 `Elasticsearch.Net`）。
* **NEST 最大的特色**：支持利用类似 LINQ 式的 Fluent API 链式流畅拼接，深受 C# 开发者喜爱。
* **现代（8.x 时代）**：官方整合了旧驱动，统一升级并重命名为具有划时代意义的官方 NuGet 新包：**`Elastic.Clients.Elasticsearch`**。
* 本章为了让你紧跟当今 .NET 8 生产标准，我们直接采用最新的 **`Elastic.Clients.Elasticsearch` 官方金牌包**来进行高能演示。

---

## 2. 🛠️ 跟着执行：C# 极速组装集成

### 第一步：一键建立 C# 工程并装配 NuGet
1. 新建并进入一个空目录（如 `d:\Workspace\dotnet_es_demo`）。
2. 在终端窗口运行，建立高能控制台并拉取官方最新驱动依赖：
   ```bash
   dotnet new console
   dotnet add package Elastic.Clients.Elasticsearch --version 8.11.0
   ```

### 第二步：彻底替换并编写 `Program.cs`
将项目下的 `Program.cs` 内容一字不漏替换为以下代码。请仔细阅读里面对标 LINQ、Lambda 拼接 DSL 以及高亮属性提取的完整闭环实战：

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
// 引入官方新版客户端核心包
using Elastic.Clients.Elasticsearch;
using Elastic.Clients.Elasticsearch.QueryDsl;

namespace DotNetElasticSearchDemo;

// 1. 定义一个与 ES Mapping 完全对齐的 C# 强类型实体 (Document Entity)
public class ArticleDocument
{
    public long Id { get; set; }
    public string Title { get; set; }
    public string Tag { get; set; }
    public double Price { get; set; }
    public int Views { get; set; }
}

class Program
{
    private const string IndexName = "articles"; // 定义操作的目标索引

    static async Task Main(string[] args)
    {
        Console.WriteLine("🚀 --- .NET 8 与 ElasticSearch 官方高级集成拉起 --- 🚀\n");

        // 2. 初始化核心强类型客户端
        // 如果是本地 Docker-compose 免密开发模式，配置极其精纯：
        var settings = new ElasticsearchClientSettings(new Uri("http://localhost:9200"))
            .DefaultIndex(IndexName); // 设置全局默认索引，免去后续每次手动指定 index 的累赘

        var client = new ElasticsearchClient(settings);

        // ====================================================================
        // 【第一战】：C# 级大批量数据录入 (Bulk API 对标 EF BulkInsert)
        // ====================================================================
        Console.WriteLine(">>> 【Step 1】开始向 ES 注入高能 C# 实体测试数据...");
        
        var documents = new List<ArticleDocument>
        {
            new() { Id = 10, Title = "C# 并发编程经典案例与 .NET8 微服务实战", Tag = "DotNet", Price = 88.5, Views = 2400 },
            new() { Id = 11, Title = "Python PyTorch 深度学习与大模型训练通关", Tag = "Python", Price = 158.0, Views = 9500 },
            new() { Id = 12, Title = "ASP.NET Core 与 Entity Framework 最佳质量管理", Tag = "DotNet", Price = 59.9, Views = 1100 }
        };

        // 一键批量写入，速度极快（底层会将 C# List 秒转为高效极轻 Bulk JSON 字节流并发通信）
        var bulkResponse = await client.IndexManyAsync(documents, IndexName);
        
        if (bulkResponse.IsValidResponse)
            Console.WriteLine("🎉 数据 Bulk 注入成功！倒排索引已光速重建。");
        else
            Console.WriteLine($"❌ 数据注入失败: {bulkResponse.ElasticsearchServerError?.Error.Reason}");


        // ====================================================================
        // 【第二战】：用 C# 强类型 Fluent 拼装高精组合 DSL (布尔 must + filter + 高亮)
        // ====================================================================
        Console.WriteLine("\n>>> 【Step 2】C# 开始拼装高级 3 维 Boolean 过滤多端搜索...");
        
        // 我们期望：
        // a. 标题 title 必须带有 "C#" 全文模糊词 (.must)
        // b. tag 必须是不分词的精确的 "DotNet" (.filter 缓存，不计分，省 GPU)
        // c. views 浏览量必须 >= 1000 (.filter 缓存)
        // d. 附带高亮 highlight 捕获，给 Title 带上高亮的红色外夹 HTML

        var searchResponse = await client.SearchAsync<ArticleDocument>(s => s
            .Query(q => q
                .Bool(b => b
                    .Must(m => m
                        .Match(t => t
                            .Field(f => f.Title) // 强类型感知：自动利用 Lambda 指向 Title 字段，杜绝拼写错误！
                            .Query("C#")
                        )
                    )
                    .Filter(
                        f => f.Term(t => t.Field(field => field.Tag).Value("DotNet")),
                        f => f.Range(r => r.NumberRange(nr => nr.Field(field => field.Views).Gte(1000)))
                    )
                )
            )
            .Highlight(h => h
                .Fields(fields => fields.Add(f => f.Title, hf => {})) // 对 Title 开启高亮
                .PreTags("<span style='color:red;'>")
                .PostTags("</span>")
            )
        );

        // ====================================================================
        // 【第三战】：高精密 DTO 字段提取与高亮重构
        // ====================================================================
        Console.WriteLine("\n>>> 【Step 3】提取并解析检索返回列表 DTO...");

        if (searchResponse.IsValidResponse)
        {
            Console.WriteLine($"📊 本轮搜索共击中: {searchResponse.Total} 个完美文档\n");

            foreach (var hit in searchResponse.Hits)
            {
                // a. 提取原始的高保真数据源实体
                ArticleDocument sourceDoc = hit.Source;

                // b. 核心黑科技：抓取并提取出被 ES 系统高亮红字加工之后的“高亮 Title 段落”
                // 有时候，如果高亮大地图中存在当前字段，我们提取高亮结果；否则，降级使用常规原 title。
                string finalDisplayTitle = sourceDoc.Title;
                
                if (hit.Highlight != null && hit.Highlight.TryGetValue("title", out var highlightCollection))
                {
                    // 提取出第一个被切割标记的高亮文本片
                    var highlightedFragment = highlightCollection.FirstOrDefault();
                    if (!string.IsNullOrEmpty(highlightedFragment))
                    {
                        finalDisplayTitle = highlightedFragment;
                    }
                }

                // 在 C# 命令行高精亮还原打印我们的查询数据
                Console.WriteLine($"[Id: {sourceDoc.Id}]");
                Console.WriteLine($"  ├─ 原 始 标 题: {sourceDoc.Title}");
                Console.WriteLine($"  ├─ 高 亮 标 题: {finalDisplayTitle}"); // 这里会显示带有的红字 HTML
                Console.WriteLine($"  ├─ 标签 / 价格: {sourceDoc.Tag} | 价位: ￥{sourceDoc.Price}");
                Console.WriteLine($"  └─ 瞬 间 阅 读: {sourceDoc.Views} 次 \n");
            }
        }
        else
        {
            Console.WriteLine($"❌ 复杂组合搜索失败: {searchResponse.ElasticsearchServerError?.Error.Reason}");
        }
    }
}
```

### 第三步：大快人心！在终端里一键运行
在终端中进入当前控制台文件夹，确保你第 0 章的 Docker-compose 沙盒处于运行状态：
```bash
dotnet run
```

#### 你会惊喜地在终端中得到类似如下的高精度打印汇总：
```text
🛡️ --- .NET 8 与 ElasticSearch 官方高级集成拉起 --- 🚀

>>> 【Step 1】开始向 ES 注入高能 C# 实体测试数据...
🎉 数据 Bulk 注入成功！倒排索引已光速重建。

>>> 【Step 2】C# 开始拼装高级 3 维 Boolean 过滤多端搜索...

>>> 【Step 3】提取并解析检索返回列表 DTO...
📊 本轮搜索共击中: 1 个完美文档

[Id: 10]
  ├─ 原 始 标 题: C# 并发编程经典案例与 .NET8 微服务实战
  ├─ 高 亮 标 题: <span style='color:red;'>C#</span> 并发编程经典案例与 .NET8 微服务实战
  ├─ 标签 / 价格: DotNet | 价位: ￥88.5
  └─ 瞬 间 阅 读: 2400 次 
```

---

## 🏁 搜索引擎全功能功德金牌结业！

至此，恭喜你已经彻底、硬核地打通了 **ElasticSearch 全文探索的终极版图**！
你已经熟练掌握了：
1. **[第0章: 安装与 Kibana](ch0_setup.md)** 借助强大的 Docker-Compose 在本机构建省心、免密的 ES 与 Kibana 大本营。
2. **[第1章: 分词与倒排](ch1_basics.md)** 了解了 ES 毫秒级百万文本处理的倒排圣杯，以及 IK 中文分词器的安装和 `text` / `keyword` 字段不可逾越的大坑。
3. **[第2章: 高阶 DSL 查询](ch2_dsl_query.md)** 用纯 JSON 代码组筑高弹性的 must、should、filter 以及实时的 Bucket/Metric 分类求和数据大屏聚合。
4. **本章** 更是带你通过 Lambda 表达式，在最新的 .NET 8 官方标准 `Elastic.Clients.Elasticsearch` 下，用纯净的 C# 代码流式组装出了高性能的级联搜索。

你可以将这些高能、高含量的代码复制到你的 ASP.NET Core WEB API 各个 Controller 或 Service 块中，迎接千万级高并发、超密集分布式模糊查询系统的大洗礼吧。祝你的系统固若金汤，一往直前！
