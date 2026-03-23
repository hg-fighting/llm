# 项目分析文档

## 项目概述

这是一个基于大语言模型（LLM）的个人知识库助手项目，使用 LangChain 框架构建。项目实现了 RAG（检索增强生成）架构，能够基于 Datawhale 开源项目的 README 文档进行智能问答。用户可以上传自己的知识库文件，系统会自动将其向量化并建立索引，然后通过向量检索与大语言模型相结合，提供准确的答案。

**核心特性：**
- 支持多种 LLM API：OpenAI（GPT-3.5/GPT-4）、文心一言、讯飞星火、智谱 AI
- 支持多种 Embedding：本地 m3e 模型、OpenAI、智谱 AI
- 支持带历史记忆和不带历史记忆的问答模式
- 提供本地 API 服务和 Gradio Web 界面
- 支持多种文档格式：PDF、Markdown、TXT

## 项目架构

项目采用分层架构设计，从底向上分为：

### 1. LLM 层（`llm/`）
- **call_llm.py**：统一的 LLM 调用接口，封装了多种模型的原生 API
  - OpenAI：GPT-3.5-turbo、GPT-4 等系列
  - 文心一言：ERNIE-Bot、ERNIE-Bot-4、ERNIE-Bot-turbo
  - 讯飞星火：Spark-1.5、Spark-2.0（使用 WebSocket）
  - 智谱 AI：chatglm_pro、chatglm_std、chatglm_lite
- **具体实现文件**：
  - `self_llm.py`：自定义 LLM 基类
  - `zhipuai_llm.py`：智谱 AI 的 LangChain 集成
  - `spark_llm.py`：讯飞星火的 WebSocket 实现
  - `wenxin_llm.py`：百度文心的 API 封装

### 2. 数据层（`embedding/`）
- **call_embedding.py**：统一的文本嵌入接口
  - 支持本地 m3e 模型（HuggingFace）
  - 支持 OpenAI Embedding API
  - 支持智谱 AI Embedding API
- **zhipuai_embedding.py**：智谱 AI 的 Embedding 实现

### 3. 数据库层（`database/`）
- **create_db.py**：知识库向量数据库的创建
  - 支持多种文档加载器（PDF、MD、TXT）
  - 使用 RecursiveCharacterTextSplitter 进行文本分割
  - 使用 ChromaDB 作为向量数据库
  - 支持持久化存储
- **test_get_all_repo.py**：从 GitHub 获取 Datawhale 组织的所有仓库 README
- **text_summary_readme.py**：使用 LLM 生成 README 的摘要（过滤无关信息和风控词汇）

### 4. 应用层（`qa_chain/`）
- **model_to_llm.py**：根据模型名称返回对应的 LLM 实例
- **get_vectordb.py**：获取向量数据库实例
- **QA_chain_self.py**：不带历史记录的检索问答链
  - 基于 LangChain 的 RetrievalQA
  - 自定义 Prompt 模板
  - 返回源文档引用
- **Chat_QA_chain_self.py**：带历史记录的检索问答链
  - 基于 ConversationalRetrievalChain
  - 使用 ConversationBufferMemory 管理对话历史
  - 支持自定义历史记录长度

### 5. 服务层（`serve/`）
- **api.py**：FastAPI 接口服务
  - 提供统一的问答 API
  - 支持模型切换和参数配置
- **run_gradio.py**：Gradio Web 界面
  - 交互式聊天界面
  - 支持知识库文件上传和向量化
  - 支持模型和参数动态调整
  - 提供三种对话模式：
    - Chat db with history（带历史记录的知识库问答）
    - Chat db without history（不带历史记录的知识库问答）
    - Chat with llm（纯大模型对话）

## 环境配置

### 系统要求
- **CPU**：Intel 5代处理器或云CPU 2核以上
- **内存**：至少 4 GB
- **Python**：3.9+
- **PyTorch**：2.0+

### 安装步骤

1. **克隆仓库**
```bash
git clone https://github.com/logan-zou/Chat_with_Datawhale_langchain.git
cd Chat_with_Datawhale_langchain
```

2. **创建 Conda 环境**
```bash
conda create -n llm-universe python==3.9.0
conda activate llm-universe
```

3. **安装依赖**
```bash
pip install -r requirements.txt
```

4. **配置环境变量**

创建 `.env` 文件，配置所需 API 密钥：
```env
# OpenAI
OPENAI_API_KEY=your_openai_api_key

# 文心一言
wenxin_api_key=your_wenxin_api_key
wenxin_secret_key=your_wenxin_secret_key

# 讯飞星火
spark_api_key=your_spark_api_key
spark_appid=your_spark_appid
spark_api_secret=your_spark_api_secret

# 智谱 AI
ZHIPUAI_API_KEY=your_zhipuai_api_key
```

## 运行方式

### 方式一：启动 API 服务

**Windows 系统：**
```bash
cd serve
python api.py
```

**Linux 系统：**
```bash
cd serve
uvicorn api:app --reload
```

