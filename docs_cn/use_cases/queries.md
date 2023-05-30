查询您的数据

从高层次上讲，LlamaIndex为您提供了查询数据以进行任何下游LLM用例（如问答，摘要或聊天机器人的组件）的能力。

本节介绍了您可以使用LlamaIndex查询数据的不同方法，大致按照最简单（top-k语义搜索）到更高级功能的顺序排列。

### 语义搜索

LlamaIndex最基本的用法是通过语义搜索。我们提供了一个简单的内存向量存储，以便您开始使用，但您也可以选择使用我们的任何一个[向量存储集成]（/ how_to / integrations / vector_stores.md）：

```python
from llama_index import GPTVectorStoreIndex, SimpleDirectoryReader
documents = SimpleDirectoryReader('data').load_data()
index = GPTVectorStoreIndex.from_documents(documents)
query_engine = index.as_query_engine()
response = query_engine.query("What did the author do growing up?")
print(response)

```

**教程**
- [快速入门]（/ getting_started / starter_example.md）

**指南**
- [示例]（../examples/vector_stores/SimpleIndexDemo.ipynb）（[笔记本]（https://github.com/jerryjliu/llama_index/tree/main/docs/examples/vector_stores/SimpleIndexDemo.ipynb））

### 摘要

摘要查询需要LLM迭代大多数文档才能综合出答案。例如，摘要查询可以是以下之一：
- “这组文本的摘要是什么？”
- “给我一个关于X人与公司的经历的摘要。”

一般来说，列表索引适合此用例。列表索引默认会遍历所有数据。

实证上，设置`response_mode =“tree_summarize”`也会导致更好的摘要结果。

```python
index = GPTListIndex.from_documents(documents)

query_engine = index.as_query_engine(
    response_mode="tree_summarize"
)
response = query_engine.query("<summarization_query>")
```

### 对结构化数据的查询

LlamaIndex支持对结构化数据（如Pandas DataFrame或SQL数据库）的查询。

以下是一些相关资源：

**教程**

- [文本到SQL的指南]（/ guides / tutorials / sql_guide.md）

**指南**
- [SQL指南（核心）]（../examples/index_structs/struct_indices/SQLIndexDemo.ipynb）（[笔记本]（https://github.com/jerryjliu/llama_index/blob/main/docs/examples/index_structs/struct_indices/SQLIndexDemo.ipynb））
- [SQL指南（设置上下文）]（../examples/index_structs/struct_indices/SQLIndexDemo-Context.ipynb）（[笔记本]（https://github.com/jerryjliu/llama_index/blob/main/docs/examples/index_structs/struct_indices/SQLIndexDemo-Context.ipynb））LlamaIndex支持跨异构数据源的路由。可以通过构建现有数据的图来实现。具体来说，可以在子索引上构建列表索引。列表索引本质上会结合每个节点的信息，因此可以跨异构数据源进行综合。

**指南**
- [可组合性](/how_to/index_structs/composability.md)
- [城市分析](../examples/composable_indices/city_analysis/PineconeDemo-CityAnalysis.ipynb) ([Notebook](https://github.com/jerryjliu/llama_index/blob/main/docs/examples/composable_indices/city_analysis/PineconeDemo-CityAnalysis.ipynb))

### 跨异构数据源的路由

LlamaIndex还支持使用“RouterQueryEngine”跨异构数据源进行路由 - 例如，如果您想要将查询“路由”到底层文档或子索引，可以这样做。

首先，在不同的数据源上构建子索引。然后构建相应的查询引擎，并为每个查询引擎提供描述以获得“QueryEngineTool”。除了上面描述的显式合成/路由流程之外，LlamaIndex还可以支持更一般的多文档查询。它可以通过我们的`SubQuestionQueryEngine`类来实现。给定一个查询，该查询引擎将生成一个“查询计划”，其中包含对子文档的子查询，然后再合成最终答案。

要做到这一点，首先为每个文档/数据源定义一个索引，然后用`QueryEngineTool`（类似于上面）将其包装起来：

```python
from llama_index.tools import QueryEngineTool, ToolMetadata

query_engine_tools = [
    QueryEngineTool(
        query_engine=sept_engine, 
        metadata=ToolMetadata(name='sept_22', description='提供有关Uber季度财务报告的信息')
    ),
    QueryEngineTool(
        query_engine=oct_engine, 
        metadata=ToolMetadata(name='oct_22', description='提供有关Uber季度财务报告的信息')
    ),
    QueryEngineTool(
        query_engine=nov_engine, 
        metadata=ToolMetadata(name='nov_22', description='提供有关Uber季度财务报告的信息')
    )
]
```

然后，我们定义一个`RouterQueryEngine`来管理它们。默认情况下，它使用`LLMSingleSelector`作为路由器，它使用LLM根据描述选择最佳的子索引来路由查询。

```python
from llama_index.query_engine import RouterQueryEngine

query_engine = RouterQueryEngine.from_defaults(
    query_engine_tools=[tool1, tool2]
)

response = query_engine.query(
    "在Notion中，给我一个产品路线图的概要。"
)

```

**指南**
- [Router Query Engine Guide](../examples/query_engine/RouterQueryEngine.ipynb)（[Notebook](https://github.com/jerryjliu/llama_index/blob/main/docs/examples/query_engine/RouterQueryEngine.ipynb)）
- [City Analysis Unified Query Interface](../examples/composable_indices/city_analysis/City_Analysis-Unified-Query.ipynb)（[Notebook](https://github.com/jerryjliu/llama_index/blob/main/docs/examples/composable_indices/city_analysis/PineconeDemo-CityAnalysis.ipynb)）

### 比较/对比查询
您可以在ComposableGraph中显式执行比较/对比查询，使用**查询转换**模块。

```python
from llama_index.indices.query.query_transform.base import DecomposeQueryTransform
decompose_transform = DecomposeQueryTransform(
    llm_predictor_chatgpt, verbose=True
)
```

此模块将帮助您将复杂查询分解为现有索引结构的简单查询。

**指南**
- [查询转换](/how_to/query/query_transformations.md)
- [City Analysis Compare/Contrast Example](../examples//composable_indices/city_analysis/City_Analysis-Decompose.ipynb)（[Notebook](https://github.com/jerryjliu/llama_index/blob/main/docs/examples/composable_indices/city_analysis/City_Analysis-Decompose.ipynb)）

您还可以依靠LLM来*推断*是否执行比较/对比查询（请参见下面的多文档查询）。

### 多文档查询

除了上面描述的显式合成/路由流程之外，LlamaIndex还可以支持更一般的多文档查询。它可以通过我们的`SubQuestionQueryEngine`类来实现。给定一个查询，该查询引擎将生成一个“查询计划”，其中包含对子文档的子查询，然后再合成最终答案。

要做到这一点，首先为每个文档/数据源定义一个索引，然后用`QueryEngineTool`（类似于上面）将其包装起来：LlamaIndex可以支持需要理解时间的查询。它可以通过两种方式来实现：
- 决定查询是否需要利用节点之间的时间关系（prev / next关系）来检索额外的上下文以回答问题。
- 按最新性排序并过滤过时的上下文。RecencyPostprocessorDemo.ipynb（位于es / node_postprocessor文件夹中）