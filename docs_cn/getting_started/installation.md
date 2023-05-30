### 安装和设置

### 从Pip安装

您可以简单地执行：
```
pip install llama-index
```

### 从源代码安装
Git克隆此存储库：`git clone https://github.com/jerryjliu/llama_index.git`。然后执行：

- 如果您想进行可编辑安装（您可以修改源文件)，请执行`pip install -e .`。
- 如果您想安装可选依赖项+用于开发的依赖项（例如，单元测试)，请执行`pip install -r requirements.txt`。

### 环境设置

默认情况下，我们使用OpenAI GPT-3 `text-davinci-003`模型。为了使用它，您必须设置OPENAI_API_KEY。
您可以通过登录[OpenAI的页面并创建新的API令牌](https://beta.openai.com/account/api-keys)来注册API密钥。

您可以在[自定义LLM How-To](/how_to/customization/custom_llms.md)中自定义底层LLM（由Langchain提供)。根据LLM提供商，您可能需要额外的环境密钥+令牌设置。