LlamaIndex使用模式

LlamaIndex的一般使用模式如下：
1.加载文档（手动或通过数据加载器）
2.将文档解析为节点
3.构建索引（来自节点或文档）
4.（可选，高级）在其他索引之上构建索引
5.查询索引

## 1.加载文档

第一步是加载数据。此数据以“文档”对象的形式表示。
我们提供了各种[数据加载器]（/ how_to / data_connectors.md），它们将通过“load_data”函数加载文档，例如：

```python
from llama_index import SimpleDirectoryReader

documents = SimpleDirectoryReader('data').load_data()
```

您也可以选择手动构造文档。 LlamaIndex公开了`Document`结构。

```python
from llama_index import Document

text_list = [text1, text2, ...]
documents = [Document(t) for t in text_list]
```

文档表示数据源的轻量级容器。您现在可以选择执行以下其中一步：
1.将Document对象直接输入索引（参见第3节）。
2.首先将文档转换为节点对象（参见第2节）。

## 2.将文档解析为节点

接下来的步骤是将这些Document对象解析为Node对象。节点表示源文档的“块”，无论是文本块，图像还是其他。它们还包含与其他节点和索引结构的元数据和关系信息。

节点是LlamaIndex的一等公民。您可以选择直接定义节点及其所有属性。您也可以通过我们的`NodeParser`类“解析”源文档为节点。

例如，您可以这样做

```python
from llama_index.node_parser import SimpleNodeParser

parser = SimpleNodeParser()

nodes = parser.get_nodes_from_documents(documents)
```

您也可以选择手动构造Node对象，并跳过第一节。例如，

```python
from llama_index.data_structs.node import Node, DocumentRelationship

node1 = Node(text="<text_chunk>", doc_id="<node_id>")
node2 = Node(text="<text_chunk>", doc_id="<node_id>")
# set relationships
node1.relationships[DocumentRelationship.NEXT] = node2.get_doc_id()
node2.relationships[DocumentRelationship.PREVIOUS] = node1.get_doc_id()
nodes = [node1, node2]
```


## 3.索引构建

我们现在可以在这些Document对象上构建索引。最简单的高级抽象是在索引初始化期间加载Document对象（如果您直接从步骤1开始并跳过步骤2，则此步骤很重要）。

```python
fromllama_index导入GPTVectorStoreIndex

索引= GPTVectorStoreIndex.from_documents（文档）

您还可以选择直接在一组Node对象上构建索引（这是第2步的延续）。

```python
from llama_index import GPTVectorStoreIndex

index = GPTVectorStoreIndex（nodes）
```

根据您使用的索引，LlamaIndex可能会调用LLM来构建索引。

### 在索引结构中重用节点

如果您定义了多个Node对象，并希望在多个索引结构中共享这些Node对象，则可以这样做。
只需实例化一个StorageContext对象，
将Node对象添加到底层DocumentStore中，
并传递StorageContext。

```python
from llama_index import StorageContext

storage_context = StorageContext.from_defaults（）
storage_context.docstore.add_documents（nodes）

index1 = GPTVectorStoreIndex（nodes，storage_context = storage_context）
index2 = GPTListIndex（nodes，storage_context = storage_context）
```

**注意**：如果未指定`storage_context`参数，则在构建索引时会隐式为每个索引创建它。您可以通过`index.storage_context`访问与给定索引关联的docstore。

### 插入文档或节点

您还可以利用索引的`insert`功能一次插入一个Document对象，而不是在构建索引时插入。

```python
from llama_index import GPTVectorStoreIndex

index = GPTVectorStoreIndex（[]）
for doc in documents：
    index.insert（doc）
```

如果要直接插入节点，可以使用`insert_nodes`函数。

```python
from llama_index import GPTVectorStoreIndex

# nodes：Sequence [Node]
index = GPTVectorStoreIndex（[]）
index.insert_nodes（nodes）
```

有关详细信息和示例笔记本，请参阅[Update Index How-To](/how_to/index_structs/update.md)。

### 自定义LLM

默认情况下，我们使用OpenAI的`text-davinci-003`模型。构建索引时，您可以选择使用另一个LLM。

```python
from llama_index import LLMPredictor，GPTVectorStoreIndex，PromptHelper，ServiceContext
from langchain import OpenAI

