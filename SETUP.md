# Pinecone Memory Setup

用于首次接入 OpenClaw + Pinecone。保持最小步骤，可跑通即可。

## 1. 前置条件

- Node.js 18+
- 已有 Pinecone 账号
- 准备好 `PINECONE_API_KEY`

## 2. 安装 SDK

```bash
npm install @pinecone-database/pinecone
```

## 3. 设置 API Key

Windows PowerShell:

```powershell
$env:PINECONE_API_KEY="你的Key"
```

macOS/Linux:

```bash
export PINECONE_API_KEY="你的Key"
```

## 4. 创建 Index（Integrated Embedding + Dense）

```javascript
import { Pinecone } from "@pinecone-database/pinecone";

const pc = new Pinecone({ apiKey: process.env.PINECONE_API_KEY });

const indexName = "openclaw-memory";

const exists = (await pc.listIndexes()).indexes?.some((x) => x.name === indexName);
if (!exists) {
  await pc.createIndexForModel({
    name: indexName,
    cloud: "aws",
    region: "us-east-1",
    embed: {
      model: "multilingual-e5-large",
      fieldMap: { text: "chunk_text" }
    }
  });
  console.log("Index created:", indexName);
} else {
  console.log("Index already exists:", indexName);
}
```

## 5. 接入后必须有的 3 个调用

创建完 index 后，至少需要打通下面三类调用，才能形成完整闭环。

### 5.1 sync（导入本地记忆）

用途：把本地 MEMORY.md 和 memory/*.md 切片后写入 Pinecone。

接口签名（建议）：

```ts
type SyncInput = {
  indexName: string;
  namespace: string;
  sourcePaths: string[]; // 例如: ["MEMORY.md", "memory/*.md"]
  chunkSize?: number;
  overlap?: number;
  dryRun?: boolean;
};

type SyncOutput = {
  scannedFiles: number;
  totalChunks: number;
  upserted: number;
  skipped: number; // 由 hash 未变化导致
  failed: number;
  durationMs: number;
};
```

预期行为：

- 使用结构化 _id（如 fileHash#chunkNumber）。
- 写入字段必须包含 chunk_text，并附带 metadata（source、created_at 等）。
- 支持增量（文件 hash 未变化时跳过）。

### 5.2 query（语义搜索验证）

用途：验证检索链路是否可用，以及返回结果是否与问题相关。

接口签名（建议）：

```ts
type QueryInput = {
  indexName: string;
  namespace: string;
  text: string;
  topK?: number;
  filter?: Record<string, unknown>;
  rerank?: boolean;
};

type QueryHit = {
  id: string;
  score: number;
  chunk_text: string;
  metadata: Record<string, unknown>;
};

type QueryOutput = {
  hits: QueryHit[];
  tookMs: number;
};
```

预期行为：

- 默认 topK=5 或 10。
- 优先返回 chunk_text + 关键 metadata，便于展示与溯源。
- 如果开启 rerank，返回最终重排后的命中顺序。

### 5.3 stats（数据量确认）

用途：确认 sync 后数据是否真的写入，以及 namespace 的向量数量。

接口签名（建议）：

```ts
type StatsInput = {
  indexName: string;
  namespace?: string;
};

type StatsOutput = {
  totalVectorCount: number;
  namespaceVectorCount?: number;
  dimension?: number;
  metric?: string;
};
```

预期行为：

- 至少输出 totalVectorCount。
- 指定 namespace 时输出该 namespace 的 vector_count。
- 与 sync 输出的 upserted 做数量级核对（允许最终一致性短暂延迟）。

## 6. 最小验收清单

按顺序跑完以下 3 项即视为接入成功：

1. 执行 sync：upserted > 0。
2. 执行 query：hits 非空且内容相关。
3. 执行 stats：vector_count 与预期接近。

## 7. 测试完成后清理测试数据

如果当前 index 只用于联调测试，建议在验收后立即清理，避免污染正式检索结果。

### 7.1 仅清理测试 namespace（推荐）

前提：测试数据写在独立 namespace（例如 test、dev）。

```javascript
import { Pinecone } from "@pinecone-database/pinecone";

const pc = new Pinecone({ apiKey: process.env.PINECONE_API_KEY });
const index = pc.Index("openclaw-memory");

// 删除该 namespace 下全部记录
await index.namespace("test").deleteAll();
console.log("Deleted all records in namespace: test");
```

预期行为：

- 仅删除 test namespace 的向量。
- default 或生产 namespace 不受影响。

### 7.2 删除整张测试 index（仅纯测试环境）

前提：该 index 没有任何正式数据。

```javascript
import { Pinecone } from "@pinecone-database/pinecone";

const pc = new Pinecone({ apiKey: process.env.PINECONE_API_KEY });
await pc.deleteIndex("openclaw-memory-test");
console.log("Deleted test index: openclaw-memory-test");
```

预期行为：

- 整张测试 index 被删除，向量与配置一并清空。
- 只对纯测试 index 执行，避免误删正式库。

## 8. 常见问题

- `Unauthorized`: API Key 错误或未生效。
- `region/cloud` 相关错误: Starter 计划通常只支持 `aws/us-east-1`。
- 写后查不到: Pinecone 最终一致性，稍等后重试。
