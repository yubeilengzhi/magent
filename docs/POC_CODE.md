# magent MVP - 代码骨架

> 这是一个可运行的 PoC 骨架，验证核心架构。
> 复制以下文件到新项目目录，按顺序运行。

## 1. 项目结构

```
magent/
├── package.json
├── tsconfig.json
├── src/
│   ├── cli/
│   │   └── index.ts        # CLI 入口
│   ├── core/
│   │   ├── router.ts       # 智能路由
│   │   ├── memory.ts       # 记忆层（本地 JSONL）
│   │   ├── config.ts       # 配置加载
│   │   └── session.ts      # Session 管理
│   ├── providers/
│   │   ├── base.ts         # Provider 接口
│   │   ├── codex.ts        # Codex adapter
│   │   └── claude-code.ts  # Claude Code adapter
│   └── index.ts            # 主入口
└── README.md
```

## 2. package.json

```json
{
  "name": "magent",
  "version": "0.1.0",
  "description": "Cross-tool AI coding assistant",
  "type": "module",
  "bin": {
    "magent": "./dist/cli/index.js"
  },
  "scripts": {
    "build": "tsc",
    "dev": "tsx src/cli/index.ts",
    "start": "node dist/cli/index.js"
  },
  "dependencies": {
    "@openai/codex-sdk": "^0.144.0",
    "commander": "^12.0.0",
    "yaml": "^2.5.0",
    "openai": "^4.60.0"
  },
  "devDependencies": {
    "typescript": "^5.4.0",
    "tsx": "^4.10.0",
    "@types/node": "^22.0.0"
  },
  "engines": {
    "node": ">=20"
  }
}
```

## 3. tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "moduleResolution": "Bundler",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "declaration": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

## 4. src/core/config.ts

```typescript
import fs from 'node:fs/promises';
import path from 'node:path';
import yaml from 'yaml';

export interface MagentConfig {
  version: number;
  user: {
    id: string;
    name?: string;
  };
  providers: Record<string, ProviderConfig>;
  memory: {
    backend: 'local' | 'mem0';
    topK?: number;
    threshold?: number;
  };
  router: {
    model: string;
    baseUrl: string;
    apiKey: string;
  };
  mcp: Record<string, MCPServerConfig>;
}

export interface ProviderConfig {
  enabled: boolean;
  defaultModel: string;
  apiKey?: string;
  baseUrl?: string;
}

export interface MCPServerConfig {
  command: string;
  args: string[];
  env?: Record<string, string>;
}

export async function loadConfig(configPath?: string): Promise<MagentConfig> {
  const path = configPath || `${process.env.HOME}/.magent/config.yaml`;
  
  try {
    const content = await fs.readFile(path, 'utf8');
    return yaml.parse(content) as MagentConfig;
  } catch (e) {
    if ((e as NodeJS.ErrnoException).code === 'ENOENT') {
      // 默认配置
      return {
        version: 1,
        user: { id: 'default' },
        providers: {},
        memory: { backend: 'local' },
        router: {
          model: 'haiku',
          baseUrl: process.env.LLM_BASE_URL || 'http://43.137.15.66:8627/v1',
          apiKey: process.env.LLM_API_KEY || '',
        },
        mcp: {},
      };
    }
    throw e;
  }
}
```

## 5. src/core/memory.ts（本地实现，MVP 用）

