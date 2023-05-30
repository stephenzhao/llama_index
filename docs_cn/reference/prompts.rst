Prompt Templates
=================

以下是参考提示模板。

我们首先显示默认提示的链接。

然后，我们显示基本提示类，
派生自`Langchain <https://langchain.readthedocs.io/en/latest/modules/prompt.html>`。

默认提示
^^^^^^^^^^^^^^^^^

默认提示列表可以在`这里找到 <https://github.com/jerryjliu/llama_index/blob/main/llama_index/prompts/default_prompts.py>`。

**注意**：我们还为ChatGPT用例策划了一组精炼提示。
ChatGPT精炼提示列表可以在`这里找到 <https://github.com/jerryjliu/llama_index/blob/main/llama_index/prompts/chat_prompts.py>`。


Prompts
^^^^^^^

.. automodule:: llama_index.prompts.prompts
   :members:
   :inherited-members:
   :exclude-members: get_full_format_args


基本提示类
^^^^^^^^^^^^^^^^^

.. automodule:: llama_index.prompts
   :members:
   :inherited-members:
   :exclude-members: Config, construct, copy, dict, from_examples, from_file, get_full_format_args, output_parser, save, template, template_format, update_forward_refs, validate_variable_names, json, template_is_valid