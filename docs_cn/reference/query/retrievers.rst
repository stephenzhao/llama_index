_Ref-Retrievers:

检索器
=================

索引检索器
^^^^^^^^^^^^^^^^
下面我们展示特定索引的检索器类。

.. toctree::
   :maxdepth: 1
   :caption: 索引检索器

   retrievers/empty.rst
   retrievers/kg.rst
   retrievers/list.rst
   retrievers/table.rst
   retrievers/tree.rst
   retrievers/vector_store.rst

注意：我们的结构化索引（例如GPTPandasIndex，GPTSQLStructStoreIndex）没有任何检索器，因为它们不是设计用于检索器API的。
有关更多详细信息，请参阅:ref:`查询引擎<Ref-Query-Engines>`页面。

附加检索器
^^^^^^^^^^^^^^^^^^^^^

这里我们展示附加检索器类; 这些类可以用新的功能（例如查询转换）来增强现有的检索器。

.. toctree::
   :maxdepth: 1
   :caption: 附加检索器

   retrievers/transform.rst


基础检索器
^^^^^^^^^^^^^^^^^^^^^

这里我们展示基础检索器类，它包含所有检索器共享的`retrieve`方法。

.. automodule:: llama_index.indices.base_retriever
   :members:
   :inherited-members:
..    :exclude-members: index_struct, query, set_llm_predictor, set_prompt_helper