```typescript
import fs from 'node:fs/promises';
import path from 'node:path';

export interface MemoryRecord {
  id: string;
  type: 'preference' | 'project' | 'decision' | 'routing' | 'session';
  scope: 'user' | 'project' | 'workspace';
  content: string;
  metadata?: Record<string, any>;
  createdAt: string;
  expiresAt?: string;
  source?: string;
}

export class LocalMemory {
  private basePath: string;
  
  constructor(basePath?: string) {
    this.basePath = basePath || `${process.env.HOME}/.magent/memory`;
  }
  
  async init(): Promise<void> {
    await fs.mkdir(this.basePath, { recursive: true });
  }
  
  async add(record: Omit<MemoryRecord, 'id' | 'createdAt'>): Promise<MemoryRecord> {
    const full: MemoryRecord = {
      ...record,
      id: `mem-${Date.now()}-${Math.random().toString(36).slice(2, 8)}`,
      createdAt: new Date().toISOString(),
    };
    
    const filename = `${record.type}s.jsonl`;
    const filepath = path.join(this.basePath, filename);
    
    await fs.appendFile(filepath, JSON.stringify(full) + '\n');
    return full;
  }
  
  async search(query: string, options?: {
    types?: MemoryRecord['type'][];
    scopes?: MemoryRecord['scope'][];
    topK?: number;
  }): Promise<MemoryRecord[]> {
    const topK = options?.topK || 10;
    const results: Array<MemoryRecord & { score: number }> = [];
    
    // 简单的关键词匹配（v1.0 接入 mem0 替换为向量检索）
    const queryLower = query.toLowerCase();
    const queryWords = queryLower.split(/\s+/);
    
    for (const type of ['preference', 'project', 'decision', 'routing', 'session'] as const) {
      if (options?.types && !options.types.includes(type)) continue;
      
      const filepath = path.join(this.basePath, `${type}s.jsonl`);
      try {
        const content = await fs.readFile(filepath, 'utf8');
        const lines = content.trim().split('\n').filter(Boolean);
        
        for (const line of lines) {
          const record = JSON.parse(line) as MemoryRecord;
          
          // 计算 score
          const contentLower = record.content.toLowerCase();
          let score = 0;
          for (const word of queryWords) {
            if (contentLower.includes(word)) score += 1;
          }
          
          // 时间衰减（越新权重越高）
          const ageDays = (Date.now() - new Date(record.createdAt).getTime()) / (1000 * 60 * 60 * 24);
          score = score / (1 + ageDays * 0.1);
          
          if (score > 0) {
            results.push({ ...record, score });
          }
        }
      } catch (e) {
        // 文件不存在，跳过
      }
    }
    
    // 按 score 排序，取 topK
    results.sort((a, b) => b.score - a.score);
    return results.slice(0, topK).map(({ score, ...rest }) => rest);
  }
  
  async getAll(): Promise<MemoryRecord[]> {
    const all: MemoryRecord[] = [];
    for (const type of ['preference', 'project', 'decision', 'routing', 'session'] as const) {
      const filepath = path.join(this.basePath, `${type}s.jsonl`);
      try {
        const content = await fs.readFile(filepath, 'utf8');
        const lines = content.trim().split('\n').filter(Boolean);
        for (const line of lines) {
          all.push(JSON.parse(line));
        }
      } catch (e) {
        // 跳过
      }
    }
    return all;
  }
}
```

## 6. src/core/router.ts（核心智能路由）

```typescript
import OpenAI from 'openai';
import type { MagentConfig } from './config.js';

export interface RouteDecision {
  provider: string;
  model: string;
  reason: string;
  confidence: number;
  fallback?: string;
}

export interface Task {
  description: string;
  type?: string;
}

export interface Context {
  files?: string[];
  estimatedSize?: number;
  timeConstraint?: 'fast' | 'normal' | 'thorough';
}

export class Router {
  private client: OpenAI;
  private model: string;
  
  constructor(config: MagentConfig) {
    this.model = config.router.model;
    this.client = new OpenAI({
      baseURL: config.router.baseUrl,
      apiKey: config.router.apiKey,
    });
  }
  
  async route(task: Task, context: Context): Promise<RouteDecision> {
    const providers = Object.entries(/* get providers */);
    
    const prompt = this.buildPrompt(task, context, providers);
    
    const response = await this.client.chat.completions.create({
      model: this.model,
      messages: [
        { role: 'system', content: '你是 AI 工具路由器。根据任务和上下文，选择最合适的 provider 和 model。' },
        { role: 'user', content: prompt },
      ],
      response_format: { type: 'json_object' },
    });
    
    return JSON.parse(response.choices[0].message.content);
  }
  
  private buildPrompt(task: Task, context: Context, providers: any[]): string {
    return `## 可用 Providers
${providers.map(p => `
- ${p.name}:
  - 默认模型: ${p.defaultModel}
  - 能力: ${p.capabilities?.join(', ') || 'unknown'}
`).join('\n')}

## 当前任务
${task.description}

## 上下文
- Files: ${context.files?.join(', ') ?? 'none'}
- Size: ${context.estimatedSize ?? 'unknown'} tokens
- Constraint: ${context.timeConstraint ?? 'normal'}

