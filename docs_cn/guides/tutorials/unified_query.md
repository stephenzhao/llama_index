LlamaIndex提供了各种不同的[查询用例](/use_cases/queries.md)。

对于简单的查询，我们可能希望使用单个索引数据结构，例如`GPTVectorStoreIndex`用于语义搜索，或`GPTListIndex`用于摘要。

对于更复杂的查询，我们可能希望使用可组合的图。

但是，我们如何将索引和图集成到我们的LLM应用程序中？不同的索引和图可能更适合您可能想要运行的不同类型的查询。

在本指南中，我们将展示如何将不同索引/图结构的各种用例统一在**单一**查询框架下。

### 设置

在本例中，我们将分析不同城市的维基百科文章：波士顿，西雅图，旧金山等。

以下代码片段将相关数据下载到文件中。

```python

from pathlib import Path
import requests

wiki_titles = ["Toronto", "Seattle", "Chicago", "Boston", "Houston"]

for title in wiki_titles:
    response = requests.get(
        'https://en.wikipedia.org/w/api.php',
        params={
            'action': 'query',
            'format': 'json',
            'titles': title,
            'prop': 'extracts',
            # 'exintro': True,
            'explaintext': True,
        }
    ).json()
    page = next(iter(response['query']['pages'].values()))
    wiki_text = page['extract']

    data_path = Path('data')
    if not data_path.exists():
        Path.mkdir(data_path)

    with open(data_path / f"{title}.txt", 'w') as fp:
        fp.write(wiki_text)

```

下一个片段将所有文件加载到Document对象中。

```python
# Load all wiki documents
city_docs = {}
for wiki_title in wiki_titles:
    city_docs[wiki_title] = SimpleDirectoryReader(input_files=[f"data/{wiki_title}.txt"]).load_data()

```


### 定义索引集

我们现在将定义一组索引和图形，以查看您的数据。您可以将每个索引/图形视为解决特定用例的轻量级结构。

我们首先将定义一个向量索引，用于每个城市的文档。

```python
from llama_index import GPTVectorStoreIndex, ServiceContext, StorageContext
from langchain.llms.openai import OpenAIChat

# set service context
llm_predictor_gpt4 = LLMPredictor(llm=OpenAIChat(temperature=0, model_name="gpt-4"))
service_context = ServiceContext.from_defaults(
    llm_predictor=llm_predictor_gpt4, chunk_size_limit=1024
)

# Build city document index
vector_indices = {}
for wiki_title in wiki_titles:
    storage_context = StorageContext.from_defaults()
    vector_indices[wiki_title] = GPTVectorStoreIndex(
        documents=city_docs[wiki_title],
        service_context=service_context,
        storage_context=storage_context
    )

```

翻译结果：LlamaIndex提供了各种不同的[查询用例](/use_cases/queries.md)。对于简单的查询，我们可能希望使用单个索引数据结构，例如`GPTVectorStoreIndex`用于语义搜索，或`GPTListIndex`用于摘要。对于更复杂的查询，我们可能希望使用可组合的图。但是，我们如何将索引和图集成到我们的LLM应用程序中？不同的索引和图可能更适合您可能想要运行的不同类型的查询。在本指南中，我们将展示如何将不同索引/图结构的各种用例统一在**单一**查询框架下。

### 设置

在本例中，我们将分析不同城市的维基百科文章：波士顿，西雅图，旧金山等。以下代码片段将相关数据下载到文件中。

### 定义索引集

我们现在将定义一组索引和图形，以查看您的数据。您可以将每个索引/图形视为解决特定用例的轻量级结构。我们首先将定义一个向量索引，用于每个城市的文档。我们现在将定义一个组合图，以运行**比较/对比**查询（参见[用例文档]（/ use_cases / queries.md））。这个图包含一个基于现有向量索引构建的关键字表。
为此，我们首先要为每个向量索引设置“摘要文本”。
然后，我们在这些向量索引之上构建一个关键字表，使用这些索引和摘要来构建图。
查询此图（使用查询转换模块）可以轻松地在不同城市之间进行比较/对比。下面是一个例子。定义统一查询接口

现在我们已经定义了一组索引/图，我们想要构建一个外部抽象层，提供统一的查询接口来访问我们的数据结构。这意味着在查询时，我们可以查询这个外部抽象层，并相信正确的索引/图将被用于工作。

有几种方法可以做到这一点，既在我们的框架内，也在框架外！
- 在现有索引/图上构建一个路由查询引擎
- 将每个索引/图定义为代理框架（例如LangChain）中的一个工具。

为了本教程的目的，我们采用前者的方法。如果您想了解后者的工作原理，请查看[我们的示例教程](/guides/tutorials/building_a_chatbot.md)。

让我们看一个构建路由器查询引擎以自动“路由”任何查询到您在后台定义的索引/图集的示例。

首先，我们为我们要路由查询的索引/图集定义查询引擎。我们还为每个查询引擎提供一个描述（关于它所持有的数据以及它的用途），以帮助路由器根据特定查询来选择它们。

```python
from llama_index.tools.query_engine import QueryEngineTool

query_engine_tools = []

# add vector index tools
for wiki_title in wiki_titles:
    index = vector_indices[wiki_title]
    summary = index_summaries[wiki_title]
    
    query_engine = index.as_query_engine(service_context=service_context)
    vector_tool = QueryEngineTool.from_query_engine(
        query_engine,
        description=f"Vector index for {wiki_title}",
        extra_info={'index_summary': summary},
    )
    query_engine_tools.append(vector_tool)

# add graph tool
graph_tool = QueryEngineTool.from_query_engine(
    graph.as_query_engine(custom_query_engines=custom_query_engines),
    description="Graph for comparing and contrasting cities",
    extra_info={'index_summary': graph.index_struct.summary},
)
query_engine_tools.append(graph_tool)
```

Finally, we define the router query engine that will route our query to the right index/graph. We can also define a custom scoring function to help the router decide which index/graph to use for a given query.

```python
from llama_index.query_engine.router_query_engine import RouterQueryEngine

# define router query engine
router_query_engine = RouterQueryEngine(
    query_engine_tools,
    scoring_fn=lambda query_str, tool: tool.extra_info['index_summary']['coverage'],
    verbose=True,
)
```

Now that we have our router query engine, we can query it with any query string and it will automatically route the query to the right index/graph.

```python
# query the router query engine
query_str = (
    "Compare and contrast the arts and culture of Houston and Boston. "
)
response_router = router_query_engine.query(query_str)
```现在，我们可以定义路由逻辑和整体路由查询引擎。这里，我们使用`LLMSingleSelector`，它使用LLM来选择底层查询引擎来路由查询。

```python
from llama_index.query_engine.router_query_engine import RouterQueryEngine
from llama_index.selectors.llm_selectors import LLMSingleSelector


router_query_engine = RouterQueryEngine(
    selector=LLMSingleSelector.from_defaults(service_context=service_context),
    query_engine_tools=query_engine_tools
)
```

### 查询我们的统一接口

统一查询接口的优势在于它可以处理不同类型的查询。

它现在可以处理关于特定城市的查询（通过路由到特定城市向量索引），也可以比较/对比不同的城市。

让我们看看几个例子！

**提出比较/对比问题**

```python
# ask a compare/contrast question 
response = router_query_engine.query(
    "Compare and contrast the arts and culture of Houston and Boston.",
)
print(str(response)
```


**提出关于特定城市的问题**

```python

response = router_query_engine.query("What are the sports teams in Toronto?")
print(str(response))

```

这个“外部”抽象能够通过路由到正确的底层抽象来处理不同的查询。