存储上下文
=================
LlamaIndex提供了关于节点、索引和向量存储的核心抽象。关键的抽象是`StorageContext`-它包含底层的`BaseDocumentStore`（用于节点)、`BaseIndexStore`（用于索引)和`VectorStore`（用于向量)。

文档/节点和索引存储依赖于一个共同的`KVStore`抽象，详情参见下文。

我们在下面展示了存储类的API参考、从存储上下文加载索引以及存储上下文类本身的参考。

|

.. toctree::
   :maxdepth: 1
   :caption: 存储类

   storage/docstore.rst
   storage/index_store.rst
   storage/vector_store.rst
   storage/kv_store.rst

|

.. toctree::
   :maxdepth: 1
   :caption: 加载索引

   storage/indices_save_load.rst

------------

.. automodule:: llama_index.storage.storage_context
   :members:
   :inherited-members: