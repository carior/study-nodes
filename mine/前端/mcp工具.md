# MCP 工具开发与发布指南

## 什么是 MCP？

MCP (Model Context Protocol) 是一个标准协议，允许 AI 助手通过标准化的接口与外部工具和服务进行交互。MCP 服务器是一个独立的程序，通过 stdio（标准输入输出）与 AI 客户端通信。

## 开发步骤

### 1. 项目初始化

创建一个新的 npm 包项目：

```bash
mkdir qm-image-parse-mcp
cd qm-image-parse-mcp
npm init -y
```

### 2. 安装依赖

```bash
npm install @modelcontextprotocol/sdk
npm install --save-dev typescript @types/node tsx
```

### 3. 项目结构

```
qm-image-parse-mcp/
├── package.json
├── tsconfig.json
├── src/
│   └── index.ts          # 主入口文件
├── README.md
└── .npmignore
```

### 4. package.json 配置

```json
{
  "name": "qm-image-parse-mcp",
  "version": "1.0.0",
  "description": "图片解析 MCP 服务器，支持阿里云 OSS 和 DashScope AI 识别",
  "main": "dist/index.js",
  "bin": {
    "qm-image-parse-mcp": "./dist/index.js"
  },
  "scripts": {
    "build": "tsc",
    "start": "tsx src/index.ts",
    "prepublishOnly": "npm run build"
  },
  "keywords": [
    "mcp",
    "model-context-protocol",
    "image-parse",
    "oss",
    "dashscope"
  ],
  "author": "your-name",
  "license": "MIT",
  "dependencies": {
    "@modelcontextprotocol/sdk": "^0.5.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "typescript": "^5.0.0",
    "tsx": "^4.0.0"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
```

### 5. tsconfig.json 配置

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "moduleResolution": "node"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### 6. 核心代码实现 (src/index.ts)

