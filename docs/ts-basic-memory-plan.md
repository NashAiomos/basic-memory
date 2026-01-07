# TypeScript Basic Memory 实现计划

## 项目概述

使用 TypeScript 实现一个简化版的 Basic Memory MCP 服务器，提供本地优先的知识管理功能。

**核心理念**：
- 文件是真相来源（Markdown 文件存储知识）
- 语义化标记（观察 `[category]` 和关系 `[[Entity]]`）
- 通过 MCP 协议与 LLM 交互

---

## 技术栈

| 组件 | 技术选择 | 说明 |
|------|----------|------|
| MCP 服务器 | `@modelcontextprotocol/server` | 官方 TypeScript SDK |
| 运行时 | Node.js 20+ | 支持原生 ES modules |
| 类型验证 | Zod | MCP SDK 依赖 |
| 文件操作 | `fs/promises` | 异步文件 I/O |
| Markdown 解析 | `gray-matter` | 解析 frontmatter |
| 搜索 | `minisearch` 或 `flexsearch` | 轻量级全文搜索 |
| 数据库（可选） | `better-sqlite3` | 本地索引加速 |

---

## 目录结构

```
ts-basic-memory/
├── src/
│   ├── index.ts              # 入口，启动 MCP 服务器
│   ├── server.ts             # MCP 服务器配置
│   ├── tools/                # MCP 工具实现
│   │   ├── write-note.ts
│   │   ├── read-note.ts
│   │   ├── search-notes.ts
│   │   ├── build-context.ts
│   │   └── list-directory.ts
│   ├── services/             # 业务逻辑
│   │   ├── file-service.ts   # 文件读写
│   │   ├── entity-service.ts # 实体 CRUD
│   │   ├── search-service.ts # 搜索功能
│   │   └── link-resolver.ts  # 解析 [[WikiLinks]]
│   ├── markdown/             # Markdown 处理
│   │   ├── parser.ts         # 解析观察和关系
│   │   ├── formatter.ts      # 生成 Markdown
│   │   └── frontmatter.ts    # Frontmatter 处理
│   ├── models/               # 数据模型
│   │   ├── entity.ts
│   │   ├── observation.ts
│   │   └── relation.ts
│   └── utils/
│       ├── permalink.ts      # 生成 permalink
│       └── path.ts           # 路径处理
├── data/                     # 知识库存储目录
├── package.json
├── tsconfig.json
└── README.md
```

---

## 核心数据模型

### Entity（实体）

```typescript
interface Entity {
  id: string;                  // UUID
  title: string;               // 标题
  filePath: string;            // 相对文件路径
  permalink: string;           // 唯一标识符
  entityType: string;          // note, decision, spec 等
  content: string;             // Markdown 内容
  observations: Observation[]; // 观察列表
  relations: Relation[];       // 关系列表
  tags: string[];              // 标签
  createdAt: Date;
  updatedAt: Date;
}
```

### Observation（观察）

```typescript
interface Observation {
  category: string;   // decision, idea, fact, technique 等
  content: string;    // 观察内容
  tags: string[];     // 从内容中提取的 #tags
}
```

### Relation（关系）

```typescript
interface Relation {
  relationType: string;  // relates_to, implements, requires 等
  targetTitle: string;   // 目标实体标题
  targetId?: string;     // 已解析的目标实体 ID
}
```

---

## Markdown 格式规范

### 文件结构

```markdown
---
title: 会议记录
type: note
permalink: meetings/2024-meeting
tags: [meeting, planning]
created: 2024-01-15T10:00:00Z
modified: 2024-01-15T10:30:00Z
---

# 会议记录

## 背景
讨论了项目架构设计。

## Observations
- [decision] 使用 PostgreSQL 作为主数据库 #database
- [rationale] 需要 JSONB 支持和事务 #technical
- [action] 下周完成数据库设计 #todo

## Relations
- implements [[系统架构]]
- requires [[用户认证模块]]
- relates_to [[API 设计]]
```

### 解析规则

**观察**：
```
- [category] 内容文本 #tag1 #tag2
```

