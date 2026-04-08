---
name: pinecone-memory
description: "将 OpenClaw 记忆系统接入 Pinecone 向量数据库，用于语义检索与记忆持久化。Use when: 用户要求同步记忆到 Pinecone、执行语义搜索、为记忆做云端备份、对大规模记忆进行高效检索。"
---

# Pinecone Memory Skill

将 OpenClaw 本地记忆系统与 Pinecone 向量数据库连接，提供可扩展的语义检索能力。

## 目标

- 把本地记忆持久化到向量数据库，支持长期检索与备份。
- 在需要回忆历史上下文时，提供高相关召回结果。
- 通过命名空间和元数据过滤控制性能、成本与安全。

## 阅读顺序

- 首次接入与运行步骤：见 setup.md。
- 测试完成后的清理步骤：见 setup.md 的测试清理章节。
- 本文档用于定义技能触发、检索和数据建模规则。

## 环境要求

- PINECONE_API_KEY 需要向用户获取。
- Node.js 18+。
- 已在 Pinecone 创建 index。

## 索引创建策略

采用方案：集成嵌入（Integrated Embedding）+ Dense Index。

- 直接 upsert 原始文本，由 Pinecone 自动向量化。
- 创建 index 时指定 embed.model 和 embed.field_map。
- field_map 的文本字段名必须与 upsert 记录字段一致（例如 chunk_text）。
- 本技能默认语义检索，不启用 sparse index。

关键约束：

- Starter 计划通常只支持 aws/us-east-1。
- cloud 和 region 创建后不可修改。
- metric 固定使用 cosine（与本技能默认 dense 语义检索保持一致）。

## 数据建模规范

### 记录结构

- _id：唯一标识，推荐结构化 ID，如 document1#chunk1。
- chunk_text：文本字段（集成嵌入时用于自动向量化）。
- metadata：用于过滤和管理，如 document_id、chunk_number、created_at、source。

### 示例记录（可直接用于 upsert）

```json
{
  "_id": "document1#chunk1",
  "chunk_text": "First chunk of the document content...",
  "document_id": "document1",
  "document_title": "Introduction to Vector Databases",
  "chunk_number": 1,
  "document_url": "https://example.com/docs/document1",
  "created_at": "2024-01-15",
  "document_type": "tutorial"
}
```

字段说明：

- _id：结构化主键，便于按前缀批量 list/fetch。
- chunk_text：与 embed.field_map 绑定的文本字段。
- 其余字段作为 metadata，用于过滤、追踪来源和增量更新。

### 元数据建议

- 仅保留高价值过滤字段，避免冗余元数据导致查询和索引变慢。
- metadata key 使用字符串，value 使用 string/number/boolean/string[]。
- 单条记录 metadata 总量控制在 Pinecone 限制内。

## 存储内容

| 类型 | 存储内容 | 写入时机 |
|------|----------|----------|
| 对话记忆 | 每轮对话摘要 + 时间戳 + 标签 | 每 5 轮批量写，重要对话即时写 |
| 项目知识 | 代码和文档按 chunk 切片 | 用户要求索引时，基于文件 hash 增量写 |
| 用户偏好 | 格式偏好、工具习惯 | 用户明确表达或系统观察到时 |
| 错误经验 | 命令失败记录 + 正确做法 | 失败或纠正后即时写 |
| 知识缓存 | Web 搜索结果 + URL | 搜索后即时写，带 TTL 过期 |
| 文件索引 | 上传文件内容摘要 | 收到文件后即时写 |

## 什么时候查（触发策略）

- 用户说“上次”“之前”“你还记得”时：查对话记忆 + 错误经验。
- 用户问代码或项目问题时：查项目知识。
- 准备执行命令前：先查历史错误经验，减少重复踩坑。

## 查完之后怎么用（Agent 行为规则）

1. 先做结果筛选：仅保留与当前问题直接相关的前 3-8 条记忆。
2. 再做冲突处理：若多条记忆冲突，优先近期记录和成功案例。
3. 然后生成回答：把检索结论合并到当前回复，不逐条转储原始命中。
4. 涉及命令执行时：优先采用历史成功命令，并显式规避已知失败写法。
5. 证据不足时：明确说明“不确定”，并向用户追问关键缺失信息。
6. 写回闭环：本轮得到新结论或纠错后，按类型增量写回 Pinecone。

## 什么时候不查

- 闲聊、简单问候、纯计算、当前上下文已有答案时，不触发向量检索，避免延迟和 token 浪费。

## 一致性与写入策略

- Pinecone 为最终一致性系统，写后立即读可能短暂不可见。
- 写入后如需立即检索，使用短暂重试或延迟机制。
- 大规模导入时优先使用 import，常规更新使用 upsert。

## 检索策略

- 默认语义检索：top_k + metadata 过滤。
- 需要更高精度时：增加 rerank。
- 对关键词敏感任务：优先使用 metadata 过滤和本地全文补充，不切换 sparse index。

## 安全

- 索引前自动过滤 API Key、密码、token、.env 内容，防止敏感数据入库。
- 不上传本地明文密钥与私有凭据。
- 对敏感项目可按 namespace 做隔离并限制检索范围。

## 技能执行约定

1. 写入前先做敏感信息过滤。
2. 写入时统一生成结构化 _id 和 metadata。
3. 查询时优先限定 namespace，其次使用 metadata filter。

## 示例配置

```json
{
  "memory": {
    "pinecone": {
      "enabled": true,
      "indexName": "openclaw-memory",
      "namespace": "default",
      "embeddingModel": "multilingual-e5-large",
      "vectorType": "dense",
      "metric": "cosine",
      "useSparseIndex": false,
      "rerankEnabled": true,
      "syncOnWrite": true
    }
  }
}
```

## 参考文档

- https://docs.pinecone.io/guides/get-started/quickstart
- https://docs.pinecone.io/guides/index-data/create-an-index
- https://docs.pinecone.io/guides/index-data/data-modeling
