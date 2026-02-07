# 知微多模态 Agent 电商客服中台

## 项目描述

电商场景下咨询意图复杂多样、PDF 商品手册与规则文档信息提取困难、核心业务办理长期依赖人工处理。针对以上痛点，搭建全栈电商智能客服中台。
系统采用 **规则路由优先 + Agent 兜底** 的混合架构，通过意图识别引擎将用户请求精准分流至 **闲聊快答、业务办理、RAG 知识问答、Agent 复杂推理** 四条处理链路。构建从多模态数据解析、多路混合检索到自动量化评测的完整闭环，在 **40 条评测样本** 上达到 **意图准确率 97.5%、检索召回率 0.87、生成质量评分 89.5/100、平均延迟 3.01s**。

---

## 技术栈

| 层级 | 技术选型 |
|------|---------|
| **大模型** | Gemini 3 Flash Preview（意图推理 + 生成 + LLM-as-a-Judge 评测） |
| **Embedding** | 通义 text-embedding-v2（DashScope API，1536 维） |
| **Reranker** | BAAI/bge-reranker-base（本地 CrossEncoder 推理，无需 API） |
| **向量库** | ChromaDB 0.4.24（HNSW + cosine 距离） |
| **全文检索** | Elasticsearch 8.11（BM25 + jieba 分词扩展） |
| **关系数据库** | MySQL 8.0（订单、商品、物流、用户数据） |
| **缓存 & 会话** | Redis 7（对话历史窗口 + RAG 结果缓存） |
| **后端框架** | FastAPI + Uvicorn（SSE 流式输出 + 同步/异步双接口） |
| **前端** | Tailwind CSS 单页应用（聊天界面 + 管理后台） |
| **Agent 框架** | LangChain + ChatOpenAI（ReAct 循环，最多 5 轮工具调用） |
| **文档解析** | PyMuPDF 文本提取 + 视觉大模型 OCR 兜底 |
| **容器化** | Docker Compose 编排 5 个服务（App / MySQL / ChromaDB / ES / Redis） |

---

## 功能特性

**四路意图分流**
- **闲聊快答**：正则模式匹配常见寒暄词，直接返回预置回复，零延迟
- **业务办理**：识别订单查询、取消订单、退款申请、物流追踪、库存查询、商品对比 6 类业务意图，调用结构化工具直接执行
- **RAG 知识问答**：命中运费规则、售后政策、DJI 产品参数等 30+ 关键词时走 RAG 检索链路
- **Agent 复杂推理**：意图模糊或需多步推理时进入 ReAct Agent，自主决策调用工具或检索知识库

**多模态文档处理**
- PDF 解析采用 **文本层优先 + 视觉模型兜底** 双策略，文本量低于 50 字符自动切换为图片 OCR
- 滑动窗口切片：chunk_size=500，overlap=100，保证语义连续性
- 支持 PDF / TXT / 图片上传，统一进入向量库 + ES 双写

**多路混合检索**
- **召回阶段**：ChromaDB 向量检索与 ES 关键词检索并行执行，各取 Top 15
- **粗排阶段**：RRF 倒数排名融合，ES 权重 1.5 × 向量权重 1.0，输出 Top 10
- **精排阶段**：本地 bge-reranker-base CrossEncoder 重排序，输出 Top 3 送入生成

**多轮对话**
- Redis 存储对话历史，窗口大小 10 轮，TTL 3600 秒
- Agent 每次请求注入完整历史消息，支持指代消解和上下文延续

**流式响应**
- SSE 逐 token 推送，前端实时渲染 + 打字机动画
- 图片消息走同步接口，文本消息走流式接口

**自动化评测体系**
- 40 条覆盖全意图类型的评测样本
- 三维度评分：意图准确率（精确匹配）、检索召回率（关键词命中）、生成质量（LLM-as-a-Judge 0-100 打分）
- Web 端一键触发评测，实时展示每条 PASS/FAIL 和各意图维度准确率

---

## 流程图

```
用户输入
   │
   ▼
┌──────────────┐
│  意图识别引擎  │ ← 正则规则 + 关键词匹配
└──────┬───────┘
       │
  ┌────┼────┬────────────┐
  ▼    ▼    ▼            ▼
闲聊  业务办理  RAG问答   Agent推理
快答   │       │         │
  │    ▼       ▼         ▼
  │  工具执行  多路召回    ReAct循环
  │  (MySQL   ├─向量检索  ├─bind_tools
  │   Mock)   ├─ES检索    ├─工具调用
  │    │      ▼          ├─知识检索
  │    │    RRF粗排       │ (最多5轮)
  │    │      ▼          │
  │    │   Reranker精排   │
  │    │      ▼          │
  │    │   LLM生成       │
  │    │      │          │
  └────┴──────┴──────────┘
              │
              ▼
       ┌──────────┐
       │ Redis缓存 │ ← 对话历史 + RAG结果
       └────┬─────┘
            ▼
      SSE流式/同步响应
```

---

## 项目结构模块说明

```
rag_project/
├── main.py              # FastAPI 入口，定义 /chat/stream、/chat/sync、/upload、/eval 等接口
├── config.py            # Pydantic Settings 统一配置，读取 .env 环境变量
├── dialog.py            # 对话管理器，协调图片处理、RAG 检索、Agent 调用
├── agent.py             # ReAct Agent，SystemPrompt + 历史注入 + 工具绑定 + 循环执行
├── intent.py            # 规则意图识别引擎，11 类意图 + FAQ/闲聊/RAG 关键词库
├── skills.py            # 7 个结构化工具定义 + Mock 数据库 + 快速路径路由器
├── rag.py               # RAG 管线：FAQ匹配 → 并行召回 → RRF粗排 → Reranker精排 → LLM生成
├── llms.py              # LLM / Embedding / Reranker 三合一封装，含缓存和重试
├── vectorstore.py       # ChromaDB 向量库操作：去重写入、cosine 检索、集合管理
├── es_client.py         # Elasticsearch 操作：索引创建、jieba 分词、match_phrase + match 混合查询
├── memory.py            # Redis 对话历史管理，滑动窗口 + 自动过期
├── multimodal.py        # 多模态处理器，调用视觉大模型描述/OCR 图片
├── ocr_processor.py     # PDF/TXT/图片文档解析，文本层提取 + 视觉兜底 + 滑动窗口切片
├── evaluation.py        # 评测引擎：意图精确匹配 + 关键词召回 + LLM-as-a-Judge 打分
├── import_kb.py         # 知识库 JSON 导入脚本，双写 ChromaDB + ES
├── import_pdf.py        # PDF 批量导入脚本，OCR 解析后双写
├── clean_kb.py          # 数据清理脚本，重置 ChromaDB / ES / Redis
├── run_eval.py          # 命令行评测脚本，输出完整报告
├── deploy.sh            # 一键部署脚本
├── Dockerfile           # Python 3.10-slim 镜像，安装系统依赖和 pip 包
├── docker-compose.yml   # 5 服务编排：App / MySQL / ChromaDB / ES / Redis
├── requirements.txt     # Python 依赖清单
├── init.sql             # MySQL 建表 DDL：products / orders / logistics / refunds 等 8 张表
├── init_data.sql        # MySQL 初始数据：5 款商品、3 笔订单、物流轨迹
├── .env                 # 环境变量配置
├── frontend/
│   ├── index.html       # 聊天界面：SSE 流式渲染、图片上传预览、会话管理
│   └── admin.html       # 管理后台：评测触发、指标看板、知识库上传
└── test_data/
    ├── eval_cases.json          # 40 条评测用例，覆盖 8 类意图
    ├── knowledge_base.json      # 30 条结构化知识条目（运费/售后/DJI参数等）
    └── *.pdf                    # DJI Mini 4 Pro 用户手册 + 4 份帮助文档
```

---

## 评测结果

| 指标 | 数值 | 说明 |
|------|------|------|
| **意图准确率** | **97.5%** | 40 条样本中 39 条意图识别正确 |
| **检索召回率** | **0.87** | 基于关键词命中率衡量文档召回质量 |
| **生成质量** | **89.5 / 100** | LLM-as-a-Judge 对回答相关性和准确性打分 |
| **平均延迟** | **3.01s** | 含意图识别 + 检索 + 生成全链路耗时 |

各意图维度准确率均达到 80% 以上，RAG 类问题覆盖运费规则、退款流程、DJI 产品参数等 20+ 细分场景。

---

## 快速开始

**1. 克隆项目并配置环境变量**

```bash
cd rag_project
cp .env.example .env   # 按需修改 API Key 和数据库连接
```

**2. 一键部署**

```bash
chmod +x deploy.sh
./deploy.sh
```

部署完成后 5 个容器自动启动：App（8000）、MySQL（3307）、ChromaDB（8001）、ES（9200）、Redis（6380）

**3. 导入知识库**

```bash
# 导入结构化知识条目
docker exec -it rag-app python import_kb.py

# 导入 PDF 文档
docker exec -it rag-app python import_pdf.py
```

**4. 运行评测**

```bash
docker exec -it rag-app python run_eval.py
```

**5. 访问服务**

- 聊天界面：http://localhost:8000
- 管理后台：http://localhost:8000/admin
- 健康检查：http://localhost:8000/health
- API 文档：http://localhost:8000/docs

---

## 优化点

**检索质量提升**
- ES 查询引入 jieba 分词扩展 + match_phrase 短语匹配（boost=3.0）与 match 模糊匹配双通道，minimum_should_match 设为 20% 提高长尾覆盖
- 向量检索设置 similarity > 0.2 阈值过滤低质量结果
- RRF 融合中 ES 权重设为向量的 1.5 倍，利用电商场景关键词匹配优势

**延迟控制**
- 向量检索与 ES 检索通过 ThreadPoolExecutor 并行执行，超时 2 秒兜底
- FAQ 正则极速匹配前置，高频问题跳过整个 RAG 管线
- Redis 缓存 RAG 结果 600 秒 + Chat 结果 300 秒，重复问题毫秒级响应
- LLM 调用内存级缓存，相同 prompt 不重复请求

**文档解析优化**
- PDF 优先提取文本层（零成本），仅当页面文字量 < 50 字符时调用视觉模型
- 渲染分辨率 2x Matrix 保证扫描件 OCR 精度
- 每 5 页主动 gc.collect() 防止大文档内存溢出

**Agent 稳定性**
- ReAct 循环硬性限制 5 轮，防止工具调用死循环
- temperature=0.0 确保工具选择确定性
- SystemPrompt 明确要求 **槽位未填充时反问而非捏造**，减少幻觉
- 工具执行异常统一 try-catch，返回错误信息而非中断对话
