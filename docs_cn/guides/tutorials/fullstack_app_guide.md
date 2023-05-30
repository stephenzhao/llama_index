LlamaIndex是一个python库，这意味着将其与全栈Web应用程序集成将与您习惯的不同。

本指南旨在走完创建用python编写的基本API服务所需的步骤，以及此与TypeScript + React前端如何交互。

此处的所有代码示例均可从[llama_index_starter_pack](https://github.com/logan-markewich/llama_index_starter_pack/tree/main/flask_react)中的flask_react文件夹中获取。

本指南中使用的主要技术如下：

- python3.11
- llama_index
- flask
- typescript
- react

## Flask后端

对于本指南，我们的后端将使用[Flask](https://flask.palletsprojects.com/en/2.2.x/) API服务器与我们的前端代码进行通信。如果您愿意，您也可以轻松将其转换为[FastAPI](https://fastapi.tiangolo.com/)服务器，或您选择的任何其他python服务器库。

使用Flask设置服务器很容易。您导入包，创建应用程序对象，然后创建端点。让我们首先为服务器创建一个基本的骨架：

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def home():
    return "Hello World!"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5601)
```

_flask_demo.py_

如果您运行此文件（`python flask_demo.py`），它将在端口5601上启动服务器。如果您访问`http://localhost:5601/`，您将在浏览器中看到“Hello World！”文本呈现。很好！

下一步是决定我们想要在服务器中包含哪些功能，以及开始使用LlamaIndex。

为了保持简单，我们可以提供的最基本的操作是查询现有索引。使用LlamaIndex中的[paul graham essay](https://github.com/jerryjliu/llama_index/blob/main/examples/paul_graham_essay/data/paul_graham_essay.txt)，创建一个documents文件夹，并将文本文件下载并放置在其中。

### 基本Flask - 处理用户索引查询

现在，让我们编写一些代码来初始化我们的索引：

```python
import os
from llama_index import SimpleDirectoryReader, GPTVectorStoreIndex, StorageContext

# NOTE: for local testing only, do NOT deploy with your key hardcoded
os.environ['OPENAI_API_KEY'] = "your key here"

index = None

def initialize_index():
    global index
    storage_context = StorageContext.from_defaults()
    if os.path.exists(index_dir):
        index = load_index_from_storage(storage_context)
    else:
        documents = SimpleDirectoryReader("./documents").load_data()
```这个函数将初始化我们的索引。如果我们在`main`函数中启动Flask服务器之前调用它，那么我们的索引就可以准备好接受用户查询了！

我们的查询端点将接受带有查询文本作为参数的`GET`请求。这里是完整的端点函数的样子：

```python
from flask import request

@app.route("/query", methods=["GET"])
def query_index():
  global index
  query_text = request.args.get("text", None)
  if query_text is None:
    return "No text found, please include a ?text=blah parameter in the URL", 400
  query_engine = index.as_query_engine()
  response = query_engine.query(query_text)
  return str(response), 200
```

现在，我们为我们的服务器引入了一些新的概念：

- 由函数装饰器定义的新的`/query`端点
- 来自Flask的新导入`request`，用于从请求中获取参数
- 如果缺少`text`参数，则我们返回错误消息和适当的HTML响应代码
- 否则，我们查询索引，并将响应作为字符串返回

您可以在浏览器中测试的完整查询示例可能如下所示：`http://localhost:5601/query?text=what did the author do growing up`（按下回车后，浏览器将将空格转换为“％20”字符）。

一切看起来都很好！我们现在有一个功能完备的API。使用您自己的文档，您可以轻松为任何应用程序提供接口，以调用Flask API并获得查询答案。

### 高级Flask - 处理用户文档上传

看起来很酷，但我们如何进一步推进？如果我们希望允许用户通过上传自己的文档来构建自己的索引，又会怎样？没有恐惧，Flask可以处理所有：muscle:。

要让用户上传文档，我们必须采取一些额外的预防措施。我们不再查询现有的索引，而是索引变得**可变**。如果有许多用户向同一索引添加内容，我们需要考虑如何处理并发性。我们的Flask服务器是多线程的，这意味着多个用户可以同时向服务器发送请求，服务器将同时处理这些请求。

一种选择可能是为每个用户或组创建一个索引，并从S3存储和获取内容。但是对于这个例子，我们假设有一个本地存储的索引，用户可以与之交互。

为了处理并发上传并确保对索引进行顺序插入，我们可以使用`BaseManager` pyt我们将所有的索引操作（初始化，查询，插入）移动到“index_server”的“BaseManager”中，从我们的Flask服务器调用它，以提供使用单独的服务器和锁的顺序访问索引的hon包。这听起来很可怕，但其实并不是那么糟糕！

我们的`index_server.py`将看起来像这样：

```python
import os
from multiprocessing import Lock
from multiprocessing.managers import BaseManager
from llama_index import SimpleDirectoryReader, GPTVectorStoreIndex, Document

# NOTE: for local testing only, do NOT deploy with your key hardcoded
os.environ['OPENAI_API_KEY'] = "your key here"

index = None
lock = Lock()

def initialize_index():
  global index

  with lock:
    # same as before ...
  ...

def query_index(query_text):
  global index
  query_engine = index.as_query_engine()
  response = query_engine.query(query_text)
  return str(response)

if __name__ == "__main__":
    # init the global index
    print("initializing index...")
    initialize_index()

    # setup server
    # NOTE: you might want to handle the password in a less hardcoded way
    manager = BaseManager(('', 5602), b'password')
    manager.register('query_index', query_index)
    server = manager.get_server()

    print("starting server...")
    server.serve_forever()
```

_index_server.py_

因此，我们已经移动了我们的函数，引入了`Lock`对象，以确保对全局索引的顺序访问，在服务器中注册了单个函数，并在端口5602上以密码“password”启动了服务器。

然后，我们可以如下调整我们的flask代码：

```python
from multiprocessing.managers import BaseManager
from flask import Flask, request

# initialize manager connection
# NOTE: you might want to handle the password in a less hardcoded way
manager = BaseManager(('', 5602), b'password')
manager.register('query_index')
manager.connect()

@app.route("/query", methods=["GET"])
def query_index():
  global index
  query_text = request.args.get("text", None)
  if query_text is None:
    return "No text found, please include a ?text=blah parameter in the URL", 400
  response = manager.query_index(query_text)._getvalue()
  return str(response), 200

@app.route("/")
def home():
    return "Hello World!"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5601)

```

_flask_demo.py_

两个主要的变化是连接到我们现有的`BaseManager`服务器并注册函数，以及在`/query`端点中通过管理器调用函数。需要注意的是，`BaseManager`服务器不会按我们期望的那样返回对象。为了将返回值转换为其原始对象，我们调用`_getvalue（）`函数。

如果我们允许用户上传自己的文档，我们可能应该先从文档文件夹中删除保罗·格雷厄姆的文章，因此让我们先做这件事。然后，让我们添加一个上传文件的端点！首先，让我们定义我们的Flask端点函数：

```python
...
manager.register('insert_into_index')
...

@app.route("/uploadFile", methods=["POST"])
def upload_file():
    global manager
    if 'file' not in request.files:
        return "Please send a POST request with a file", 400

    filepath = None
    try:
        uploaded_file = request.files["file"]
        filename = secure_filename(uploaded_file.filename)
        filepath = os.path.join('documents', os.path.basename(filename))
        uploaded_file.save(filepath)

        if request.form.get("filename_as_doc_id", None) is not None:
            manager.insert_into_index(filepath, doc_id=filename)
        else:
            manager.insert_into_index(filepath)
    except Exception as e:
        # cleanup temp file
        if filepath is not None and os.path.exists(filepath):
            os.remove(filepath)
        return "Error: {}".format(str(e)), 500

    # cleanup temp file
    if filepath is not None and os.path.exists(filepath):
        os.remove(filepath)

    return "File inserted!", 200
```

不错！您会注意到我们将文件写入磁盘。如果我们只接受像`txt`文件这样的基本文件格式，我们可以跳过这一步，但是写入磁盘，我们可以利用LlamaIndex的`SimpleDirectoryReader`来处理更复杂的文件格式。可选地，我们还可以使用第二个`POST`参数来使用文件名作为doc_id，或者让LlamaIndex为我们生成一个。一旦我们实现前端，这将更有意义。

对于这些更复杂的请求，我还建议使用[Postman](https://www.postman.com/downloads/?utm_source=postman-home)等工具。有关使用Postman测试我们的端点的示例，请参阅[此项目的存储库](https://github.com/logan-markewich/llama_index_starter_pack/tree/main/flask_react/postman_examples)。

最后，您会注意到我们为管理器添加了一个新函数。让我们在`index_server.py`中实现它：

```python
def insert_into_index(doc_text, doc_id=None):
    global index
    document = SimpleDirectoryReader(input_files=[doc_text]).load_data()[0]
    if doc_id is not None:
        document.doc_id = doc_id

    with lock:
        index.insert(documen很简单！如果我们启动`index_server.py`和`flask_demo.py`两个Python文件，我们就拥有了一个可以处理多个请求以将文档插入向量索引并响应用户查询的Flask API服务器！

为了支持前端的一些功能，我调整了Flask API的一些响应样式，并添加了一些功能来跟踪哪些文档存储在索引中（LlamaIndex目前没有以用户友好的方式支持这一点，但我们可以自行增强它！）。最后，我使用`Flask-cors`Python包为服务器添加了CORS支持。

查看存储库中完整的`flask_demo.py`和`index_server.py`脚本以及`requirements.txt`文件和用于部署的示例`Dockerfile`。

## React前端

通常，React和Typescript是当今用于编写Web应用程序的最流行的库和语言之一。本指南假定您熟悉这些工具的工作原理，否则本指南的长度将增加两倍：smile:。

在[存储库](https://github.com/logan-markewich/llama_index_starter_pack/tree/main/flask_react)中，前端代码位于`react_frontend`文件夹中。

前端的最相关的部分将是`src/apis`文件夹。这是我们调用Flask服务器的地方，支持以下查询：

- `/query` - 对现有索引进行查询
- `/uploadFile` - 将文件上传到Flask服务器以插入索引
- `/getDocuments` - 列出当前文档标题及其文本的一部分

使用这三个查询，我们可以构建一个强大的前端，允许用户上传和跟踪其文件，查询索引，并查看查询响应和有关哪些文本节点用于形成响应的信息。

### fetchDocuments.tsx

此文件包含用于，您猜到了，获取索引中当前文档列表的函数。代码如下：

```typescript
export type Document = {
  id: string;
  text: string;
};

const fetchDocuments = async (): Promise<Document[]> => {
  const response = await fetch("http://localhost:5601/getDocuments", {
    mode: "cors",
  });

  if (!response.ok) {
    return [];
  }

  const documentList = (await response.json()) as Document[];
  return documentList;
};
```

正如您所见，我们向Flask服务器（这里假设在本地主机上运行）。请注意，我们需要包括`mode：'cors'`选项，因为我们正在进行外部请求。

然后，我们检查响应是否正常，如果是，获取响应json并返回它。在这里，响应json是在同一个文件中定义的`Document`对象列表。

### queryIndex.tsx

此文件将用户查询发送到Flask服务器，并获取响应，以及有关我们索引中哪些节点提供响应的详细信息。

```typescript
export type ResponseSources = {
  text: string;
  doc_id: string;
  start: number;
  end: number;
  similarity: number;
};

export type QueryResponse = {
  text: string;
  sources: ResponseSources[];
};

const queryIndex = async (query: string): Promise<QueryResponse> => {
  const queryURL = new URL("http://localhost:5601/query?text=1");
  queryURL.searchParams.append("text", query);

  const response = await fetch(queryURL, { mode: "cors" });
  if (!response.ok) {
    return { text: "Error in query", sources: [] };
  }

  const queryResponse = (await response.json()) as QueryResponse;

  return queryResponse;
};

export default queryIndex;
```

这与`fetchDocuments.tsx`文件类似，主要区别在于我们将查询文本作为URL中的参数包含在内。然后，我们检查响应是否正常，并使用适当的typescript类型返回它。

### insertDocument.tsx

可能最复杂的API调用是上传文档。此函数接受文件对象并使用`FormData`构造`POST`请求。

实际响应文本在应用程序中没有使用，但可以用于提供有关文件上传失败的一些用户反馈。

```typescript
const insertDocument = async (file: File) => {
  const formData = new FormData();
  formData.append("file", file);
  formData.append("filename_as_doc_id", "true");

  const response = await fetch("http://localhost:5601/uploadFile", {
    mode: "cors",
    method: "POST",
    body: formData,
  });

  const responseText = response.text();
  return responseText;
};

export default insertDocument;
```

### 所有其他前端好的东西

这就是前端部分！其余的react前端代码是一些非常基本的react组件，我尽力使它看起来至少有点好看:smile:。

我鼓励你阅读[代码库](https://github.com/logan-markewich/llama_index_starter_pack/tree/main/flask_react/react_frontend)的其余部分，并提交任何改进的PR！

## 结论

本指南涵盖了大量信息，从如何构建Flask服务器到如何使用React来调用API。我希望这有助于你开始使用Flask和React来构建你自己的应用程序！我们从用Python编写的基本的“Hello World”Flask服务器开始，到一个完全可运行的LlamaIndex后端，以及如何将其连接到前端应用程序。

正如您所看到的，我们可以轻松地增强和包装LlamaIndex提供的服务（比如小型外部文档跟踪器），以帮助在前端提供良好的用户体验。

您可以采用这种方式添加许多功能（多索引/用户支持，将对象保存到S3，添加Pinecone向量服务器等）。当您在阅读本文后构建应用程序时，请务必在Discord中分享最终结果！祝你好运！:muscle: