嵌入
=================

当涉及到嵌入时，用户有几个选择。

- :code:`OpenAIEmbedding`：默认嵌入类。默认为“text-embedding-ada-002”
- :code:`LangchainEmbedding`：Langchain嵌入模型的包装器。

我们还引入了一个:code:`LangchainEmbedding`类，它是Langchain嵌入模型的包装器。可以在这里找到完整的嵌入列表<https://langchain.readthedocs.io/en/latest/reference/modules/embeddings.html>`_。

.. automodule:: llama_index.embeddings.langchain
   :members:
   :inherited-members: