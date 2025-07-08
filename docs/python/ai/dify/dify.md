# [Dify](https://docs.dify.ai/zh-hans/introduction)

## 介绍

官方说Dify是一个大语言模型(LLM) 应用开发平台，它融合了**后端即服务**（Backend as Service， BaaS）和 [**LLMOps**](https://docs.dify.ai/zh-hans/learn-more/extended-reading/what-is-llmops) 的理念。

> **BaaS**：
>
> 就是把复杂的后端功能打包成服务，你无需自己编写服务器代码或管理数据库，就能快速构建应用。开发者只需要专注于前端或业务逻辑部分，其他的工作，比如数据库管理、用户认证、文件存储等，都由BaaS平台自动处理。
>
> 举个工作中的例子：之前公司项目中有些统计、过滤需求，是借助腾讯云云函数实现的，企业用户只需要几行代码在云平台上定义一个云函数，后面业务中就可以调用这个云函数实现对应的功能，而不需要考虑云函数的存储、解析执行、搜索、查找等实现。
>
> **LLMOps**：
>
> 个人理解就是将 LLM 应用开发的流程标准。
>
> 可以理解 Dify 就是将 LLM 应用开发的一些基本组件流程开发好了，只需要复用这些组件、流程，从而开发更加关注自己的业务。另外这里说的开发，也不是单纯的编码开发，**有些业务可能只需要简单的配置根本不需要额外编码**，更像是一个低代码平台（尤其是工作流部分）。

## 场景

## 应用

Dify应用依次是：

1. Dify应用启动
2. 接入大模型
3. 构建应用
4. 创建工具流
5. 构建知识库
6. 接入工具

### 源码启动

由于后面需要研究原理，这里选择[使用源代码本地启动](https://docs.dify.ai/zh-hans/getting-started/install-self-hosted/local-source-code)。

```shell
git clone https://github.com/langgenius/dify.git
# 切换当前最新发行版
git checkout -b 1.3.1 1.3.1 

# 1 Docker部署Dify依赖的基础组件：PostgreSQL、Redis、DifySandbox、Dify插件后台进程、Squid、Weaviate
# 	Squid 用作数据代理与缓存层
cd docker
cp middleware.env.example middleware.env
docker compose -f docker-compose.middleware.yaml up -d

# 2 Dify本身是个Web应用，分为前后端，后端服务分为API服务和Worker服务
# 2.1 可能需要先安装 pyenv 这个python版本管理工具
pyenv install 3.12
pyenv global 3.12
exec $SHELL -l
# 	查看安装的python版本和激活的版本
pyenv versions
python3 -V
# 2.2 启动API服务（基于Flask实现）
cd api
cp .env.example .env
awk -v key="$(openssl rand -base64 42)" '/^SECRET_KEY=/ {sub(/=.*/, "=" key)} 1' .env > temp_env && mv temp_env .env
# 	uv 是 Rust实现的Python包管理工具
# 	pip install uv
# 	curl -LsSf https://astral.sh/uv/install.sh | sh
# 	同步前先配置国内镜像源，比如在 pyproject.toml 中添加
# 		[[tool.uv.index]]
# 		url = "https://pypi.tuna.tsinghua.edu.cn/simple"
# 		default = true
#  	如果要使用VSCode启动调试,参考：https://vscode.github.net.cn/docs/python/tutorial-flask，需要额外添加 launch.json，Dify API服务看上去是用的 PyCharm 开发的（可以看到.idea文件夹）但是代码中也提供了 VSCode 调试配置 .vscode/launch.json.example, 拷贝一份到 launch.json
uv sync
# 	用于在数据库中自动创建或更新数据表，比如创建新表、列、索引等
uv run flask db upgrade
uv run flask run --host 0.0.0.0 --port=5001 --debug
# 2.3 启动Worker服务
# 	用于从队列中消费异步任务，例如数据集文件导入和数据集文档更新
# 	Celery 是一个异步任务队列或分布式任务调度系统
uv run celery -A app.celery worker -P gevent -c 1 --loglevel INFO -Q dataset,generation,mail,ops_trace
# 2.4 启动前端（基于React实现）
# 	前端服务端口：3000
#	需要 Node.js v22 (LTS) 和 PNPM v10
cd web
pnpm install --frozen-lockfile
# 	按需修改
cp .env.example .env.local
pnpm build
pnpm start
```

### 接入大模型

如果本地的显卡性能够强可以本地部署大模型。这里使用接入 OpenRouter。

> 运行大模型对显卡性能要求很高。
>
> 比如 DeepSeek 不同参数版本流畅运行需要的硬件参数：
>
> | 模型参数规模      | 典型用途                 | CPU 建议                                 | GPU 建议                                        | 内存建议 (RAM) | 磁盘空间建议      | 适用场景                          |
> | :---------------- | :----------------------- | :--------------------------------------- | :---------------------------------------------- | :------------- | :---------------- | :-------------------------------- |
> | **1.5b (15亿)**   | 小型推理、轻量级任务     | 4核以上 (Intel i5 / AMD Ryzen 5)         | 可选，入门级 GPU (如 NVIDIA GTX 1650, 4GB 显存) | 8GB            | 10GB 以上 SSD     | 小型 NLP 任务、文本生成、简单分类 |
> | **7b (70亿)**     | 中等推理、通用任务       | 6核以上 (Intel i7 / AMD Ryzen 7)         | 中端 GPU (如 NVIDIA RTX 3060, 12GB 显存)        | 16GB           | 20GB 以上 SSD     | 中等规模 NLP、对话系统、文本分析  |
> | **14b (140亿)**   | 中大型推理、复杂任务     | 8核以上 (Intel i9 / AMD Ryzen 9)         | 高端 GPU (如 NVIDIA RTX 3090, 24GB 显存)        | 32GB           | 50GB 以上 SSD     | 复杂 NLP、多轮对话、知识问答      |
> | **32b (320亿)**   | 大型推理、高性能任务     | 12核以上 (Intel Xeon / AMD Threadripper) | 高性能 GPU (如 NVIDIA A100, 40GB 显存)          | 64GB           | 100GB 以上 SSD    | 大规模 NLP、多模态任务、研究用途  |
> | **70b (700亿)**   | 超大规模推理、研究任务   | 16核以上 (服务器级 CPU)                  | 多 GPU 并行 (如 2x NVIDIA A100, 80GB 显存)      | 128GB          | 200GB 以上 SSD    | 超大规模模型、研究、企业级应用    |
> | **671b (6710亿)** | 超大规模训练、企业级任务 | 服务器级 CPU (如 AMD EPYC / Intel Xeon)  | 多 GPU 集群 (如 8x NVIDIA A100, 320GB 显存)     | 256GB 或更高   | 1TB 以上 NVMe SSD | 超大规模训练、企业级 AI 平台      |

Dify 按模型的使用场景将模型分为以下 4 类：

+ 系统推理模型

  这个应该就是指的语言模型（文本生成模型），并不是LLM中定义的推理模型（在输出最终回答之前，模型会先输出一段思维链内容，以提升最终答案的准确性）。

+ Embedding 模型

  将离散的输入数据（如单词、字符或标记）映射到连续的向量空间中。在 NLP 任务中，嵌入层主要用于将词汇表中的每个单词或标记转换为固定维度的数值向量，这些向量可以捕捉单词的语义和上下文信息。

+ Rerank 模型

  Rerank 模型用于增强检索能力，改善 LLM 的搜索结果。会对RAG检索返回的结果进行重新排序。

+ 语音转文字模型

+ 文本转语音模型

模型可以通过插件的方式进行拓展，参考后面章节。

模型可配置参数详解：

TODO。

### 构建应用

Dify 提供了5种类型应用：

+ **聊天助手**：基于 LLM 构建对话式交互的助手
+ **Agent**：能够分解任务、推理思考、调用工具的对话式智能助手
+ **文本生成应用**：面向文本生成类任务的助手，例如撰写故事、文本分类、翻译等
+ **工作流**：适用于自动化、批处理等单轮生成类任务的场景的应用编排方式构建知识库
+ **对话流**：适用于定义等复杂流程的多轮对话场景，具有记忆功能的应用编排方式

感觉描述的有些抽象，可以先从应用配置中观察表象区别，还可以看下源码实现了解内部真正的区别。

|            | 聊天助手                 | Agent                              | 文本生成应用             | 工作流               | 对话流               |
| ---------- | ------------------------ | ---------------------------------- | ------------------------ | -------------------- | -------------------- |
| 支持的配置 | 变量、知识库、元数据过滤 | 变量、知识库、元数据过滤、**工具** | 变量、知识库、元数据过滤 | 流程图、知识库、工具 | 流程图、知识库、工具 |
| 源码实现   |                          |                                    |                          |                      |                      |

> 个人对上面5中应用区别的见解：
>
> Agent：相对于聊天助手，最主要的就是可以通过调用**工具**，完成更加复杂的任务。上面描述“能够分解任务”这个个人理解还是需要通过提示词告诉AI怎么进行任务拆分，LLM现阶段并不具备完全独立自主的任务拆分规划能力。
>
> 也有人将 Agent 定义为：智能体=大模型+知识库+业务系统API+工作流编排，不过和这里的 Agent 应用应该不一样，这种定义更偏向于后面的工作流应用。
>
> 文本生成应用：官方提供了和聊天助手的区别对比，相对于聊天助手，文本生成应用**仅仅保存当次交互上下文**，个人理解每次提交问题都被当作一个全新的问题，不会附带历史记录信息。虽然**无法使用工具但是可以使用Dify的插件**。
>
> 工作流：相对于Agent通过提示词让AI对任务进行拆分的方式，工作流提供了一个**清晰的任务处理流程**（流程图）。另外工作流处理的任务是**单轮的**（从提问开始到响应结束），不受其他提问影响。
>
> 对话流：相对于工作流的单轮任务，**支持多轮对话处理**，比如官方提供的文件翻译场景，首次对话就是一次工作流，但是翻译内容返回之后，用户可能对某些翻译的内容不满意会要求AI进行修改。

### 构建知识库

应用中可以导入知识库。

### 接入工具

应用中可以使用工具。

可以从Dify市场搜索并添加。

TODO: 如何实现自己的工具？

### 开发插件

Dify 可以通过插件拓展模块功能，实现包括**接入模型**、**定义工具**、**定义Agent策略**、**Endpoint 拓展**等功能。

TODO： Dify 官方 SQL 生成器（文本生成应用）借助插件接入了 Minimax 大模型。

#### Agent策略

### 应用案例

网上应用案例很多，这里主要分析一个简单的综合案例（自然语言查询数据库）的实现，并基于此研究内部原理。

这里会使用工作流编排处理流程，使用RAG自动扩展数据库元数据信息提示词，使用LLM生成可执行SQL，使用 MCP SSE 调用 MySQL 服务器执行SQL，最终生成结果报表。

关键步骤：

1. 开始节点：输入查询的自然语言描述

2. RAG节点：Ollama 执行 bge-m3 文本向量模型，然后提交数据库表信息文件创建知识库

   Ollama 支持运行多个模型，通过 API 中模型参数选择使用的模型。

   RAG 其实不是必须的，通过工具调用或MCP也可以让大模型直接去数据库查询数据表信息，这里仅仅是用于后面研究RAG工作原理。

   > Ollama 本地部署 bge-m3 文本向量模型的坑：
   >
   > 1. 由于本地装了Docker，Docker创建了一些虚拟网络，Dify 使用 127.0.0.1 可能连接失败，应该直接指定本地物理网卡IP地址，不要使用 127.0.0.1。
   > 2. Ollama 注册模型时需要将版本信息也加上，比如 `ollama pull bge-m3:567m`，Dify中Ollama注册模型名称 必须是 `bge-m3:567m`，否则会报没有此模型。

3. 可执行SQL生成节点：结合上一步的增强提示词，通过 LLM 生成可执行 SQL

4. MCP 调用节点：安装 MCP SSE 插件，启动 MySQL MCP Server，等待调用

   插件安装貌似是异步任务执行的，需要耐心等待，前端会定时轮询是否安装完成。

   现在 Dify 只通过插件支持了 SSE 协议，暂时不支持 Stdio；所以 MySQL MCP 服务器需要使用支持 SSE 协议的实现。

   MySQL MCP Server 使用的是 [mangooer](https://github.com/mangooer)/[mysql-mcp-server-sse](https://github.com/mangooer/mysql-mcp-server-sse)，需要提前启动，否则配置 MCP Server 时会报连接失败的错误。

   > mysql-mcp-server-sse 工作原理在专门的 MD 文档中说明。
   >
   
5. 查询结果报表节点

## 优劣

对比其他 LLM 开发平台。

TODO。

## 原理

### 上面五种应用的工作原理

#### 工作流流程引擎

### 知识库构建与调用原理

### 工具构建与调用原理

### 插件构建与调用原理

### MCP 支持

通过插件（MCP SSE）实现对 MCP 协议的支持。