```typescript
#!/usr/bin/env node

import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
  ListResourcesRequestSchema,
  ReadResourceRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';
import * as fs from 'fs';
import * as path from 'path';

// 工具定义
const TOOLS = [
  {
    name: 'parse_image',
    description: '解析图片内容，支持本地文件和 OSS URL',
    inputSchema: {
      type: 'object',
      properties: {
        image_path: {
          type: 'string',
          description: '图片路径（本地文件路径或 OSS URL）',
        },
        use_ai: {
          type: 'boolean',
          description: '是否使用 AI 识别（需要配置 OSS）',
          default: false,
        },
      },
      required: ['image_path'],
    },
  },
];

// 资源定义
const RESOURCES = [
  {
    uri: 'image://config',
    name: '配置信息',
    description: '当前 MCP 服务器的配置信息',
    mimeType: 'application/json',
  },
];

class ImageParseServer {
  private server: Server;
  private dashscopeApiKey: string;
  private ossConfig?: {
    accessKeyId: string;
    accessKeySecret: string;
    bucket: string;
    region: string;
  };

  constructor() {
    // 从环境变量读取配置
    this.dashscopeApiKey = process.env.DASHSCOPE_API_KEY || '';
    if (process.env.OSS_ACCESS_KEY_ID) {
      this.ossConfig = {
        accessKeyId: process.env.OSS_ACCESS_KEY_ID,
        accessKeySecret: process.env.OSS_ACCESS_KEY_SECRET || '',
        bucket: process.env.OSS_BUCKET || '',
        region: process.env.OSS_REGION || 'oss-cn-shanghai',
      };
    }

    this.server = new Server(
      {
        name: 'qm-image-parse-mcp',
        version: '1.0.0',
      },
      {
        capabilities: {
          tools: {},
          resources: {},
        },
      }
    );

    this.setupHandlers();
  }

  private setupHandlers() {
    // 列出可用工具
    this.server.setRequestHandler(ListToolsRequestSchema, async () => ({
      tools: TOOLS,
    }));

    // 列出可用资源
    this.server.setRequestHandler(ListResourcesRequestSchema, async () => ({
      resources: RESOURCES,
    }));

    // 读取资源
    this.server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
      if (request.params.uri === 'image://config') {
        return {
          contents: [
            {
              uri: request.params.uri,
              mimeType: 'application/json',
              text: JSON.stringify(
                {
                  dashscopeConfigured: !!this.dashscopeApiKey,
                  ossConfigured: !!this.ossConfig,
                  ossRegion: this.ossConfig?.region,
                },
                null,
                2
              ),
            },
          ],
        };
      }
      throw new Error(`Unknown resource: ${request.params.uri}`);
    });

    // 调用工具
    this.server.setRequestHandler(CallToolRequestSchema, async (request) => {
      const { name, arguments: args } = request.params;

      if (name === 'parse_image') {
        return await this.handleParseImage(args as any);
      }

      throw new Error(`Unknown tool: ${name}`);
    });
  }

  private async handleParseImage(args: {
    image_path: string;
    use_ai?: boolean;
  }): Promise<{
    content: Array<{
      type: string;
      text: string;
    }>;
  }> {
    const { image_path, use_ai = false } = args;

    try {
      // 检查文件是否存在（本地文件）
      if (fs.existsSync(image_path)) {
        const stats = fs.statSync(image_path);
        const ext = path.extname(image_path).toLowerCase();

        if (!['.jpg', '.jpeg', '.png', '.gif', '.webp'].includes(ext)) {
          throw new Error('不支持的文件格式');
        }

        // 如果启用 AI 识别且配置了 OSS
        if (use_ai && this.ossConfig && this.dashscopeApiKey) {
          // 上传到 OSS 并使用 DashScope 识别
          return await this.parseWithAI(image_path);
        } else {
          // 基础图片信息
          return {
            content: [
              {
                type: 'text',
                text: JSON.stringify(
                  {
                    path: image_path,
                    size: stats.size,
                    extension: ext,
                    message: '图片文件已找到，如需 AI 识别请配置 OSS 和 DashScope API Key',
                  },
                  null,
                  2
                ),
              },
            ],
          };
        }
      } else if (image_path.startsWith('http://') || image_path.startsWith('https://')) {
        // 处理 URL
        if (use_ai && this.ossConfig && this.dashscopeApiKey) {
          return await this.parseWithAI(image_path);
        } else {
          return {
            content: [
              {
                type: 'text',
                text: JSON.stringify(
                  {
                    url: image_path,
                    message: '图片 URL 已识别，如需 AI 识别请配置 OSS 和 DashScope API Key',
                  },
                  null,
                  2
                ),
              },
            ],
          };
        }
      } else {
        throw new Error(`文件不存在: ${image_path}`);
      }
    } catch (error: any) {
      return {
        content: [
          {
            type: 'text',
            text: JSON.stringify(
              {
                error: error.message,
              },
              null,
              2
            ),
          },
        ],
      };
    }
  }

  private async parseWithAI(imagePath: string): Promise<{
    content: Array<{
      type: string;
      text: string;
    }>;
  }> {
    // 这里实现 AI 识别逻辑
    // 1. 如果是本地文件，上传到 OSS
    // 2. 调用 DashScope API 进行识别
    // 3. 返回识别结果

    // 示例实现（需要安装相应的 SDK）
    return {
      content: [
        {
          type: 'text',
          text: JSON.stringify(
            {
              message: 'AI 识别功能需要实现 OSS 上传和 DashScope API 调用',
              imagePath,
            },
            null,
            2
          ),
        },
      ],
    };
  }

  async run() {
    const transport = new StdioServerTransport();
    await this.server.connect(transport);
    console.error('qm-image-parse-mcp server running on stdio');
  }
}

// 启动服务器
const server = new ImageParseServer();
server.run().catch(console.error);
```

### 7. README.md 编写