API 服务默认在 `http://localhost:8000` 启动，提供 `/` POST 接口用于问答。

### 方式二：启动 Gradio Web 界面

```bash
cd serve
python run_gradio.py -model_name='chatglm_std' -embedding_model='m3e' -db_path='./knowledge_db' -persist_path='./vector_db'
```

Gradio 界面会在浏览器中自动打开。

## 使用说明

### Gradio 界面操作流程

1. **上传知识库文件**
   - 点击"请选择知识库目录"上传文件夹
   - 支持的文件格式：.txt, .md, .docx, .pdf

2. **初始化向量数据库**
   - 点击"知识库文件向量化"按钮
   - 等待向量化完成（可能需要较长时间）

3. **配置参数**
   - **llm temperature**：控制模型输出的随机性（0-1）
   - **vector db search top k**：检索返回的文档数量（1-10）
   - **history length**：保留的对话历史轮数（0-5）

4. **选择模型**
   - **Large Language Model**：选择大语言模型
   - **Embedding Model**：选择文本嵌入模型

5. **开始对话**
   - **Chat db with history**：带历史记录的知识库问答
   - **Chat db without history**：不带历史记录的知识库问答
   - **Chat with llm**：纯大模型对话（不使用知识库）

### API 调用示例

```python
import requests

url = "http://localhost:8000/"
data = {
    "prompt": "什么是南瓜书？",
    "model": "chatglm_std",
    "temperature": 0.1,
    "embedding": "m3e",
    "top_k": 3
}

response = requests.post(url, json=data)
print(response.json())
```

## 核心技术实现

### RAG 流程

1. **索引阶段（Indexing）**
   - 加载文档（支持 PDF、MD、TXT）
   - 文本分割（chunk_size=500, chunk_overlap=150）
   - 文本向量化（使用 Embedding 模型）
   - 存储到向量数据库（ChromaDB）

2. **检索阶段（Retrieval）**
   - 用户提问 Query 向量化
   - 在向量数据库中检索最相似的 top-k 个文档片段
   - 返回检索到的文档

3. **生成阶段（Generation）**
   - 将检索到的文档作为上下文 Context
   - 将 Context 和 Question 组合成 Prompt
   - 提交给 LLM 生成答案

### 文本处理

- **PDF 加载**：使用 PyMuPDFLoader
- **Markdown 加载**：使用 UnstructuredMarkdownLoader
- **文本分割**：使用 RecursiveCharacterTextSplitter
- **文本清洗**：过滤 URL 和风控词汇（在生成摘要时）

### 向量检索

- 使用 ChromaDB 作为向量数据库
- 支持持久化存储到本地
- 支持相似度搜索和重排序

## 开发规范

### 代码风格
- 使用 Python 类型提示
- 遵循 PEP 8 编码规范
- 函数和类使用文档字符串说明

### 目录结构规范
```
Chat_with_Datawhale_langchain/
├── database/          # 数据库相关操作
├── embedding/         # 文本嵌入
├── llm/              # 大模型调用封装
├── qa_chain/         # 问答链实现
├── serve/            # 服务层
├── vector_db/        # 向量数据库存储
├── knowledge_db/     # 知识库文件
└── figures/          # 图片资源
```

### API 密钥管理
- 使用 `.env` 文件存储敏感信息
- 使用 `python-dotenv` 加载环境变量
- 不在代码中硬编码 API 密钥

## 常见问题

### 1. 向量化时间过长
- **原因**：文档数量多或文件较大
- **解决**：减少文档数量，或分批向量化

### 2. 模型调用失败
- **原因**：API 密钥未配置或已过期
- **解决**：检查 `.env` 文件中的 API 密钥配置

### 3. 检索效果不佳
- **原因**：chunk_size 或 top_k 参数设置不当
- **解决**：调整参数，增加 chunk_overlap 或 top_k 值

### 4. 内存不足
- **原因**：向量数据库占用内存过大
- **解决**：使用更小的 chunk_size，或使用更高效的向量数据库

## 未来规划

- [ ] 支持用户自主上传并建立个人知识库
- [ ] 从 RAG 架构升级到 Multi-Agent 多智体框架
- [ ] 优化检索函数，提高检索准确性
- [ ] 支持更多文档格式（Word、Excel 等）
- [ ] 添加流式输出支持
- [ ] 优化向量数据库性能

## 技术栈总结

- **语言**：Python 3.9+
- **框架**：LangChain 0.0.292
- **向量数据库**：ChromaDB 0.3.29
- **Web 框架**：FastAPI 0.85.1, Gradio 3.40.1
- **文本处理**：Unstructured 0.9.0
- **PDF 处理**：PyMuPDF 1.23.6
- **其他重要库**：
  - OpenAI 0.27.6
  - zhipuai 1.0.7
  - python-dotenv 1.0.0
  - loguru 0.7.2

## 致谢

本项目基于 Datawhale 的 llm-universe 项目开发，特别感谢散师傅的项目贡献。