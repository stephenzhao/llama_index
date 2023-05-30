本指南旨在帮助您使用具有生产准备就绪的Web应用程序启动模板[Delphic](https://github.com/JSv4/Delphic)来使用LlamaIndex。此处的所有代码示例均可从[Delphic](https://github.com/JSv4/Delphic)存储库获得。

## 我们要建造什么

以下是Delphic出厂功能的快速演示：

https://user-images.githubusercontent.com/5049984/233236432-aa4980b6-a510-42f3-887a-81485c9644e6.mp4

## 架构概述

Delphic利用LlamaIndex python库，让用户可以创建自己的文档集合，然后在响应式前端中查询。

我们选择了一个堆栈，它提供了一个响应式、强大的技术混合，可以（1)协调复杂的python处理任务，同时提供（2)现代、响应式的前端和（3)安全的后端，以便构建额外的功能。

核心库是：

1. [Django](https://www.djangoproject.com/)
2. [Django Channels](https://channels.readthedocs.io/en/stable/)
3. [Django Ninja](https://django-ninja.rest-framework.com/)
4. [Redis](https://redis.io/)
5. [Celery](https://docs.celeryq.dev/en/stable/getting-started/introduction.html)
6. [LlamaIndex](https://gpt-index.readthedocs.io/en/latest/)
7. [Langchain](https://python.langchain.com/en/latest/index.html)
8. [React](https://github.com/facebook/react)
9. Docker & Docker Compose

由于基于超稳定的Django Web框架开发的这种现代堆栈，启动器Delphic应用程序拥有流线型的开发者体验，内置身份验证和用户管理，异步向量存储处理，以及基于Web套接字的查询连接，以实现响应式UI。此外，我们的前端使用TypeScript构建，基于MUI React，以实现响应式和现代的用户界面。

## 系统要求

Celery不能在Windows上工作。它可以通过Windows子系统进行部署，但配置超出了本教程的范围。因此，我们建议您只有在运行Linux或OSX时才遵循本教程。您需要安装Docker和Docker Compose才能部署应用程序。本地开发将需要节点版本管理器（nvm)。

## Django后端

### 项目目录概述

Delphic应用程序具有结构化的后端目录组织，遵循常见的Django项目约定。
从存储库根目录开始，在`./delphic`子文件夹中，主文件夹是：

1. `contrib`：此目录包含对Django内置`contrib`应用程序的自定义修改或添加。Django Ninja是一个用于构建API的Web框架。使用Django和Python 3.7+类型提示，Delphic repo提供了一种简单，直观和表达的方式来定义API端点，利用Python的类型提示自动生成输入验证，序列化和文档。[`./config/api/endpoints.py`](https://github.com/JSv4/Delphic/blob/main/config/api/endpoints.py)文件包含API路由和API端点的逻辑。现在，让我们简要介绍一下`endpoints.py`文件中每个端点的目的：

1. `/heartbeat`：一个简单的GET端点，用于检查API是否正在运行。如果API可以访问，则返回`True`。这对于期望能够查询容器以确保其正在运行的Kubernetes设置非常有用。

2. `/collections/create`：一个POST端点，用于创建新的`Collection`。接受表单参数，如`title`，`description`和`files`列表。为每个文件创建新的`Collection`和`Document`实例，并调度Celery任务来创建索引。

3. `/collections/query` - 一个POST端点，用于使用LLM查询文档集合。接受包含`collection_id`和`query_str`的JSON有效负载，并返回通过查询集合生成的响应。我们实际上没有在聊天GUI中使用此端点（我们使用WebSocket - 请参见下文)，但您可以构建一个应用程序来集成到此REST端点以查询特定的集合。WebSocket是一种通信协议，可在单个长期连接上在客户端和服务器之间实现双向全双工通信。 WebSocket协议设计用于在HTTP和HTTPS（端口80和443)的相同端口上工作，并使用类似的握手过程来建立连接。一旦建立连接，就可以在两个方向上发送“帧”数据，而无需每次重新建立连接，与传统的HTTP请求不同。

使用WebSockets有几个原因，特别是在处理需要花费很长时间加载到内存中但一旦加载就很快运行的代码时：

1. **性能**：WebSockets消除了为每个请求打开和关闭多个连接所带来的开销，从而降低了延迟。
2. **效率**：WebSockets允许实时传输数据，而不需要等待服务器响应。WebSocket可以在不需要轮询的情况下进行通信，从而更有效地利用资源并提高响应能力。

可扩展性：WebSocket可以处理大量同时连接，使其成为需要高并发的应用的理想选择。

在Delphic应用程序的情况下，使用WebSocket是有意义的，因为LLMs可能很昂贵，需要加载到内存中。通过建立WebSocket连接，LLM可以保持加载到内存中，从而允许后续请求快速处理，而无需每次重新加载模型。

ASGI配置文件[`./config/asgi.py`]定义了应用程序应如何处理传入连接，使用Django Channels的`ProtocolTypeRouter`根据其协议类型路由连接。在这种情况下，我们有两种协议类型：“http”和“websocket”。

“http”协议类型使用标准的Django ASGI应用程序来处理HTTP请求，而“websocket”协议类型使用自定义的`TokenAuthMiddleware`来验证WebSocket连接。`TokenAuthMiddleware`中的`URLRouter`定义了一个`CollectionQueryConsumer`的URL模式，该模式负责处理与查询文档集相关的WebSocket连接。

这种配置允许客户端与Delphic应用程序建立WebSocket连接，以有效地使用LLMs查询文档集，而无需为每个请求重新加载模型。

### Websocket处理程序

[`config/api/websockets/queries.py`]中的`CollectionQueryConsumer`类负责处理与查询文档集相关的WebSocket连接。它继承自Django Channels提供的`AsyncWebsocketConsumer`类。

`CollectionQueryConsumer`类有三个主要方法：

1. `connect`：在连接过程中握手时调用WebSocket。
2. `disconnect`：由于任何原因关闭WebSocket时调用。
3. `receive`：服务器从WebSocket接收消息时调用。连接侦听器

`connect`方法负责建立连接，从连接路径中提取集合ID，加载集合模型，并接受连接。

```python
async def connect(self):
    try:
        self.collection_id = extract_connection_id(self.scope["path"])
        self.index = await load_collection_model(self.collection_id)
        await self.accept()

except ValueError as e:
await self.accept()
await self.close(code=4000)
except Exception as e:
pass
```

#### Websocket断开连接侦听器

在这种情况下，`disconnect`方法是空的，因为在WebSocket关闭时没有其他操作要执行。

#### Websocket接收侦听器

`receive`方法负责处理来自WebSocket的传入消息。它接收传入的消息，对其进行解码，然后使用提供的查询查询加载的集合模型。然后，将响应格式化为markdown字符串，并通过WebSocket连接发送回客户端。

```python
async def receive(self, text_data):
    text_data_json = json.loads(text_data)

    if self.index is not None:
        query_str = text_data_json["query"]
        modified_query_str = f"Please return a nicely formatted markdown string to this request:\n\n{query_str}"
        query_engine = self.index.as_query_engine()
        response = query_engine.query(modified_query_str)

        markdown_response = f"## Response\n\n{response}\n\n"
        if response.source_nodes:
            markdown_sources = f"## Sources\n\n{response.get_formatted_sources()}"
        else:
            markdown_sources = ""

        formatted_response = f"{markdown_response}{markdown_sources}"

        await self.send(json.dumps({"response": formatted_response}, indent=4))
    else:
        await self.send(json.dumps({"error": "No index loaded for this connection."}, indent=4))
```

要加载集合模型，使用`load_collection_model`函数，该函数可以在[`delphic/utils/collections.py`](https://github.com/JSv4/Delphic/blob/main/delphic/utils/collections.py)中找到。该函数使用给定的集合ID检索集合对象，检查集合模型的JSON文件是否存在，如果不存在，则创建一个。然后，在使用缓存文件加载`GPTVectorStoreIndex`之前，它会设置`LLMPredictor`和`ServiceContext`。我们选择使用TypeScript，React和Material-UI（MUI)作为Delphic项目的前端，原因有几个。首先，作为最受欢迎的组件库（MUI)和最受欢迎的前端框架（React)，这个选择可以让我们节省大量的时间和精力。其次，TypeScript提供了更多的类型安全性，可以帮助我们更快地开发和调试应用程序。React使这个项目可以访问大量的开发者社区。其次，React目前是一个稳定且普遍受欢迎的框架，它通过其虚拟DOM提供有价值的抽象，同时仍然相对稳定，在我们看来，学习起来也相对容易，这也使它变得更加容易访问。

前端项目结构可以在存储库的[`/frontend`](https://github.com/JSv4/Delphic/tree/main/frontend)目录中找到，与React相关的组件位于`/frontend/src`中。您会注意到`frontend`目录中有一个DockerFile和几个与配置我们的前端Web服务器[nginx](https://www.nginx.com/)相关的文件夹和文件。

`/frontend/src/App.tsx`文件用作应用程序的入口点。它定义了主要组件，例如登录表单，抽屉布局和集合创建模态。根据用户是否已登录并具有身份验证令牌，将有条件地呈现主要组件。

DrawerLayout2组件在`DrawerLayour2.tsx`文件中定义。该组件管理应用程序的布局，并提供导航和主要内容区域。

由于应用程序相对简单，我们可以不使用像Redux这样的复杂状态管理解决方案，而只使用React的useState钩子。

### 从后端抓取集合

登录用户可以访问的集合在DrawerLayout2组件中被检索和显示。该过程可以分解为以下步骤：

1.初始化状态变量：

```tsx
const[collections, setCollections] = useState < CollectionModelSchema[] > ([]);
const[loading, setLoading] = useState(true);
```

在这里，我们初始化两个状态变量：`collections`用于存储集合列表，`loading`用于跟踪是否正在获取集合。

2.使用`fetchCollections（)`函数获取登录用户的集合：

```tsx
const
fetchCollections = async () = > {
try {
const accessToken = localStorage.getItem("accessToken");
if (accessToken) {
const response = await getMyCollections(accessToken);
setCollections(response.data);
}
} catch (error) {
console.error(error);
} finally {
setLoading(false);
}
};
```

`fetchCollections`函数通过使用用户的访问令牌调用`getMyCollections` API函数来检索登录用户的集合。然后，它使用检索到的数据更新`collections`状态，并将`loading`状态设置为`false`以指示获取完成。

### 显示ColleChat View组件

`ChatView`组件位于`frontend/src/chat/ChatView.tsx`，负责处理和显示用户与集合交互的聊天界面。该组件建立WebSocket连接，以实时与服务器通信，发送和接收消息。

`ChatView`组件的主要功能包括：

1. 与服务器建立和管理WebSocket连接。
2. 以聊天格式显示用户和服务器的消息。
3. 处理用户输入以向服务器发送消息。
4. 更新消息状态并在用户发送消息时显示发送状态。Chat Websocket Client

`ChatView`组件中的WebSocket连接用于建立客户端和服务器之间的实时通信。 `ChatView`组件中的WebSocket连接的设置和管理如下：

首先，我们要初始化WebSocket引用：

const websocket = useRef<WebSocket | null>(null);

使用`useRef`创建`websocket`引用，该引用保存将用于通信的WebSocket对象。 `useRef`是React中的一个钩子，允许您创建一个持久的可变引用对象，跨渲染。 当您需要保留对可变对象（例如WebSocket连接)的引用时，它特别有用，而不会引起不必要的重新渲染。

在`ChatView`组件中，需要在组件的生命周期内建立和维护WebSocket连接，并且当连接状态更改时不应触发重新渲染。 通过使用`useRef`，您可以确保将WebSocket连接保留为引用，并且仅在实际状态更改时（例如更新消息或显示错误)才重新渲染组件。

`setupWebsocket`函数负责建立WebSocket连接并设置事件处理程序以处理不同的WebSocket事件。

总的来说，setupWebsocket函数看起来像这样：

```tsx
const setupWebsocket = () => {
  setConnecting(true);
  // 在这里，使用指定的URL创建一个新的WebSocket对象，其中包括所选集合的ID和用户的身份验证令牌。

  websocket.current = new WebSocket(
    `ws://localhost:8000/ws/collections/${selectedCollection.id}/query/?token=${authToken}`
  );

  websocket.current.onopen = (event) => {
    //...
  };

  websocket.current.onmessage = (event) => {
    //...
  };

  websocket.current.onclose = (event) => {
    //...
  };

  websocket.current.onerror = (event) => {
    //...
  };

  return () => {
    websocket.current?.close();
  };
};
```

注意，我们在许多地方基于Web Socket客户端的信息触发更新GUI。

当组件首次打开并尝试建立连接时，`onopen`会触发更新GUI，以显示连接状态。 当收到服务器发送的消息时，`onmessage`会触发更新GUI，以显示消息。 当发生错误时，`onerror`会触发更新GUI，以显示连接状态和错误，例如加载消息，连接到服务器或加载集合时遇到错误。

所有这些功能都使用户能够以非常流畅，低延迟的体验与所选集合进行交互。当WebSocket连接建立时，侦听器被触发。在回调中，组件更新状态以反映连接已建立，任何先前的错误都已清除，并且没有消息正在等待响应：

```tsx
websocket.current.onopen = (event) => {
  setError(false);
  setConnecting(false);
  setAwaitingMessage(false);

  console.log("WebSocket connected:", event);
};
```

当从服务器通过WebSocket连接接收到新消息时，`onmessage`被触发。在回调中，接收到的数据被解析，并且`messages`状态被更新为服务器发送的新消息：

```
websocket.current.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log("WebSocket message received:", data);
  setAwaitingMessage(false);

  if (data.response) {
    // Update the messages state with the new message from the server
    setMessages((prevMessages) => [
      ...prevMessages,
      {
        sender_id: "server",
        message: data.response,
        timestamp: new Date().toLocaleTimeString(),
      },
    ]);
  }
};
```

当WebSocket连接关闭时，`onclose`被触发。在回调中，组件检查特定的关闭代码（`4000`)以显示警告提示，并相应地更新组件状态。它还记录关闭事件：

```tsx
websocket.current.onclose = (event) => {
  if (event.code === 4000) {
    toast.warning(
      "Selected collection's model is unavailable. Was it created properly?"
    );
    setError(true);
    setConnecting(false);
    setAwaitingMessage(false);
  }
  console.log("WebSocket closed:", event);
};
```

最后，当WebSocket连接出现错误时，`onerror`被触发。在回调中，组件更新状态以反映错误，并记录错误事件：

```tsx
    websocket.current.onerror = (event) => {
      setError(true);
      setConnecting(false);
      setAwaitingMessage(false);

      console.error("WebSocket error:", event);
    };
  ```

#### 渲染我们的聊天消息

在`ChatView`组件中，布局是使用CSS样式和Material-UI组件确定的。主布局由具有`flex`显示和列导向的`flexDirection`的容器组成。这确保容器内的内容沿垂直方向排列。

布局中有三个主要部分：

1.聊天消息区域：此部分占用大部分可用空间，显示用户和服务器之间交换的消息列表。它具有溢出-用户输入在`ChatView`组件中接受的是用户在`TextField`中输入的文本消息。该组件处理这些文本输入，并通过WebSocket连接将它们发送到服务器。

##部署

###先决条件

要部署应用程序，您需要安装Docker和Docker Compose。如果您使用的是Ubuntu或其他常见的Linux发行版，DigitalOcean有一个很棒的Docker教程[great Docker tutorial](https://www.digitalocean.com/community/tutorial_collections/how-to-install-and-use-docker)和另一个很棒的教程[Docker Compose](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-compose-on-ubuntu-20-04)您可以遵循。如果这些都不适合您，请尝试[官方docker文档](https://docs.docker.com/engine/install/)。

###构建和部署

该项目基于django-cookiecutter，可以很容易地在VM上部署并配置为为特定域名提供HTTPS流量。然而，配置有些复杂-不是因为这个项目，而是因为配置证书，DNS等是一个相当复杂的主题。

为了本指南的目的，让我们只是在本地运行。也许我们会发布一个关于生产部署的指南。与此同时，请查看[Django Cookiecutter项目文档](https://cookiecutter-django.readthedocs.io/en/latest/deployment-with-docker.html)作为起点。

本指南假定您的目标是将应用程序启动并运行以供使用。如果您想开发，则很可能不会使用--profiles fullstack标志启动compose堆栈，而是使用node开发服务器启动react前端。

要部署，首先克隆repo：

```comm使用应用程序

###设置用户

为了实际使用应用程序（目前，我们打算使其可以与未经认证的用户共享某些模型)，您需要登录。您可以使用超级用户或非超级用户。在任何情况下，都需要有人首先使用控制台创建超级用户：

**为什么要设置Django超级用户？** Django超级用户拥有应用程序中的所有权限，可以管理系统的所有方面，包括创建、修改和删除用户、集合和其他数据。设置超级用户可以让您完全控制和管理应用程序。

**如何创建Django超级用户：**

1.运行以下命令以创建超级用户：

sudo docker-compose -f local.yml run django python manage.py createsuperuser

2.您将被提示提供超级用户的用户名、电子邮件地址和密码。输入所需的信息。

**如何使用Django管理员创建其他用户：**

1.按照部署说明在本地启动您的Delphic应用程序。
2.通过导航到`http://localhost:8000/`访问Django管理界面1.在浏览器中输入“admin”。
2.使用您之前创建的超级用户凭据登录。
3.在“身份验证和授权”部分下，单击“用户”。
4.在右上角单击“添加用户+”按钮。
5.输入新用户的必要信息，如用户名和密码。单击“保存”以创建用户。
6.要为新用户授予其他权限或使其成为超级用户，请在用户列表中单击其用户名，滚动到“权限”部分，并相应地配置其权限。保存您的更改。