## 输出（仅 JSON）
{
  "provider": "name",
  "model": "model-name",
  "reason": "explanation",
  "confidence": 0.0-1.0,
  "fallback": "backup-provider"
}`;
  }
}
```

## 7. src/providers/base.ts（接口定义）

```typescript
import type { Task, Context } from '../core/router.js';

export interface ProviderSession {
  id: string;
  provider: string;
  model: string;
  
  events: AsyncIterable<SessionEvent>;
  send(message: string): Promise<void>;
  abort(): Promise<void>;
  
  status: 'starting' | 'running' | 'paused' | 'completed' | 'failed';
  toolCalls: ToolCall[];
  totalTokens: number;
  
  startedAt: Date;
  completedAt?: Date;
}

export interface SessionEvent {
  type: 'thinking' | 'tool-call' | 'tool-result' | 'message' | 'error' | 'done';
  content: any;
  timestamp: Date;
}

export interface ToolCall {
  name: string;
  args: any;
  result?: any;
}

export interface ProviderAdapter {
  name: string;
  capabilities: string[];
  
  startSession(opts: {
    task: Task;
    context: Context;
    model?: string;
    cwd?: string;
    systemPrompt?: string;
  }): Promise<ProviderSession>;
  
  resumeSession(id: string, opts: {
    cwd?: string;
    systemPrompt?: string;
  }): Promise<ProviderSession>;
}
```

## 8. src/providers/codex.ts（Codex adapter）

```typescript
import { Codex } from '@openai/codex-sdk';
import type { ProviderAdapter, ProviderSession, SessionEvent } from './base.js';

export class CodexAdapter implements ProviderAdapter {
  name = 'codex';
  capabilities = ['file-ops', 'long-context', 'reasoning', 'fast'];
  
  private apiKey?: string;
  
  setApiKey(key: string) {
    this.apiKey = key;
  }
  
  async startSession(opts: any): Promise<CodexSession> {
    const codex = new Codex({
      env: {
        ...process.env,
        OPENAI_API_KEY: this.apiKey,
        CODEX_INTERNAL_ORIGINATOR_OVERRIDE: 'magent',
      },
    });
    
    const thread = codex.startThread({
      model: opts.model,
      workingDirectory: opts.cwd,
      sandboxMode: 'workspace-write',
    });
    
    return new CodexSession(thread, opts.systemPrompt);
  }
  
  async resumeSession(id: string, opts: any): Promise<CodexSession> {
    // 类似的实现
    const codex = new Codex({ /* ... */ });
    const thread = codex.resumeThread(id, {
      workingDirectory: opts.cwd,
    });
    return new CodexSession(thread, opts.systemPrompt);
  }
}

export class CodexSession implements ProviderSession {
  id: string = '';
  provider = 'codex';
  model: string = '';
  status: any = 'starting';
  toolCalls: any[] = [];
  totalTokens = 0;
  startedAt = new Date();
  completedAt?: Date;
  
  private thread: any;
  private systemPrompt?: string;
  private currentInput: string = '';
  
  constructor(thread: any, systemPrompt?: string) {
    this.thread = thread;
    this.systemPrompt = systemPrompt;
  }
  
  async *events(): AsyncIterable<SessionEvent> {
    const stream = await this.thread.runStreamed(this.currentInput);
    for await (const event of stream.events) {
      yield this.transformEvent(event);
    }
  }
  
  async send(message: string): Promise<void> {
    this.currentInput = message;
  }
  
  async abort(): Promise<void> {
    // Codex SDK 没有显式 abort
  }
  
  private transformEvent(event: any): SessionEvent {
    // 把 Codex 事件转换成统一格式
    if (event.type === 'item.completed') {
      if (event.item?.type === 'agent_message') {
        return {
          type: 'message',
          content: event.item.text,
          timestamp: new Date(),
        };
      }
      if (event.item?.type === 'command_execution') {
        return {
          type: 'tool-call',
          content: {
            name: 'bash',
            args: { command: event.item.command },
            result: event.item.aggregated_output,
          },
          timestamp: new Date(),
        };
      }
    }
    if (event.type === 'turn.completed') {
      return { type: 'done', content: null, timestamp: new Date() };
    }
    return {
      type: 'thinking',
      content: JSON.stringify(event),
      timestamp: new Date(),
    };
  }
}
```

