LlamaIndex提供了许多高级功能，由LLM驱动，既可以将非结构化数据转换为结构化数据，也可以通过增强的文本到SQL功能来分析这些结构化数据。
本指南将帮助您了解这些功能。具体来说，我们将涵盖以下主题：
-**推断结构化数据点**：将非结构化数据转换为结构化数据。
-**文本到SQL（基本)**：如何使用自然语言查询一组表。
-**注入上下文**：如何为文本到SQL提示注入上下文。上下文可以手动添加，也可以从非结构化文档中派生。
-**在索引中存储表上下文**：默认情况下，我们会直接将上下文插入提示中。有时，如果上下文很大，这是不可行的。在这里，我们将展示如何实际使用LlamaIndex数据结构来包含表上下文！
我们将通过一个包含城市/人口/国家信息的玩具表格来讲解。
##设置
首先，我们使用SQLAlchemy来设置一个简单的sqlite db：
```python
from sqlalchemy import create_engine, MetaData, Table, Column, String, Integer, select, column

engine = create_engine("sqlite:///:memory:")
metadata_obj = MetaData(bind=engine)

```

然后，我们创建一个玩具`city_stats`表：
```python
# create city SQL table
table_name = "city_stats"
city_stats_table = Table(
    table_name,
    metadata_obj,
    Column("city_name", String(16), primary_key=True),
    Column("population", Integer),
    Column("country", String(16), nullable=False),
)
metadata_obj.create_all()
```

现在是时候插入一些数据点了！

如果您想通过从非结构化数据推断结构化数据点来填充此表，请查看下面的部分。否则，您可以选择直接填充此表：

```python
from sqlalchemy import insert
rows = [
    {"city_name": "Toronto", "population": 2731571, "country": "Canada"},
    {"city_name": "Tokyo", "population": 13929286, "country": "Japan"},
    {"city_name": "Berlin", "population": 600000, "country": "Germany"},
]
for row in rows:
    stmt = insert(city_stats_table).values(**row)
    with engine.connect() as connection:
        cursor = connection.execute(stmt)
```

最后，我们可以使用我们的SQLDatabase包装器将SQLAlchemy引擎包装起来;
这样可以在LlamaIndex中使用数据库：

```python
from llama_index import SQLData基础

sql_database = SQLDatabase（引擎，include_tables = ["city_stats"])

如果数据库已经填充了数据，我们可以使用空文档列表实例化SQL索引。否则请参阅下面的部分。

```python
index = GPTSQLStructStoreIndex（
    []，
    sql_database = sql_database，
    table_name =“city_stats”，
)
```

## 推断结构数据点

LlamaIndex提供将非结构化数据点转换为结构化数据的功能。在本节中，我们将展示如何通过摄取有关每个城市的维基百科文章来填充“city_stats”表。

首先，我们使用LlamaHub中的Wikipedia阅读器加载一些关于相关数据的页面。

```python
from llama_index import download_loader

WikipediaReader = download_loader（“WikipediaReader”)
wiki_docs = WikipediaReader（).load_data（pages = ['Toronto'，'Berlin'，'Tokyo'])

```

当我们构建SQL索引时，我们可以将这些文档指定为第一个输入；这些文档将被转换为结构化数据点并插入数据库：

```python
from llama_index import GPTSQLStructStoreIndex，SQLDatabase

sql_database = SQLDatabase（engine，include_tables = ["city_stats"])
#注意：这里指定的表名是你
#要从非结构化文档中提取的表。
index = GPTSQLStructStoreIndex.from_documents（
    wiki_docs，
    sql_database = sql_database，
    table_name =“city_stats”，
)
```

您可以查看当前表以验证数据点已插入！

```python
#查看当前表
stmt = select（
    [column（“city_name”)，column（“population”)，column（“country”)]
)。从city_stats_table中选择

with engine.connect（)as connection：
    results = connection.execute（stmt).fetchall（)
    print（results)
```

## Text-to-SQL（基本)

LlamaIndex提供“text-to-SQL”功能，既可以在非常基本的层次上，也可以在更高级别上使用。在本节中，我们将展示如何在基本层次上使用这些text-to-SQL功能。

这里有一个简单的例子：

```python
#将Logging设置为DEBUG以获得更详细的输出
query_engine = index.as_query_engine（)
response = query_engine.query（“哪个城市人口最多？”)
print（response)