```markdown
# qm-image-parse-mcp

图片解析 MCP 服务器，支持阿里云 OSS 和 DashScope AI 识别。

## 安装

```bash
npm install -g qm-image-parse-mcp
```

## 配置

在 Cursor 的 `.cursor/mcp.json` 文件中添加配置：

```json
{
  "mcpServers": {
    "image-parse": {
      "command": "npx",
      "args": ["-y", "qm-image-parse-mcp"],
      "env": {
        "DASHSCOPE_API_KEY": "你的阿里云API Key",
        "OSS_ACCESS_KEY_ID": "你的OSS AccessKeyId",
        "OSS_ACCESS_KEY_SECRET": "你的OSS AccessKeySecret",
        "OSS_BUCKET": "你的Bucket名称",
        "OSS_REGION": "oss-cn-shanghai"
      }
    }
  }
}
```

## 环境变量说明

- `DASHSCOPE_API_KEY` (必填): 阿里云 DashScope API Key，用于 AI 图片识别
- `OSS_ACCESS_KEY_ID` (可选): OSS AccessKeyId，用于大文件 AI 识别
- `OSS_ACCESS_KEY_SECRET` (可选): OSS AccessKeySecret
- `OSS_BUCKET` (可选): OSS Bucket 名称
- `OSS_REGION` (可选): OSS 区域，默认为 `oss-cn-shanghai`

## 使用方法

配置完成后，重启 Cursor，即可在 AI 对话中使用图片解析功能。

## 开发

```bash
# 安装依赖
npm install

# 开发模式运行
npm run start

# 构建
npm run build
```

## License

MIT
```

### 8. .npmignore 配置

```
src/
tsconfig.json
*.ts
*.ts.map
node_modules/
.env
.env.*
*.log
.DS_Store
```

### 9. 发布到 npm

```bash
# 1. 登录 npm
npm login

# 2. 检查包名是否可用
npm search qm-image-parse-mcp

# 3. 构建项目
npm run build

# 4. 发布（会自动运行 prepublishOnly 脚本）
npm publish

# 5. 发布后验证
npm view qm-image-parse-mcp
```

### 10. 使用方式

用户只需要在 `.cursor/mcp.json` 中添加配置：

```json
{
  "mcpServers": {
    "image-parse": {
      "command": "npx",
      "args": ["-y", "qm-image-parse-mcp"],
      "env": {
        "DASHSCOPE_API_KEY": "你的阿里云API Key",
        "OSS_ACCESS_KEY_ID": "你的OSS AccessKeyId",
        "OSS_ACCESS_KEY_SECRET": "你的OSS AccessKeySecret",
        "OSS_BUCKET": "你的Bucket名称",
        "OSS_REGION": "oss-cn-shanghai"
      }
    }
  }
}
```

## 开发要点

### 1. MCP SDK 使用

- 使用 `@modelcontextprotocol/sdk` 提供的 Server 类
- 通过 `StdioServerTransport` 进行通信
- 实现标准的请求处理器

### 2. 工具定义

工具需要定义：
- `name`: 工具名称
- `description`: 工具描述
- `inputSchema`: 输入参数 schema（JSON Schema 格式）

### 3. 资源定义

资源需要定义：
- `uri`: 资源唯一标识
- `name`: 资源名称
- `description`: 资源描述
- `mimeType`: MIME 类型

### 4. 错误处理

- 使用 try-catch 捕获错误
- 返回格式化的错误信息
- 确保服务器不会崩溃

### 5. 环境变量

- 通过 `process.env` 读取配置
- 提供合理的默认值
- 在文档中说明必填和可选参数

## 测试

在发布前，建议进行本地测试：

```bash
# 1. 本地构建
npm run build

# 2. 本地安装测试
npm link

# 3. 在另一个项目中测试
cd test-project
npm link qm-image-parse-mcp
# 配置 mcp.json 使用本地包
```

## 版本管理

遵循语义化版本控制：
- `1.0.0`: 初始版本
- `1.0.1`: 补丁版本（bug 修复）
- `1.1.0`: 次要版本（新功能）
- `2.0.0`: 主要版本（破坏性变更）

## 常见问题

### Q: 如何调试 MCP 服务器？

A: 可以在代码中使用 `console.error()` 输出调试信息，这些信息会输出到 stderr。

### Q: 如何处理大文件？

A: 对于大文件，建议先上传到 OSS，然后通过 URL 进行处理。

### Q: 如何添加新的工具？

A: 在 `TOOLS` 数组中添加新工具定义，然后在 `CallToolRequestSchema` 处理器中添加对应的处理逻辑。

## 参考资源

- [MCP 官方文档](https://modelcontextprotocol.io/)
- [MCP SDK GitHub](https://github.com/modelcontextprotocol/typescript-sdk)
- 项目中的示例：`.cursor/mcp.json`
