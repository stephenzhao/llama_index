本指南描述了每个索引如何使用图表工作。

一些术语：
- **节点**：对应于文档中的一块文本。LlamaIndex接受Document对象，并在内部将它们解析/分块为Node对象。
- **响应合成**：我们的模块，可以根据检索到的节点合成响应。您可以在这里看到如何[指定不同的响应模式](setting-response-mode)。

## 列表索引

列表索引只是将节点存储为顺序链。

！[](/_static/indices/list.png)

### 查询

在查询时间，如果没有指定其他查询参数，LlamaIndex只是将列表中的所有节点加载到我们的响应合成模块中。

！[](/_static/indices/list_query.png)

列表索引确实提供了许多查询列表索引的方法，从基于嵌入的查询，将获取前k个邻居，或者添加关键字过滤器，如下所示：

！[](/_static/indices/list_filter_query.png)

## 向量存储索引

向量存储索引将每个节点及其相应的嵌入存储在[向量存储](vector-store-index)中。

！[](/_static/indices/vector_store.png)

### 查询

查询向量存储索引涉及获取前k个最相似的节点，并将这些节点传递到我们的响应合成模块中。

！[](/_static/indices/vector_store_query.png)

## 树索引

树索引从一组节点（在此树中成为叶节点）构建一个层次结构树。

！[](/_static/indices/tree.png)

### 查询

查询树索引涉及从根节点遍历到叶节点。默认情况下（`child_branch_factor = 1`），查询给定父节点时选择一个子节点。如果`child_branch_factor = 2`，查询每级选择两个子节点。

！[](/_static/indices/tree_query.png)

## 关键字表索引

关键字表索引从每个节点中提取关键字，并建立从每个关键字到相应节点的映射。

！[](/_static/indices/keyword.png)

### 查询

在查询时，我们从查询中提取相关关键字，并将这些关键字与预先提取的节点关键字进行匹配，以获取相应的节点。提取的节点被传递到我们的响应合成模块中。

！[](/_static/indices/keyword_query.png)