## 9. src/providers/claude-code.ts（Claude Code adapter）

```typescript
import { spawn } from 'node:child_process';
import type { ProviderAdapter, ProviderSession, SessionEvent } from './base.js';
import { EventEmitter } from 'node:events';

export class ClaudeCodeAdapter implements ProviderAdapter {
  name = 'claude-code';
  capabilities = ['file-ops', 'long-context', 'deep-reasoning', 'plan-mode'];
  
  async startSession(opts: any): Promise<ClaudeCodeSession> {
    return new ClaudeCodeSession(opts);
  }
  
  async resumeSession(id: string, opts: any): Promise<ClaudeCodeSession> {
    const session = new ClaudeCodeSession(opts);
    session.id = id;
    return session;
  }
}

export class ClaudeCodeSession extends EventEmitter implements ProviderSession {
  id: string = '';
  provider = 'claude-code';
  model: string = '';
  status: any = 'starting';
  toolCalls: any[] = [];
  totalTokens = 0;
  startedAt = new Date();
  completedAt?: Date;
  
  private proc?: any;
  private currentInput: string = '';
  private output: string = '';
  
  constructor(private opts: any) {}
  
  async *events(): AsyncIterable<SessionEvent> {
    // 用 child_process 启动 claude CLI
    const args = [
      '--print',
      '--output-format', 'stream-json',
      '--verbose',
      '--model', this.opts.model || 'sonnet',
      '--append-system-prompt', this.opts.systemPrompt || '',
    ];
    
    if (this.opts.cwd) {
      args.push('--cwd', this.opts.cwd);
    }
    
    this.proc = spawn('claude', args, {
      cwd: this.opts.cwd || process.cwd(),
      env: process.env,
    });
    
    // 喂入输入
    this.proc.stdin.write(this.currentInput);
    this.proc.stdin.end();
    
    // 流式输出
    let buffer = '';
    for await (const chunk of this.proc.stdout) {
      buffer += chunk.toString();
      const lines = buffer.split('\n');
      buffer = lines.pop() || '';
      
      for (const line of lines) {
        try {
          const json = JSON.parse(line);
          yield this.transformEvent(json);
        } catch (e) {
          // 非 JSON 输出
        }
      }
    }
  }
  
  async send(message: string): Promise<void> {
    this.currentInput = message;
  }
  
  async abort(): Promise<void> {
    this.proc?.kill();
  }
  
  private transformEvent(json: any): SessionEvent {
    // claude --output-format stream-json 格式
    if (json.type === 'assistant') {
      return {
        type: 'message',
        content: json.message?.content?.[0]?.text || '',
        timestamp: new Date(),
      };
    }
    if (json.type === 'tool_use') {
      return {
        type: 'tool-call',
        content: {
          name: json.name,
          args: json.input,
        },
        timestamp: new Date(),
      };
    }
    if (json.type === 'tool_result') {
      return {
        type: 'tool-result',
        content: {
          name: json.name,
          result: json.content,
        },
        timestamp: new Date(),
      };
    }
    if (json.type === 'result') {
      return { type: 'done', content: json, timestamp: new Date() };
    }
    return {
      type: 'thinking',
      content: JSON.stringify(json),
      timestamp: new Date(),
    };
  }
}
```

## 10. src/cli/index.ts（CLI 入口）

