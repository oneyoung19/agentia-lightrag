# 知识图谱的文档边界

LightRAG 检索作用域技术备忘：为什么知识图谱按文档过滤在语义上不成立，以及三条实现路径（API 包装层 / 源码改造 doc_ids / workspace 物理隔离）的成本对比。

- 文档链接: https://claude.ai/code/artifact/39517882-8867-413f-9dc0-4ae5516e76f2
- 日期: 2026-09-09

## 方案 C 的成本在哪呢？我理解的是 假如我们上传文件（文件按照会话设置 workspace 进而入库的时候 chunks和 graph都有 workspace id），然后在检索的时候按照 workspace_id 不就可以了么？（这个思路似乎改源码或者直接用 api 都行吧？）

确认了两点关键事实（用你们这套部署实际在用的存储后端来说明，更具体）：

- 你的 .env 没设 KV_STORAGE/VECTOR_STORAGE/GRAPH_STORAGE，走的是默认值——启动日志也印证了：JsonKVStorage / NanoVectorDBStorage / NetworkXStorage，全是文件型后端。
- lightrag/lightrag.py:418 里 workspace 是 LightRAG 这个 dataclass 的一个字段，构造时赋值一次，然后在 lightrag.py:1476-1545 这十几处，原封不动传给 KV/向量/图/文档状态等所有存储后端的构造函数。

为什么"加个 workspace_id 列，查询时过滤"这个直觉在你们的实际部署里不成立

你设想的模型是"一张共享表/图，每行带 workspace_id，查询时 WHERE workspace_id = X"——这套模型对关系型后端（Postgres 等）确实是成立的，那些后端本来就是这么实现 workspace 隔离的。但你们现在用的文件型后端不是这样：图不是"一张表里若干行"，是一整个独立的 GraphML 文件（working_dir/[workspace]/graph_<namespace>.graphml）。换句话说，"workspace" 在你们这套部署里，物理上就是"换一个文件/换一份加载进内存的 NetworkX 图对象"，根本没有"一列 workspace_id 可以 WHERE 过滤"这个东西存在。

哪怕换成 Postgres 之类支持 workspace 列的后端，缺的也不是"加一列"——那本来就有。真正缺的是上面这一整层应用逻辑：LightRAG 这个对象在构造的那一刻就把 workspace 焊死了，之后所有检索代码（operate.py、merge_nodes_and_edges 等）走的都是 self.workspace/global_config["workspace"] 这一个值，没有任何函数签名支持"这次查询临时用另一个 workspace"。要在同一个进程里让多个 workspace 并存、按请求切换，本质上是要把"一个进程 = 一个实例 = 一个 workspace"这个假设，换成"一个进程 = 一个实例注册表 = 按请求路由到正确的实例"——这才是真正的工作量所在，跟"文件用哪种格式""要不要加一列"没关系。

关于"这思路改源码或者用 API 都行吧"——用 API 现在做不到

之前查过：请求里确实有个 LIGHTRAG-WORKSPACE header，但目前只有 /health 接口在读它，/documents、/query 这些真正读写数据的接口完全不理会这个 header，一律操作那个唯一的全局 rag 实例。所以现在不管你在请求里传不传 workspace_id，服务端都不会按它路由——纯 API 这条路现在是走不通的，必须改源码。

改源码要改多少：对着 #3631 的 9-PR 计划看

上次查到的那份设计文档里，拆解出的工作量刚好印证"中～高"这个评级不是随口说的：
- 实例注册表：一个 get_rag() 依赖，根据 workspace 从注册表里取（或懒加载创建）对应的 LightRAG 实例——这是让"多 workspace 共存"成立的核心，目前完全没有
- 缓存隔离是路由开放的硬前提（设计文档里 PR4 明确卡在 PR5 之前）：LLM 响应缓存现在没有按 workspace 严格隔离，直接开放路由会导致跨 workspace 缓存串用
- doc_status / pipeline 隔离必须先完成，写路由才能开——pipeline_status 虽然已经是按 workspace 建 key 的共享字典（这部分是现成的），但完整的写入路径隔离还没做完
- 鉴权/成员模型：设计文档里专门提了一句——"LIGHTRAG-WORKSPACE header 和 Ollama tag 是不可信的路由输入，不是鉴权"，如果不想让任何人靠改个 header 就读到别的 workspace 的数据，还得建一层 workspace 成员/权限模型

所以准确的说法是：存储层"按 workspace 隔离"这件事本来就是现成的，没有成本；缺的是应用层"让一个进程动态服务多个 workspace，并且每个请求都能安全、正确地路由到属于它的那一个"，这一层完全没有，要新建，而且不小。
