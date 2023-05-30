这里是使用LlamaIndex的入门示例。首先，确保您已经遵循[安装]（安装.md）步骤。

###下载

LlamaIndex示例可以在LlamaIndex存储库的“示例”文件夹中找到。我们首先要下载这个“示例”文件夹。一种简单的方法是克隆存储库：

```bash
$ git clone https://github.com/jerryjliu/llama_index.git
```

接下来，导航到您新克隆的存储库，并验证内容：

```bash
$ cd llama_index
$ ls
LICENSE                data_requirements.txt  tests/
MANIFEST.in            examples/              pyproject.toml
Makefile               experimental/          requirements.txt
README.md              llama_index/             setup.py
```

现在我们要导航到以下文件夹：

```bash
$ cd examples/paul_graham_essay
```

其中包含LlamaIndex示例，围绕Paul Graham的文章[“我做了什么”]（http://paulgraham.com/worked.html）。在`TestEssay.ipynb`中已经提供了一套全面的示例。为了本教程的目的，我们可以专注于让LlamaIndex起步的简单示例。

###构建和查询索引

使用以下内容创建一个新的`.py`文件：

```python
from llama_index import GPTVectorStoreIndex, SimpleDirectoryReader

documents = SimpleDirectoryReader('data').load_data()
index = GPTVectorStoreIndex.from_documents(documents)
```

这将在`data`文件夹中构建一个索引（在这种情况下，只包含文章文本）。然后运行以下内容：

```python
query_engine = index.as_query_engine()
response = query_engine.query("What did the author do growing up?")
print(response)
```

您应该得到类似以下内容的响应：`作者写了短篇小说，并试图在IBM 1401上编程。`

###使用日志查看查询和事件

在Jupyter笔记本中，您可以使用以下片段查看信息和/或调试日志：

```python
import logging
import sys

logging.basicConfig(stream=sys.stdout, level=logging.DEBUG)
logging.getLogger().addHandler(logging.StreamHandler(stream=sys.stdout))
```

您可以将级别设置为`DEBUG`以获得详细输出，或使用`level=logging.INFO`以获得较少的输出。

###保存和加载

默认情况下，数据存储在内存中。
要持久化到磁盘（在`./storage`下）：

```python
index.storage_context.persist()
```

要从磁盘重新加载：
```python
from llama_index import StorageContext, load_index_from_storage

#重建存储上下文
storage_context = StorageContext.from_defaults(persist_dir="./storage")
#加载索引
ind### 下一步

就是这样！要了解更多关于LlamaIndex功能的信息，请查看左侧的众多“指南”。如果您有兴趣进一步探索LlamaIndex的工作原理，请查看我们的[入门指南](/guides/primer.rst)。

此外，如果您想玩示例笔记本，请查看[此链接](/reference/example_notebooks.rst)。