LlamaIndex是您的数据和LLM之间的接口，它提供了工具包，可以为您的数据设置查询界面，用于任何下游任务，无论是问答，摘要还是其他。

在本教程中，我们将向您展示如何构建上下文增强聊天机器人。我们使用Langchain作为底层Agent / Chatbot抽象，并使用LlamaIndex进行数据检索/查找/查询！结果是一个聊天机器人代理，可以访问LlamaIndex提供的丰富的“数据接口”工具，以回答有关您数据的查询。

**注意**：这是最初构建SEC 10-K文件查询界面的一些初步工作的延续-[在这里查看](https://medium.com/@jerryjliu98/how-unstructured-and-llamaindex-can-help-bring-the-power-of-llms-to-your-own-data-3657d063e30d)。

### 上下文

在本教程中，我们通过从Dropbox下载原始UBER 10-K HTML文件来构建“10-K聊天机器人”。用户可以选择询问有关10-K文件的问题。

### 摄取数据

让我们首先下载原始的10-k文件，从2019-2022年。

```python
# 注意：代码示例假定您正在Jupyter笔记本中操作。
# 下载文件
！mkDIR数据
！wget“https://www.dropbox.com/s/948jr9cfs7fgj99/UBER.zip？dl = 1”-O data / UBER.zip
！unzip data / UBER.zip -d data

```

我们使用[Unstructured](https://github.com/Unstructured-IO/unstructured)库将HTML文件解析为格式化文本。
我们通过[LlamaHub](https://llamahub.ai/)与Unstructured直接集成-这使我们可以将任何文本转换为LlamaIndex可以摄取的文档格式。

```python

from llama_index import download_loader, GPTVectorStoreIndex, ServiceContext, StorageContext, load_index_from_storage
from pathlib import Path

years = [2022, 2021, 2020, 2019]
UnstructuredReader = download_loader("UnstructuredReader", refresh_cache=True)

loader = UnstructuredReader()
doc_set = {}
all_docs = []
for year in years:
    year_docs = loader.load_data(file=Path(f'./data/UBER/UBER_{year}.html'), split_documents=False)
    # 将年份元数据插入每一年
    for d in year_docs:
        d.extra_info = {"year": year}
    doc_set[year] = year_docs
    all_docs.extend(year_docs)
```

### 为每年设置向量索引

我们首先为每年设置一个向量索引。每个向量索引都允许我们
询问有关给定年份的10-K文件的问题。

我们构建每个索引并将其保存到磁盘。

```python
# 初始化简单的向量索引+全局向量索引
service_context = ServiceContext.from_defaults(chunk_size_li我们使用Langchain来设置外部聊天机器人代理，它可以访问一组工具。LlamaIndex提供了一些索引和图形的包装，以便它们可以轻松地被Langchain访问。我们希望为每个索引（对应于给定的年份）以及图定义一个单独的工具，我们可以在一个中央“LlamaToolkit”接口下定义所有工具。

下面，我们为我们的图定义一个“IndexToolConfig”。请注意，我们还导入了一个“DecomposeQueryTransform”模块，用于图中每个向量索引中的使用-这允许我们将整个查询“分解”为可以从每个子索引回答的查询（见下面的示例）。

最后，我们将这些配置与我们的“LlamaToolkit”结合起来：最后，我们调用`create_llama_chat_agent`来创建我们的Langchain聊天机器人代理，它可以访问我们上面定义的5个工具：

```python
memory = ConversationBufferMemory(memory_key="chat_history")
llm=OpenAI(temperature=0)
agent_chain = create_llama_chat_agent(
    toolkit,
    llm,
    memory=memory,
    verbose=True
)
```

### 测试代理

我们现在可以用各种查询来测试代理。

如果我们用一个简单的“hello”查询来测试它，代理不会使用任何工具。

```python
agent_chain.run(input="hi, i am bob")
```

```
> Entering new AgentExecutor chain...

Thought: Do I need to use a tool? No
AI: Hi Bob, nice to meet you! How can I help you today?

> Finished chain.
'Hi Bob, nice to meet you! How can I help you today?'
```

如果我们用一个关于某一年度10-k的查询来测试它，代理将使用相关的向量索引工具。

```python
agent_chain.run(input="What were some of the biggest risk factors in 2020 for Uber?")
```

```
> Entering new AgentExecutor chain...

Thought: Do I need to use a tool? Yes
Action: Vector Index 2020
Action Input: Risk Factors
...

Observation: 

Risk Factors

The COVID-19 pandemic and the impact of actions to mitigate the pandemic has adversely affected and continues to adversely affect our business, financial condition, and results of operations.

...
'\n\nRisk Factors\n\nThe COVID-19 pandemic and the impact of actions to mitigate the pandemic has adversely affected and continues to adversely affect our business,

```

最后，如果我们用一个查询来比较/对比跨年度的风险因素，代理将使用图形索引工具。

```python
cross_query_str = (
    "Compare/contrast the risk factors described in the Uber 10-K across years. Give answer in bullet points."
)
agent_chain.run(input=cross_query_str)
```

```
> Entering new AgentExecutor chain...

Thought: Do I need to use a tool? Yes
Action: Graph Index
Action Input: Compare/contrast the risk factors described in the Uber 10-K across years.> Current query: Compare/contrast the risk factors described in the Uber 10-K across years.
> New query:  What are the risk factors described in the Uber 10-K for the 2022 fiscal year?
> Current query: Compare/contrast the risk factors described in the Uber 10-K across years.
> New query:  What are the risk factors described in the Uber 10-K for the 2022 fiscal year?
INFO:llama_index.token_counter.token_counter:> [query] Total LLM token usage: 964 tokens
INFO:llama_index.token_counter.token_counter:> [query] Total embedding token usage: 18 tokens
> Got response: 
The risk factor
翻译结果：
最后，我们可以用各种查询来测试代理。如果我们用一个简单的“hello”查询来测试它，代理不会使用任何工具。如果我们用一个关于某一年度10-k的查询来测试它，代理将使用相关的向量索引工具。最后，如果我们用一个查询来比较/对比跨年度的风险因素，代理将使用图形索引工具。Uber 10-K报告中描述的2021财年的风险因素包括：司机分类可能发生变化的潜在风险，竞争力可能增加的潜在风险，可能出现的技术问题，可能出现的政策变化，可能出现的政府监管，可能出现的诉讼，可能出现的投资者和消费者信心变化，可能出现的经济不确定性，可能出现的技术和产品发展问题，可能出现的网络安全问题，可能出现的税收变化，可能出现的收入和利润变化，可能出现的货币汇率变化，可能出现的成本和供应链管理问题，可能出现的社会和政治不稳定性，可能出现的环境和自然灾害，以及可能出现的其他风险。观察：2020年，风险因素包括疫苗普及的时机、政府当局可能采取的进一步行动、对司机业务的进一步影响。