**关系**：
```
- relation_type [[目标实体标题]]
```
或行内引用：
```
这个功能依赖于 [[其他模块]]
```

---

## MCP 工具设计

### 1. write_note - 创建/更新笔记

```typescript
const writeNoteTool = {
  name: "write_note",
  description: "创建或更新一个笔记",
  inputSchema: z.object({
    title: z.string().describe("笔记标题"),
    content: z.string().describe("Markdown 内容，包含观察和关系"),
    folder: z.string().default("notes").describe("存储文件夹"),
    tags: z.array(z.string()).optional().describe("标签列表"),
    noteType: z.string().default("note").describe("笔记类型"),
  }),
};

// 实现逻辑
async function writeNote(params: WriteNoteParams): Promise<WriteNoteResult> {
  // 1. 生成 permalink
  const permalink = generatePermalink(params.folder, params.title);

  // 2. 解析内容，提取观察和关系
  const { observations, relations } = parseMarkdown(params.content);

  // 3. 构建 frontmatter
  const frontmatter = {
    title: params.title,
    type: params.noteType,
    permalink,
    tags: params.tags,
    created: new Date().toISOString(),
    modified: new Date().toISOString(),
  };

  // 4. 生成完整 Markdown
  const markdown = formatMarkdown(frontmatter, params.content);

  // 5. 写入文件
  const filePath = `${permalink}.md`;
  await writeFile(filePath, markdown);

  // 6. 更新搜索索引
  await indexEntity({ permalink, title, observations, relations });

  // 7. 返回结果
  return { filePath, permalink, observations, relations };
}
```

### 2. read_note - 读取笔记

```typescript
const readNoteTool = {
  name: "read_note",
  description: "读取一个笔记的内容",
  inputSchema: z.object({
    identifier: z.string().describe("笔记标题、permalink 或 memory:// URL"),
  }),
};

async function readNote(params: ReadNoteParams): Promise<string> {
  // 1. 解析标识符
  const entity = await resolveIdentifier(params.identifier);

  // 2. 读取文件
  const content = await readFile(entity.filePath);

  // 3. 返回 Markdown 内容
  return content;
}
```

### 3. search_notes - 搜索笔记

```typescript
const searchNotesTool = {
  name: "search_notes",
  description: "在知识库中搜索笔记",
  inputSchema: z.object({
    query: z.string().describe("搜索关键词"),
    limit: z.number().default(10).describe("返回结果数量"),
  }),
};

async function searchNotes(params: SearchParams): Promise<SearchResult[]> {
  // 使用 MiniSearch 进行全文搜索
  const results = searchIndex.search(params.query, { limit: params.limit });

  return results.map(r => ({
    title: r.title,
    permalink: r.permalink,
    excerpt: r.excerpt,
    score: r.score,
  }));
}
```

### 4. build_context - 构建上下文

```typescript
const buildContextTool = {
  name: "build_context",
  description: "从知识图谱中构建相关上下文",
  inputSchema: z.object({
    url: z.string().describe("memory:// URL 或 permalink"),
    depth: z.number().default(1).describe("遍历深度"),
  }),
};

async function buildContext(params: BuildContextParams): Promise<string> {
  // 1. 解析起始实体
  const startEntity = await resolveIdentifier(params.url);

  // 2. BFS 遍历关系图
  const visited = new Set<string>();
  const entities: Entity[] = [];

  async function traverse(entityId: string, currentDepth: number) {
    if (currentDepth > params.depth || visited.has(entityId)) return;
    visited.add(entityId);

    const entity = await getEntity(entityId);
    entities.push(entity);

    // 遍历关系
    for (const relation of entity.relations) {
      if (relation.targetId) {
        await traverse(relation.targetId, currentDepth + 1);
      }
    }
  }

  await traverse(startEntity.id, 0);

  // 3. 格式化输出
  return formatContext(entities);
}
```

### 5. list_directory - 列出目录

```typescript
const listDirectoryTool = {
  name: "list_directory",
  description: "列出知识库中的文件和文件夹",
  inputSchema: z.object({
    path: z.string().default("").describe("目录路径"),
    depth: z.number().default(1).describe("递归深度"),
  }),
};
```

