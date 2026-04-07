# MCP 工具快速启动模板

## 最小化示例

### 1. 创建项目

```bash
mkdir my-mcp-tool
cd my-mcp-tool
npm init -y
```

### 2. 安装依赖

```bash
npm install @modelcontextprotocol/sdk
npm install --save-dev typescript @types/node tsx
```

### 3. 最小化 package.json

```json
{
  "name": "my-mcp-tool",
  "version": "1.0.0",
  "description": "我的 MCP 工具",
  "main": "dist/index.js",
  "bin": {
    "my-mcp-tool": "./dist/index.js"
  },
  "scripts": {
    "build": "tsc",
    "start": "tsx src/index.ts"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^0.5.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "typescript": "^5.0.0",
    "tsx": "^4.0.0"
  }
}
```

### 4. 最小化代码 (src/index.ts)

```typescript
#!/usr/bin/env node

import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';

const server = new Server(
  {
    name: 'my-mcp-tool',
    version: '1.0.0',
  },
  {
    capabilities: {
      tools: {},
    },
  }
);

// 定义工具
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: 'hello',
      description: '说你好',
      inputSchema: {
        type: 'object',
        properties: {
          name: {
            type: 'string',
            description: '你的名字',
          },
        },
        required: ['name'],
      },
    },
  ],
}));

// 处理工具调用
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  if (name === 'hello') {
    return {
      content: [
        {
          type: 'text',
          text: `你好，${(args as any).name}！`,
        },
      ],
    };
  }

  throw new Error(`未知工具: ${name}`);
});

// 启动服务器
async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error('MCP 服务器已启动');
}

main().catch(console.error);
```

### 5. tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true
  },
  "include": ["src/**/*"]
}
```

### 6. 测试运行

```bash
npm run start
```

### 7. 配置到 Cursor

在 `.cursor/mcp.json` 中添加：

```json
{
  "mcpServers": {
    "my-tool": {
      "command": "npx",
      "args": ["-y", "my-mcp-tool"]
    }
  }
}
```

### 8. 发布

```bash
npm run build
npm publish
```

## 完整示例：环境变量配置

如果需要环境变量，在 `mcp.json` 中配置：

```json
{
  "mcpServers": {
    "my-tool": {
      "command": "npx",
      "args": ["-y", "my-mcp-tool"],
      "env": {
        "API_KEY": "your-api-key"
      }
    }
  }
}
```

在代码中读取：

```typescript
const apiKey = process.env.API_KEY || '';
```

## 完整示例：资源支持

```typescript
import { ListResourcesRequestSchema, ReadResourceRequestSchema } from '@modelcontextprotocol/sdk/types.js';

// 定义资源
server.setRequestHandler(ListResourcesRequestSchema, async () => ({
  resources: [
    {
      uri: 'config://info',
      name: '配置信息',
      description: '当前配置',
      mimeType: 'application/json',
    },
  ],
}));

// 读取资源
server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
  if (request.params.uri === 'config://info') {
    return {
      contents: [
        {
          uri: request.params.uri,
          mimeType: 'application/json',
          text: JSON.stringify({ apiKey: !!process.env.API_KEY }, null, 2),
        },
      ],
    };
  }
  throw new Error(`未知资源: ${request.params.uri}`);
});
```

## 发布检查清单

- [ ] 更新 `package.json` 中的版本号
- [ ] 编写 `README.md`
- [ ] 添加 `.npmignore`
- [ ] 运行 `npm run build` 确保构建成功
- [ ] 测试本地安装：`npm link` 和 `npm link my-mcp-tool`
- [ ] 检查包名是否可用：`npm search my-mcp-tool`
- [ ] 登录 npm：`npm login`
- [ ] 发布：`npm publish`