```

您可以通过`response.extra_info ['sql_query']`访问底层派生的SQL查询。
它应该看起来像这样：
```sql
SELECT city_name，population
FROM city_stats
ORDER BY population DESC
LIMIT 1
```

## 注入上下文

默认情况下，text-to-SQL提示只会将表模式信息注入提示中。
但是，通常您可能希望添加自己的上下文此部分将向您展示如何手动添加上下文，或者通过文档提取上下文。

我们提供了一个上下文构建器类来更好地管理SQL表中的上下文：`SQLContextContainerBuilder`。该类接受`SQLDatabase`对象以及其他一些可选参数，并生成一个`SQLContextContainer`对象，您可以在构建+查询时将其传递给索引。

您可以手动将上下文添加到上下文构建器中。下面的代码片段向您展示了如何操作：

```python
# 手动设置文本
city_stats_text = (
    "This table gives information regarding the population and country of a given city.\n"
    "The user will query with codewords, where 'foo' corresponds to population and 'bar'"
    "corresponds to city."
)
table_context_dict={"city_stats": city_stats_text}
context_builder = SQLContextContainerBuilder(sql_database, context_dict=table_context_dict)
context_container = context_builder.build_context_container()

# 构建索引
index = GPTSQLStructStoreIndex.from_documents(
    wiki_docs,
    sql_database=sql_database,
    table_name="city_stats",
    sql_context_container=context_container
)
```

您还可以选择从一组非结构化文档中**提取**上下文。
要做到这一点，您可以调用`SQLContextContainerBuilder.from_documents`。
我们使用`TableContextPrompt`和`RefineTableContextPrompt`（参见[参考文档](/ reference / prompts.rst))。

```python
# 这是一个我们将从GPTSQLContextContainerBuilder中提取上下文的虚拟文档
city_stats_text = (
    "This table gives information regarding the population and country of a given city.\n"
)
context_documents_dict = {"city_stats": [Document(city_stats_text)]}
context_builder = SQLContextContainerBuilder.from_documents(
    context_documents_dict,
    sql_database
)
context_container = context_builder.build_context_container()

# 构建索引
index = GPTSQLStructStoreIndex.from_documents(
    wiki_docs,
    sql_database=sql_database,
    table_name="city_stats",
    sql_context_container=context_container,
)
```

## 将表上下文存储在索引中

一个数据库集合可以有许多表，如果每个表都有许多列+与之相关联的描述，那么总的上下文可能会很大。

幸运的是，您可以选择使用LlamaIndex数据结构来存储此表上下文！
然后，当查询SQL索引时，我们可以使用这个“侧面”索引来检索适当的上下文，该上下文可以输入文本到SQL提示。

这里我们使用`derive_index_from_context`函数在`SQLContextContainerBuilder`中，您可以自由选择指定哪个索引类以及传入哪些参数来创建一个新的索引。然后，我们使用一个名为`query_index_for_context`的辅助方法，它是对`query`调用的简单封装，它包装了一个查询模板并将上下文存储在生成的上下文容器中。

然后，您可以构建上下文容器，并在查询时将其传递给索引！

```python
from llama_index import GPTSQLStructStoreIndex, SQLDatabase, GPTVectorStoreIndex
from llama_index.indices.struct_store import SQLContextContainerBuilder

sql_database = SQLDatabase(engine)
# 从表架构信息构建一个向量索引
context_builder = SQLContextContainerBuilder(sql_database)
table_schema_index = context_builder.derive_index_from_context(
    GPTVectorStoreIndex,
    store_index=True
)

query_str = "Which city has the highest population?"

# 使用辅助方法查询表架构索引，以检索表上下文
SQLContextContainerBuilder.query_index_for_context(
    table_schema_index,
    query_str,
    store_context_str=True
)

# 使用表上下文查询SQL索引
query_engine = index.as_query_engine()
response = query_engine.query(query_str, sql_context_container=context_container)
print(response)

```

## 最后总结

这就是现在的情况！我们一直在寻找改善结构化数据支持的方法。如果您有任何疑问，请在[我们的Discord](https://discord.gg/dGcwcsnxhU)中告诉我们。