...

# define LLM
llm_predictor = LLMPredictor（llm = OpenAI（temperature = 0，model_name =“text-davinci-003”））

# define prompt helper
# set maximum input size
max_input_size = 4096
# set number of output tokens
num_output = 256
# set maximum chunk overlap
max_chunk_overlap = 20
prompt_helper = PromptHelper（max_input_size，num_output，max_chunk_overlap）

service_context = ServiceContext.from_defaults（llm_predictor = llm_predictor，prompt_helper = prompt_helper）

indexGPTVectorStoreIndex.from_documents（文档，service_context = service_context）

更多详细信息，请参阅[自定义LLM的How-To](/how_to/customization/custom_llms.md)。

### 全局ServiceContext

如果您想要上一节中的服务上下文始终是默认值，则可以如下配置：

```python
from llama_index import set_global_service_context
set_global_service_context(service_context)
```

如果在LlamaIndex函数中未指定为关键字参数，则将始终使用此服务上下文作为默认值。

有关服务上下文的更多详细信息，包括如何创建全局服务上下文，请参阅[自定义ServiceContext](/how_to/customization/service_context.md)页面。

### 自定义提示

根据所使用的索引，我们使用默认的提示模板来构建索引（以及插入/查询）。
有关如何自定义提示的更多详细信息，请参阅[自定义提示How-To](/how_to/customization/custom_prompts.md)。

### 自定义嵌入

对于基于嵌入的索引，您可以选择传入自定义嵌入模型。有关更多详细信息，请参阅[自定义嵌入How-To](custom-embeddings)。

### 成本预测器

创建索引，插入索引和查询索引可能会使用令牌。我们可以通过这些操作的输出来跟踪令牌使用情况。在运行操作时，将打印令牌使用情况。

您还可以通过`index.llm_predictor.last_token_usage`获取令牌使用情况。有关更多详细信息，请参阅[成本预测器How-To](/docs/how_to/analysis/cost_analysis.md)。

### [可选] 保存索引以备将来使用

默认情况下，数据存储在内存中。
要持久化到磁盘：

```python
index.storage_context.persist(persist_dir="<persist_dir>")
```
您可以省略persist_dir以默认情况下持久化到`./storage`。

要从磁盘重新加载：
```python
from llama_index import StorageContext, load_index_from_storage

# 重建存储上下文
storage_context = StorageContext.from_defaults(persist_dir="<persist_dir>")

# 加载索引
index = load_index_from_storage(storage_context)
```

**注意**：如果您使用自定义的`ServiceContext`对象初始化索引，则在`load_index_from_storage`期间也需要传入相同的ServiceContext。

```python

service_context = ServiceContext.from_defaults(llm_predictor=llm_predictor)

# 首次构建索引
index = GPTVectorStoreIndex.from_documents(
    documents, service_context=service_context
)

...

# 从磁盘加载索引
index = load_index_from_storage(
    service_context=service_context,
)

```[可选，高级]在其他索引之上构建索引
您可以在其他索引之上构建索引！可组合性为您提供了更大的索引异构数据源的能力。有关相关用例的讨论，请参阅我们的[查询用例](/use_cases/queries.md)。有关技术细节和示例，请参阅我们的[组合How-To](/how_to/index_structs/composability.md)。

## 5.查询索引。

构建索引后，您现在可以使用“QueryEngine”查询它。请注意，“查询”只是LLM的输入 - 这意味着您可以使用索引进行问答，但您还可以做更多！

### 高级API
首先，您可以使用默认的“QueryEngine”（即使用默认配置）查询索引，如下所示：

```python
query_engine = index.as_query_engine()
response = query_engine.query("What did the author do growing up?")
print(response)

response = query_engine.query("Write an email to the user given their background information.")
print(response)
```

