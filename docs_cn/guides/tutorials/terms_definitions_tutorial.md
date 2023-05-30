Llama Index有许多使用案例（语义搜索，摘要等），这些都有[很好的文档](https://gpt-index.readthedocs.io/en/latest/use_cases/queries.html)。但是，这并不意味着我们不能将Llama Index应用于非常特定的用例！

在本教程中，我们将深入了解使用Llama Index从文本中提取术语和定义的设计过程，同时允许用户稍后查询这些术语。使用[Streamlit](https://streamlit.io/)，我们可以提供一个易于构建的前端来运行和测试所有这些，并快速迭代我们的设计。

本教程假定您已安装了Python3.9+和以下软件包：

- llama-index
- streamlit

在基本层面上，我们的目标是从文档中提取术语和定义，然后提供一种方法，让用户查询该术语知识库和定义。本教程将介绍Llama Index和Streamlit的功能，并希望为常见问题提供一些有趣的解决方案。

本教程的最终版本可以在[这里](https://github.com/logan-markewich/llama_index_starter_pack)找到，并且可以在[Huggingface Spaces](https://huggingface.co/spaces/llamaindex/llama_index_term_definition_demo)上提供一个在线托管演示。

## 上传文本

第一步是给用户提供上传文档的方式。让我们使用Streamlit编写一些代码来提供这个界面！使用以下代码并使用`streamlit run app.py`启动应用程序。

```python
import streamlit as st

st.title("🦙 Llama Index Term Extractor 🦙")

document_text = st.text_area("Or enter raw text")
if st.button("Extract Terms and Definitions") and document_text:
    with st.spinner("Extracting..."):
        extracted_terms = document text  # this is a placeholder!
    st.write(extracted_terms)
```

太简单了！但是您会注意到应用程序还没有做任何有用的事情。要使用llama_index，我们还需要设置OpenAI LLM。LLM有很多可能的设置，所以我们可以让用户找出最好的设置。我们还应该让用户设置提取术语的提示（这也将有助于我们调试什么最好）。

## LLM设置

下一步引入了一些标签到我们的应用程序，将其分隔为提供不同功能的不同窗格。让我们为LLM设置和上传文本创建一个标签：

```python
import os
import streamlit as st

DEFAULT_TERM_STR = (
    "Make a list of terms and definitions that are defined in the context, "
    "with one pair on each line. For example:

term1 - definition1
term2 - definition2
term3 - definition3

"
)

st.title("🦙 Llama Index Term Extractor 🦙")

st.sidebar.title("Settings")
st.sidebar.markdown("## LLM Settings")
llm_prompt = st.sidebar.text_area("LLM Prompt", DEFAULT_TERM_STR)

st.title("Upload Text")
document_text = st.text_area("Or enter raw text")
if st.button("Extract Terms and Definitions") and document_text:
    with st.spinner("Extracting..."):
        extracted_terms = document text  # this is a placeholder!
    st.write(extracted_terms)
```st.title("🦙 Llama Index Term Extractor 🦙")

setup_tab, upload_tab = st.tabs(["设置", "上传/提取术语"])

在setup_tab中：
st.subheader("LLM设置")
api_key = st.text_input("在这里输入您的OpenAI API密钥", type="password")
llm_name = st.selectbox('哪个LLM？', ["text-davinci-003", "gpt-3.5-turbo", "gpt-4"])
model_temperature = st.slider("LLM温度", min_value=0.0, max_value=1.0, step=0.1)
term_extract_str = st.text_area("用于提取术语和定义的查询。", value=DEFAULT_TERM_STR)

在upload_tab中：
st.subheader("提取和查询定义")
document_text = st.text_area("或输入原始文本")
如果st.button("提取术语和定义")和document_text：
    with st.spinner("提取中..."):
        extracted_terms = document text  # 这是一个占位符！
    st.write(extracted_terms)

从llama_index导入Document，GPTListIndex，LLMPredictor，ServiceContext，PromptHelper，load_index_from_storage

def get_llm（llm_name，model_temperature，api_key，max_tokens = 256）：
    os.environ['OPENAI_API_KEY'] = api_key
    if llm_name == "text-davinci-003":
        return OpenAI（temperature = model_temperature，model_name = llm_name，max_tokens = max_tokens）
    else：
        return ChatOpenAI（temperature = model_temperature，model_name = llm_name，max_tokens = max_tokens）

def extract_terms（documents，term_extract_str，llm_name，model_temperature，api_key）：
    llm = get_llm（llm_name，model_temperature，api_key，max_tokens = 1024）

    service_context = ServiceContext.from_defaults（llm_predictor = LLMPredictor（llm = llm），
                                                   prompt_helper = PromptHelper（max_input_size = 4096，
                                                                              max_chunk_overlap = 20，现在，使用新的函数，我们终于可以提取我们的术语了！

```python
...
with upload_tab:
    st.subheader("提取和查询定义")
    document_text = st.text_area("或输入原始文本")
    if st.button("提取术语和定义") and document_text:
        with st.spinner("提取中..."):
            extracted_terms = extract_terms([Document(document_text)],
                                            term_extract_str, llm_name,
                                            model_temperature, api_key)
        st.write(extracted_terms)
```

现在发生了很多事情，让我们花一点时间来回顾一下发生了什么。

`get_llm()`根据设置选项卡中的用户配置实例化LLM。根据模型名称，我们需要使用适当的类（`OpenAI`与`ChatOpenAI`）。

`extract_terms()`是所有好东西发生的地方。首先，我们使用`max_tokens = 1024`调用`get_llm()`，因为我们不想在提取术语和定义时过多限制模型（如果未设置，默认值为256）。然后，我们定义我们的`ServiceContext`对象，将`num_output`与我们的`max_tokens`值对齐，并将块大小设置为不大于输出。当文档由Llama Index索引时，如果它们很大，它们将被分割成块（也称为节点），并且`chunk_size_limit`设置了这些块的最大大小。

接下来，我们创建一个临时列表索引并传入我们的服务上下文。列表索引将读取索引中的每一个文本，这对于提取术语非常完美。最后，我们使用预定义的查询文本使用`response_mode =“tree_summarize”`提取术语。此响应模式将从底部开始生成摘要树，其中每个父级概括其子级。最后，返回树的顶部，其中包含所有提取的术语和定义。最后，我们做一些小的后处理。我们假设模型按照说明执行，并在每行放置一个术语/定义对。如果缺少`Term：`或`Definition：`标签，我们将跳过它。然后，我们将其转换为字典以便于存储！

## 保存提取的术语

既然我们可以提取术语，我们需要将它们放在某个地方，以便以后可以查询它们。 `GPTVectorStoreIndex`应该是现在的完美选择！但是此外，我们的应用程序还应该跟踪哪些术语被插入索引，以便以后可以检查它们。使用`st.session_state`，我们可以将当前的术语列表存储在每个用户的会话字典中！

首先，让我们添加一个功能来初始化全局向量索引，以及另一个函数来插入提取的术语。

```python
...
如果'st.session_state'中没有'all_terms'，则st.session_state['all_terms'] = DEFAULT_TERMS
...

def insert_terms(terms_to_definition):
    对于terms_to_definition中的term和definition，
    doc = Document(f"Term: {term}\nDefinition: {definition}")
    st.session_state['llama_index'].insert(doc)

@st.cache_resource
def initialize_index(llm_name, model_temperature, api_key):
    """Create the GPTSQLStructStoreIndex object."""
    llm = get_llm(llm_name, model_temperature, api_key)

    service_context = ServiceContext.from_defaults(llm_predictor=LLMPredictor(llm=llm))

    index = GPTVectorStoreIndex([], service_context=service_context)

    return index

...

with upload_tab:
    st.subheader("提取和查询定义")
    如果st.button("初始化索引并重置术语"):
        st.session_state['llama_index'] = initialize_index(llm_name, model_temperature, api_key)
        st.session_state['all_terms'] = {}

    如果'st.session_state'中有"llama_index"：
        st.markdown("上传文档的图像/屏幕截图，或手动输入文本。")
        document_text = st.text_area("或输入原始文本")
        如果st.button("提取术语和定义")并且（uploaded_file或document_text）：
            st.session_state['terms'] = {}
            terms_docs = {}
            with st.spinner("提取中..."):
                terms_docs.update(extract_terms([Document(document_text)], term_extract_str, llm_name, model_temperature, api_key))
            st.session_state['terms'].update(terms_docs)

        如果'st.session_state'中有"terms"并且st.session_state["terms"]：
            st.markdown("提取的术语")
            st.json(st.session_state['terms'])

            如果st.button("插入术语？")：现在你真的开始利用streamlit的强大功能了！让我们从上传选项卡下的代码开始。我们添加了一个按钮来初始化向量索引，并将其存储在全局streamlit状态字典中，同时重置当前提取的术语。然后，从输入文本中提取术语后，我们将其存储在全局状态中，并给用户一个机会在插入之前查看它们。如果按下插入按钮，则我们调用插入术语函数，更新我们的全局跟踪插入术语，并从会话状态中删除最近提取的术语。

## 查询提取的术语/定义

提取并保存了术语和定义，我们如何使用它们？用户又如何记住以前保存的内容？？我们可以简单地在应用程序中添加一些更多的选项卡来处理这些功能。

...
setup_tab，terms_tab，upload_tab，query_tab = st.tabs（
    ["设置"，"所有术语"，"上传/提取术语"，"查询术语"]
）
...
with terms_tab：
    with terms_tab：
    st.subheader（"当前提取的术语和定义"）
    st.json（st.session_state ["all_terms"]）
...
with query_tab：
    st.subheader（"查询术语/定义！"）
    st.markdown（
        （
            "LLM将尝试回答您的查询，并使用您插入的术语/定义来增强它的答案。"
            "如果索引中没有该术语，它将使用其内部知识回答查询。"
        ）
    ）
    如果st.button（"初始化索引并重置术语"，key ="init_index_2"）：
        st.session_state ["llama_index"] = initialize_index（
            llm_name，model_temperature，api_key
        ）
        st.session_state ["all_terms"] = {}

    如果"llama_index"在st.session_state中：
        query_text = st.text_input（"查询术语或定义："）
        如果query_text：
            query_text = query_text + "\n如果您找不到答案，请根据您的最佳知识回答查询。"
            with st.spinner（"生成答案..."）：
                response = st.session_state ["llama_index"]。query（
                    query_text，similarity_top_k = 5，response_mode ="compact"
                ）
            st.markdown（str（response））
```

虽然这主要是基本的，但要注意一些重要的事情：

- 我们的初始化但这个按钮与我们的其他按钮具有相同的文本。Streamlit会对此抱怨，因此我们提供了一个唯一的键。
- 查询中添加了一些附加文本！这是为了尝试弥补索引没有答案的情况。
- 在我们的索引查询中，我们指定了两个选项：
  - `similarity_top_k=5`意味着索引将获取与查询最接近的5个最匹配的术语/定义。
  - `response_mode="compact"`意味着将尽可能多的文本从5个匹配的术语/定义中用于每个LLM调用。如果没有这个，索引将至少对LLM进行5次调用，这可能会减慢用户的速度。

## 干预测试

嗯，实际上我希望你一直在测试。但现在，让我们尝试一次完整的测试。

1. 刷新应用程序
2. 输入您的LLM设置
3. 转到查询选项卡
4. 询问以下内容：“什么是bunnyhug？”
5. 应用程序应该给出一些无意义的回复。如果你不知道，bunnyhug是加拿大大平原人用来描述连帽衫的另一个词！
6. 让我们将这个定义添加到应用程序中。打开上传选项卡，输入以下文本：“Bunnyhug是一个常用术语，用来描述连帽衫。这个术语是由加拿大大平原人使用的。”
7. 单击提取按钮。几分钟后，应用程序应该显示正确提取的术语/定义。单击插入术语按钮以保存！
8. 如果我们打开术语选项卡，刚刚提取的术语和定义应该会显示
9. 回到查询选项卡，尝试问什么是bunnyhug。现在，答案应该是正确的！

## 改进#1 - 创建一个起点索引

随着我们的基本应用程序正常工作，建立一个有用的索引可能会感觉很累。如果我们给用户提供一些起点来展示应用程序的查询功能怎么办？我们可以做到！首先，让我们对我们的应用程序做一些小的改变，以便我们在每次上传后将索引保存到磁盘：

```python
def insert_terms(terms_to_definition):
    for term, definition in terms_to_definition.items():
        doc = Document(f"Term: {term}\nDefinition: {definition}")
        st.session_state['llama_index'].insert(doc)
    # TEMPORARY - save to disk
    st.session_state['llama_index'].storage_context.persist()
```

现在，我们需要一些文档来提取！本项目的存储库使用了有关纽约市的维基百科页面，您可以在[这里](https://github.com/jerryjliu/llama_index/blob/main/examples/test_wiki/data/nyc_text.txt)找到文本。

如果您将文本粘贴到上传选项卡中并运行它（可能需要一些时间），我们可以插入提取的te从langchain.chains.prompt_selec中，确保也将提取的术语的文本复制到记事本或类似的文件中，然后再插入索引！我们马上就需要它们了。插入后，删除我们用来将索引保存到磁盘的代码行。现在已经保存了起始索引，我们可以修改我们的“initialize_index”函数，看起来像这样：

```python
@st.cache_resource
def initialize_index(llm_name, model_temperature, api_key):
    """Create the GPTSQLStructStoreIndex object."""
    llm = get_llm(llm_name, model_temperature, api_key)

    service_context = ServiceContext.from_defaults(llm_predictor=LLMPredictor(llm=llm))

    index = load_index_from_storage(service_context=service_context)

    return index
```

你记得把那个巨大的提取术语列表保存在记事本里吗？现在，当我们的应用程序初始化时，我们希望将索引中的默认术语传递给我们的全局术语状态：

```python
...
if "all_terms" not in st.session_state:
    st.session_state["all_terms"] = DEFAULT_TERMS
...
```

在以前重置“all_terms”值的任何地方重复以上内容。

## 改进#2 -（细化）更好的提示

现在，如果你玩一下这个应用程序，你可能会注意到它不再遵循我们的提示！记住，我们向我们的“query_str”变量添加了，如果找不到术语/定义，就按照它的最佳知识回答。但是现在，如果你尝试查询随机术语（比如bunnyhug！），它可能会遵循这些指令，也可能不会遵循这些指令。

这是由于“细化”答案的概念。由于我们在前5个匹配结果中查询，有时所有结果都不适合单个提示！OpenAI模型通常具有4097个令牌的最大输入大小。因此，Llama Index通过将匹配结果分割为适合提示的块来解决此问题。 Llama Index从第一个API调用获得初始答案后，将下一块发送到API，并且使用先前的答案向模型请求细化该答案。

因此，细化过程似乎正在破坏我们的结果！不要将额外的指令附加到“query_str”中，而是删除它，Llama Index将允许我们提供自己的自定义提示！现在让我们使用[默认提示]（https://github.com/jerryjliu/llama_index/blob/main/llama_index/prompts/default_prompts.py）和[聊天特定提示]（https://github.com/jerryjliu/llama_index/blob/main/llama_index/prompts/chat_prompts.py）作为指南创建它们。使用一个新的文件“constants.py”，让我们创建一些新的查询模板：

```python
from langchain.chains.prompt_selec这看起来像很多代码，但其实并不复杂！如果你看过默认提示，你可能已经注意到有默认提示，也有针对聊天模型的提示。继续这一趋势，我们也为我们的自定义提示做同样的事情。然后，使用提示选择器，我们可以组合我们的自定义提示，以便在不同的模型中使用。我们可以将这两个提示合并为一个对象。如果使用的LLM是聊天模型（ChatGPT，GPT-4），则使用聊天提示。否则，使用正常的提示模板。

另外要注意的是，我们只定义了一个QA模板。在聊天模型中，这将被转换为一条“人类”消息。

因此，现在我们可以将这些提示导入我们的应用程序，并在查询期间使用它们。

如果你再多尝试一些查询，希望你会注意到现在的响应更加遵循我们的指示！

## 改进#3 - 图像支持

Llama index也支持图像！使用Llama Index，我们可以上传文档（论文，信件等）的图像，Llama Index处理提取文本。我们可以利用这一点，也允许用户上传他们文档的图像，并从中提取术语和定义。

如果你得到一个关于PIL的导入错误，首先使用`pip install Pillow`安装它。

```python
from PIL import Image
from llama_index.readers.file.base import DEFAULT_FILE_EXTRACTOR, ImageParser

@st.cache_resource
def get_file_extractor():
    image_parser = ImageParser(keep_image=True, parse_text=True)
    file_extractor = DEFAULT_FILE_EXTRACTOR
    file_extractor.update(
        {
            ".jpg": image_parser,
            ".png": image_parser,
            ".jpeg": image_parser,
        }
    )

    return file_extractor

file_extractor = get_file_extractor()
...
with upload_tab:
    st.subheader("Extract and Query Definitions")
    if st.button("Initialize Index and Reset Terms", key="init_index_1"):
        st.session_state["llama_index"] = initialize_index(
            llm_name, model_temperature, api_key
        )
        st.session_state["all_terms"] = DEFAULT_TERMS

    if "llama_index" in st.session_state:
        st.markdown(
            "Either upload an image/screenshot of a document, or enter the text manually."
        )
        uploaded_file = st.file_uploader(
            "上传文档的图像/屏幕截图："
        )在这篇教程中，我们涵盖了大量信息，同时解决了一些常见的问题和问题：

- 使用不同的索引用于不同的用例（列表vs.向量索引）
- 使用Streamlit的“session_sta”存储全局状态值本教程的最终版本可以在[这里](https://github.com/logan-markewich/llama_index_starter_pack)找到，并且可以在[Huggingface Spaces](https://huggingface.co/spaces/llamaindex/llama_index_term_definition_demo)上访问一个在线演示。使用Llama Index自定义内部提示和从图像中读取文本。