```typescript
#!/usr/bin/env node
import { Command } from 'commander';
import { loadConfig } from '../core/config.js';
import { LocalMemory } from '../core/memory.js';
import { Router } from '../core/router.js';
import { CodexAdapter } from '../providers/codex.js';
import { ClaudeCodeAdapter } from '../providers/claude-code.js';

const program = new Command();

program
  .name('magent')
  .description('Cross-tool AI coding assistant')
  .version('0.1.0');

program
  .command('run')
  .description('Run a task with intelligent routing')
  .argument('<task...>', 'Task description')
  .option('-p, --provider <name>', 'Force a specific provider')
  .option('-m, --model <name>', 'Force a specific model')
  .option('-c, --cwd <path>', 'Working directory')
  .option('--no-memory', 'Disable memory injection')
  .option('--router-model <name>', 'Model for routing decision')
  .action(async (taskArgs: string[], options) => {
    const config = await loadConfig();
    const task = taskArgs.join(' ');
    const memory = new LocalMemory();
    await memory.init();
    
    // 1. 检索相关记忆
    let memoryContext = '';
    if (options.memory !== false) {
      const memories = await memory.search(task, { topK: 5 });
      if (memories.length > 0) {
        memoryContext = memories
          .map(m => `[${m.type}] ${m.content}`)
          .join('\n');
      }
    }
    
    // 2. 路由决策
    let provider: string;
    let model: string;
    
    if (options.provider) {
      provider = options.provider;
      model = options.model || config.providers[provider]?.defaultModel || '';
    } else {
      const router = new Router(config);
      const decision = await router.route(
        { description: task },
        { files: [], timeConstraint: 'normal' },
      );
      provider = decision.provider;
      model = decision.model;
      console.log(`[Router] ${decision.provider} / ${decision.model}`);
      console.log(`[Router] Reason: ${decision.reason}`);
    }
    
    // 3. 启动 provider session
    let adapter;
    if (provider === 'codex') {
      adapter = new CodexAdapter();
      adapter.setApiKey(config.providers.codex?.apiKey || '');
    } else if (provider === 'claude-code') {
      adapter = new ClaudeCodeAdapter();
    } else {
      throw new Error(`Unknown provider: ${provider}`);
    }
    
    const systemPrompt = memoryContext
      ? `你是 AI 助手。用户的相关上下文：\n${memoryContext}\n\n`
      : '';
    
    const session = await adapter.startSession({
      task: { description: task },
      context: {},
      model,
      cwd: options.cwd,
      systemPrompt,
    });
    
    await session.send(task);
    
    // 4. 流式输出
    for await (const event of session.events) {
      if (event.type === 'message') {
        process.stdout.write(event.content);
      } else if (event.type === 'thinking') {
        console.log(`\n[thinking] ${event.content.slice(0, 200)}`);
      } else if (event.type === 'tool-call') {
        console.log(`\n[tool] ${event.content.name}`);
      } else if (event.type === 'done') {
        break;
      }
    }
    
    // 5. 保存到记忆
    await memory.add({
      type: 'session',
      scope: 'user',
      content: `Task: ${task}\nProvider: ${provider}/${model}`,
      source: 'magent',
    });
  });

program
  .command('memory')
  .description('Manage memories')
  .argument('[query]', 'Search query')
  .option('--add <text>', 'Add a memory')
  .action(async (query, options) => {
    const memory = new LocalMemory();
    await memory.init();
    
    if (options.add) {
      await memory.add({
        type: 'preference',
        scope: 'user',
        content: options.add,
        source: 'user',
      });
      console.log('Added:', options.add);
    } else if (query) {
      const results = await memory.search(query);
      for (const r of results) {
        console.log(`[${r.type}] ${r.content}`);
      }
    } else {
      const all = await memory.getAll();
      console.log(`Total: ${all.length} memories`);
      for (const r of all) {
        console.log(`[${r.type}] ${r.content}`);
      }
    }
  });

program
  .command('config')
  .description('Show configuration')
  .action(async () => {
    const config = await loadConfig();
    console.log(JSON.stringify(config, null, 2));
  });

program.parse();
```

## 11. 第一个 demo：让它跑起来

```bash
# 1. 安装
cd magent
npm install

# 2. 配置
mkdir -p ~/.magent/memory
cat > ~/.magent/config.yaml << 'EOF'
version: 1
user:
  id: dustking
providers:
  codex:
    enabled: true
    defaultModel: qwen3.7-plus
    apiKey: sk-xxx
    baseUrl: http://43.137.15.66:8627/v1
memory:
  backend: local
router:
  model: haiku
  baseUrl: http://43.137.15.66:8627/v1
  apiKey: sk-xxx
EOF

# 3. 运行
npm run dev -- run "重构这个文件"

# 4. 测试记忆
npm run dev -- memory "用户偏好"
npm run dev -- memory --add "用户喜欢 TypeScript strict mode"
```

## 12. 下一步

- [ ] 接入 mem0 替换 LocalMemory
- [ ] 集成 ECC 的 278 个 skills
- [ ] 实现 tree-structured session history
- [ ] MCP 共享中心
- [ ] Web UI
- [ ] npm publish
