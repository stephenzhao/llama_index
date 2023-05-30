_Ref-Query:

查询索引
=================

本文档展示了用于查询索引的类。

主要查询类
^^^^^^^^^^^^^^^^^^

查询索引涉及三个主要组件：

- **检索器**：检索器类根据查询从索引中检索一组节点。
- **响应合成器**：此类接收一组节点，并根据查询合成答案。
- **查询引擎**：此类接收查询，并返回响应对象。它可以在底层使用检索器和响应合成器模块。
- **聊天引擎**：此类使基于知识库的对话成为可能。它是查询引擎的有状态版本，可跟踪对话历史。


.. toctree::
   :maxdepth: 1
   :caption: 主要查询类

   query/retrievers.rst
   query/response_synthesizer.rst
   query/query_engines.rst
   query/chat_engines.rst


其他查询类
^^^^^^^^^^^^^^^^^^^^^^^^

我们还在下面详细介绍了一些其他查询类。

- **查询包**：这是查询类的输入：检索器，响应合成器和查询引擎。它使用户能够自定义用于基于嵌入的查询的字符串。
- **查询转换**：此类使用相关转换来增强原始查询字符串，以改善索引查询。可以与检索器一起使用（请参阅TransformRetriever）或QueryEngine。


.. toctree::
   :maxdepth: 1
   :caption: 其他查询类

   query/query_bundle.rst
   query/query_transform.rst