### 低级API
我们还支持低级组合API，它为您提供了更细粒度的查询逻辑控制。
下面我们突出显示了一些可能的自定义。
```python
from llama_index import (
    GPTVectorStoreIndex,
    ResponseSynthesizer,
)
from llama_index.retrievers import VectorIndexRetriever
from llama_index.query_engine import RetrieverQueryEngine
from llama_index.indices.postprocessor import SimilarityPostprocessor

# build index
index = GPTVectorStoreIndex.from_documents(documents)

# configure retriever
retriever = VectorIndexRetriever(
    index=index,
    similarity_top_k=2,
)

# configure response synthesizer
response_synthesizer = ResponseSynthesizer.from_args(
    node_postprocessors=[
        SimilarityPostprocessor(similarity_cutoff=0.7)
    ]
)

# assemble query engine
query_engine = RetrieverQueryEngine(
    retriever=retriever,
    response_synthesizer=response_synthesizer,
)

# query
response = query_engine.query("What did the author do growing up?")
print(response)
```
您还可以通过实现相应的接口来添加自己的检索，响应合成和整体查询逻辑。

有关实现的组件的完整列表以及支持的配置，请参阅详细的[参考文档](/reference/query.rst)。

在下面，我们详细讨论一些常用的配置。
### 配置检索器
索引可以具有各种特定于索引的检索模式。
例如，列表索引支持默认的`ListIndexRetriever`，它可以检索所有节点，以及`ListIndexEmb您可以使用以下简写：

```python
    # ListIndexRetriever
    retriever = index.as_retriever(retriever_mode='default')
    # ListIndexEmbeddingRetriever
    retriever = index.as_retriever(retriever_mode='embedding')
```

选择您想要的检索器后，您可以构建查询引擎：

```python
query_engine = RetrieverQueryEngine(retriever)
response = query_engine.query("What did the author do growing up?")
```

每个索引的完整检索器列表（及其简写）都在[查询参考](/reference/query.rst)中有文档。

（setting-response-mode）=
### 配置响应合成
检索器获取相关节点后，`ResponseSynthesizer`通过组合信息合成最终响应。

您可以通过以下方式配置它：
```python
query_engine = RetrieverQueryEngine.from_args(retriever, response_mode=<response_mode>)
```

现在，我们支持以下选项：
- `default`：通过顺序处理每个检索到的`Node`来“创建和完善”答案;
    这会为每个节点单独调用LLM。适合详细答案。
- `compact`：在每次LLM调用期间“压缩”提示，以填充尽可能多的`Node`文本块。如果有
    要塞入一个提示的块太多，请通过多个提示“创建和完善”答案。
- `tree_summarize`：给定一组`Node`对象和查询，递归构建树
    并将根节点作为响应返回。适用于读取摘要
- `no_text`：仅运行检索器以获取将发送到LLM的节点，
    而不实际发送它们。然后可以通过检查`response.source_nodes`来检查它们。
    响应对象在第5节中有更详细的介绍。
- `accumulate`：给定一组`Node`对象和查询，将查询应用于每个`Node`文本
    块，同时将响应累积到数组中。返回所有
    响应的连接字符串。当您需要对每个文本
    块单独运行相同的查询时很有用。我们还支持高级的节点过滤和增强，以进一步提高检索节点对象的相关性。这可以帮助减少时间/ LLM调用/成本的数量或提高响应质量。例如：* `KeywordNodePostprocessor`：通过`required_keywords`和`exclude_keywords`过滤节点。* `SimilarityPostprocessor`：通过设置相似性分数的阈值来过滤节点（因此仅支持基于嵌入的检索器）* `PrevNextNodePostprocessor`：基于`Node`关系增强检索的`Node`对象的其他相关上下文。要配置所需的节点后处理器：
```python
node_postprocessors = [
    KeywordNodePostprocessor(
        required_keywords=["Combinator"],
        exclude_keywords=["Italy"]
    )
]
query_engine = RetrieverQueryEngine.from_args(
    retriever, node_postprocessors=node_postprocessors
)
response = query_engine.query("What did the author do growing up?")
```
解析响应：返回的对象是[`Response`对象](/reference/response.rst)。该对象包含响应文本以及响应的“源”：
```python
response = query_engine.query("<query_str>")

# get response
# response.response
str(response)

# get sources
response.source_nodes
# formatted sources
response.get_formatted_sources()
```