---

## 实现阶段

### 阶段 1：基础框架（1-2天）

- [ ] 初始化项目结构
- [ ] 配置 TypeScript 和依赖
- [ ] 实现基础 MCP 服务器
- [ ] 实现 `write_note` 工具（无解析）
- [ ] 实现 `read_note` 工具

**里程碑**：能够通过 MCP 创建和读取简单笔记

### 阶段 2：Markdown 解析（1-2天）

- [ ] 实现 frontmatter 解析（使用 gray-matter）
- [ ] 实现观察解析 `[category] content #tags`
- [ ] 实现关系解析 `[[WikiLink]]`
- [ ] 实现 Markdown 生成器

**里程碑**：能够正确解析和生成语义化 Markdown

### 阶段 3：搜索功能（1天）

- [ ] 集成 MiniSearch
- [ ] 实现索引构建（启动时扫描文件）
- [ ] 实现增量索引更新
- [ ] 实现 `search_notes` 工具

**里程碑**：能够全文搜索笔记

### 阶段 4：知识图谱（1-2天）

- [ ] 实现 permalink 解析和生成
- [ ] 实现 `[[WikiLink]]` 解析器
- [ ] 实现关系图遍历
- [ ] 实现 `build_context` 工具

**里程碑**：能够通过关系遍历知识图谱

### 阶段 5：增强功能（可选）

- [ ] 添加 SQLite 索引（加速大型知识库）
- [ ] 实现 `edit_note` 增量编辑
- [ ] 实现 `recent_activity` 最近活动
- [ ] 添加文件监控（watchman/chokidar）

---

## 关键实现细节

### 1. Markdown 解析器

```typescript
// src/markdown/parser.ts
import matter from 'gray-matter';

interface ParsedMarkdown {
  frontmatter: Record<string, any>;
  content: string;
  observations: Observation[];
  relations: Relation[];
}

export function parseMarkdown(raw: string): ParsedMarkdown {
  // 解析 frontmatter
  const { data: frontmatter, content } = matter(raw);

  // 提取观察
  const observations = extractObservations(content);

  // 提取关系
  const relations = extractRelations(content);

  return { frontmatter, content, observations, relations };
}

function extractObservations(content: string): Observation[] {
  const regex = /^-\s+\[(\w+)\]\s+(.+)$/gm;
  const observations: Observation[] = [];

  let match;
  while ((match = regex.exec(content)) !== null) {
    const [, category, text] = match;
    const tags = extractTags(text);
    const cleanContent = text.replace(/#\w+/g, '').trim();

    observations.push({ category, content: cleanContent, tags });
  }

  return observations;
}

function extractRelations(content: string): Relation[] {
  const relations: Relation[] = [];

  // 专用关系行: - relation_type [[Target]]
  const blockRegex = /^-\s+(\w+)\s+\[\[([^\]]+)\]\]/gm;
  let match;
  while ((match = blockRegex.exec(content)) !== null) {
    relations.push({
      relationType: match[1],
      targetTitle: match[2],
    });
  }

  // 行内引用: 任意 [[Target]]（默认为 relates_to）
  const inlineRegex = /\[\[([^\]]+)\]\]/g;
  while ((match = inlineRegex.exec(content)) !== null) {
    // 避免重复添加已在专用行中的关系
    if (!relations.some(r => r.targetTitle === match[1])) {
      relations.push({
        relationType: 'relates_to',
        targetTitle: match[1],
      });
    }
  }

  return relations;
}

function extractTags(text: string): string[] {
  const tagRegex = /#(\w+)/g;
  const tags: string[] = [];
  let match;
  while ((match = tagRegex.exec(text)) !== null) {
    tags.push(match[1]);
  }
  return tags;
}
```

### 2. MCP 服务器设置

```typescript
// src/server.ts
import { McpServer } from '@modelcontextprotocol/server';
import { StdioServerTransport } from '@modelcontextprotocol/server/stdio';
import { z } from 'zod';

const server = new McpServer({
  name: 'ts-basic-memory',
  version: '0.1.0',
});

// 注册工具
server.tool(
  'write_note',
  'Create or update a note in the knowledge base',
  {
    title: z.string(),
    content: z.string(),
    folder: z.string().default('notes'),
    tags: z.array(z.string()).optional(),
  },
  async (params) => {
    const result = await writeNote(params);
    return {
      content: [{ type: 'text', text: JSON.stringify(result, null, 2) }],
    };
  }
);

server.tool(
  'read_note',
  'Read a note from the knowledge base',
  {
    identifier: z.string(),
  },
  async (params) => {
    const content = await readNote(params);
    return {
      content: [{ type: 'text', text: content }],
    };
  }
);

// ... 其他工具

// 启动服务器
async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
}

main().catch(console.error);
```

### 3. 搜索索引

```typescript
// src/services/search-service.ts
import MiniSearch from 'minisearch';

const searchIndex = new MiniSearch({
  fields: ['title', 'content', 'tags', 'observations'],
  storeFields: ['title', 'permalink', 'excerpt'],
  searchOptions: {
    boost: { title: 2 },
    fuzzy: 0.2,
  },
});

export async function buildIndex(dataDir: string) {
  const files = await glob(`${dataDir}/**/*.md`);

  for (const file of files) {
    const content = await readFile(file, 'utf-8');
    const parsed = parseMarkdown(content);

    searchIndex.add({
      id: parsed.frontmatter.permalink,
      title: parsed.frontmatter.title,
      content: parsed.content,
      tags: parsed.frontmatter.tags?.join(' ') || '',
      observations: parsed.observations.map(o => o.content).join(' '),
      permalink: parsed.frontmatter.permalink,
      excerpt: parsed.content.slice(0, 200),
    });
  }
}

export function search(query: string, limit = 10) {
  return searchIndex.search(query, { limit });
}
```

---

## 配置文件示例

### package.json

```json
{
  "name": "ts-basic-memory",
  "version": "0.1.0",
  "type": "module",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "dev": "tsx watch src/index.ts"
  },
  "dependencies": {
    "@modelcontextprotocol/server": "^2.0.0",
    "zod": "^3.25.0",
    "gray-matter": "^4.0.3",
    "minisearch": "^7.0.0",
    "glob": "^11.0.0",
    "uuid": "^10.0.0"
  },
  "devDependencies": {
    "typescript": "^5.6.0",
    "tsx": "^4.0.0",
    "@types/node": "^22.0.0",
    "@types/uuid": "^10.0.0"
  }
}
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "esModuleInterop": true,
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true
  },
  "include": ["src/**/*"]
}
```

---

## Claude Desktop 配置

```json
{
  "mcpServers": {
    "ts-basic-memory": {
      "command": "node",
      "args": ["/path/to/ts-basic-memory/dist/index.js"],
      "env": {
        "MEMORY_DATA_DIR": "/path/to/your/knowledge-base"
      }
    }
  }
}
```

---

## 与 Python 版本的简化对比

| 功能 | Python 版 | TypeScript 简化版 |
|------|-----------|-------------------|
| 数据库 | SQLite/Postgres | 内存索引（可选 SQLite） |
| API 层 | FastAPI REST | 无（直接 MCP） |
| 多项目 | 支持 | 单项目 |
| 云同步 | 支持 | 不支持 |
| 文件监控 | watchfiles | 可选 chokidar |
| 关系解析 | 后台异步 | 同步解析 |

**保留的核心功能**：
- ✅ Markdown 文件存储
- ✅ Frontmatter 元数据
- ✅ 观察和关系语义
- ✅ WikiLink 解析
- ✅ 全文搜索
- ✅ 知识图谱遍历

---

## 参考资源

- [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)
- [MCP 协议规范](https://modelcontextprotocol.io/docs)
- [gray-matter](https://github.com/jonschlinkert/gray-matter) - Frontmatter 解析
- [MiniSearch](https://github.com/lucaong/minisearch) - 轻量级搜索

---

## 下一步

1. 创建项目目录并初始化
2. 安装依赖
3. 按阶段实现功能
4. 使用 Claude Desktop 测试

需要我帮你开始实现第一阶段吗？
