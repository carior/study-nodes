# Hermes Agent 实测笔记

> 来源：[Hermes Agent 两个月狂揽 4.7 万 GitHub Star，号称"会自我进化的 AI Agent"](https://mp.weixin.qq.com/s/0BEGacHOMC6Ez2_wsVfmcQ)
> 实测时间：2026-04-14
> 原文作者：在一周实测后整理

## 一句话定位

Hermes Agent 是 Nous Research 开源的 AI Agent 框架，核心卖点是**自进化**——它会从你的使用中自动学习，把复杂任务沉淀成可复用的 Skill，越用越懂你。

## 和龙虾（OpenClaw）的核心区别

| 维度 | 龙虾 (OpenClaw) | Hermes Agent |
|------|----------------|--------------|
| 类型 | 连接型选手 | 进化型选手 |
| 强项 | 多平台接入、工具链管理、GUI 上手快 | 自动学习、技能沉淀、CLI 为主 |
| 记忆 | 被动的——你写什么存什么 | 主动的——自己总结、自己生成 Skill、自己迭代 |
| Skill 来源 | 5700 个 Skill 全部社区手写 | Skill 是它自己长的 |
| Claude 支持 | 已被封杀，只能用 API | **原生支持 Claude Code 凭证** |
| 界面 | 有 GUI | CLI 为主 |

> 我的判断：两者不是替代关系，而是互补。龙虾当执行位（Skill 多、平台接入全），Hermes 当指挥位（记忆好、会进化）。

## 二、安装方式

### 方式一：一键脚本（推荐新手）

Linux / macOS / WSL2：
```bash
curl -s https://raw.githubusercontent.com/NousResearch/Hermes/main/scripts/install.sh | bash
```

Windows（原生 PowerShell）：
```powershell
irm https://raw.githubusercontent.com/NousResearch/Hermes/main/scripts/install.ps1 | iex
```

脚本会自动安装 Python 3.11、Node.js、Git、ripgrep 等所有依赖。

### 方式二：仓库开发安装（想改代码的）
```bash
git clone https://github.com/NousResearch/Hermes.git
cd Hermes
pip install -e .
```

### 方式三：Docker（想隔离环境的）
```bash
docker pull nousresearch/hermes-agent:latest
docker run -it nousresearch/hermes-agent:latest
```

> ⚠️ 官方推荐 Linux / macOS / WSL2，原生 Windows 坑多。

## 三、首次配置：Quick Setup

安装完成后会自动进入配置向导，选 `Quick setup`。

### 3.1 配置模型供应商

支持 200+ 模型，国内常用配置：

- **Kimi / 豆包**：选 OpenAI Compatible，填 API Key 和 Base URL
- **Claude**：原生支持，可用 Claude Code 订阅凭证（这是相对龙虾的巨大优势）
- **OpenAI**：直接选，填 API Key

### 3.2 跳过消息平台

向导里的消息平台列表可能没有你要的（比如飞书），先跳过，后面用 `hermes gateway setup` 单独配。

### 3.3 确认自动化配置

一路输入 `Y` 确认。看到欢迎界面就说明安装成功。

## 四、Windows 踩坑实录

### 坑一：模型未被识别

安装完发现模型显示不对。解决：用内置命令手动指定：
```
/model
```
然后选择你要的模型，看到名称正确显示即可。

### 坑二：飞书网关缺依赖

运行 `hermes gateway` 启动飞书网关时报错，原因是 venv 里缺 `lark-oapi`。

解决：
```powershell
uv pip install lark-oapi --python "C:\Users\你的用户名\AppData\Local\hermes\hermes-agent\venv\Scripts\python.exe"
```

### 坑三：WSL 下飞书插件断联

WSL 部署时飞书网关老是断。解决：放弃 WSL，写 PowerShell 脚本在 Windows 原生启动。

## 五、接入飞书：完整流程

### 5.1 创建飞书机器人

1. 去飞书开放平台新建机器人应用
2. 获取 App ID 和 App Secret
3. 配置机器人权限（消息读写等）

### 5.2 配置 Gateway
```bash
hermes gateway setup
```

选飞书，依次填入：
- **App ID**：你的飞书应用 ID
- **App Secret**：你的飞书应用密钥
- **来源**：国内版填 `feishu`，海外版填 `lark`
- **连接方式**：默认 websocket，直接回车
- **允许的 User ID**：留空，鉴权选 1（不限制）

### 5.3 启动 Gateway
```bash
hermes gateway
```

成功后，在飞书群里 @ 机器人就能对话了。

## 六、核心能力拆解

### 6.1 自动生成 Skill（核心卖点）

机制：当你完成一个复杂任务（调用 5 次以上工具），Hermes 会自动把整个过程沉淀成一份 Markdown 格式的 Skill 文档，遵循 agentskills.io 开放标准。

实测：我让它帮我抓取某网站数据并生成可视化图表。完成后，它自动生成了一个 `data_scraping_visualization.md`，里面包含：
- 任务拆解步骤
- 用到的工具和参数
- 错误处理逻辑
- 可复用的代码片段

下次我说"帮我抓个数据做图表"，它直接调用这个 Skill，不用从头推理。

更牛的是：Skill 会自我迭代。如果后续执行发现更好的方法，它会自动更新文档。

> 龙虾的 5700 个 Skill 是社区人写的。Hermes 的 Skill 是它自己长的。这是本质区别。

### 6.2 四层记忆系统

| 层级 | 名称 | 形式 | 说明 |
|------|------|------|------|
| 第一层 | MEMORY.md | 文件 | 存环境信息和项目上下文 |
| 第二层 | USER.md | 文件 | 存用户偏好和工作习惯 |
| 第三层 | SQLite | 数据库 | 历史对话全量存储 |
| 第四层 | 模型摘要 | 实时生成 | 自动压缩关键信息 |

实测体验：我让它记住"我喜欢简洁的代码风格，不要写注释"。之后的所有代码生成任务，它都自动遵循这个偏好，不用每次提醒。

> 对比龙虾的记忆：龙虾是"你说什么它存什么"，Hermes 是"它自己理解、自己总结、自己调用"。

### 6.3 安全机制

Hermes 在框架层面做了安全设计：
- **用户授权**：敏感操作需要确认
- **危险命令审批**：删除、修改系统文件等会弹窗
- **容器隔离**：可选 Docker 沙箱运行
- **上下文扫描**：防止 prompt 注入

> 龙虾的安全依赖大模型自己判断，Hermes 在框架层就拦住了。用一般的模型也够安全。

## 七、实战场景

### 场景一：指挥龙虾干活

让 Hermes 当指挥，龙虾当执行。在 Discord 里跟 Hermes 说任务，它调用龙虾的 Skill、Claude Code、CLI 来完成。Hermes 记住偏好和上下文，龙虾负责接入各种平台和工具。

### 场景二：自动沉淀工作流

做 GEO 需要重复的流程：抓竞品数据 → 分析结构 → 生成报告。以前每次都要重新描述需求。现在 Hermes 自动把这套流程沉淀成 Skill，只要说"跑一遍竞品分析"，它就自动执行完整流程。

### 场景三：本地知识库

Hermes 支持 Karpathy 大神的 LLM wiki，能在 Obsidian 围绕各种主题做持续迭代的本地知识库。把行业研究资料扔进去，它自动整理、建立关联、形成可检索的知识体系。

## 八、从龙虾迁移

```bash
npx @openclaws/hermes-migrator
```

它会自动导入：
- 记忆文件
- Skill 配置
- API Key

可用 `--dry-run` 参数先预览，搬家之前验货。

## 九、避坑清单

| 坑 | 原因 | 解决 |
|----|------|------|
| Windows 下飞书断联 | 环境兼容问题 | 用原生 Windows 启动，别用 WSL |
| 模型不识别 | 配置未生效 | 用 `/model` 命令手动指定 |
| 飞书缺依赖 | venv 没装 lark-oapi | 手动 pip install 到 Hermes 的 venv |
| Skill 没自动生成 | 任务太简单，工具调用不到 5 次 | 给它复杂任务或手动写 Skill |
| 记忆没生效 | USER.md 格式不对 | 按官方模板写，别自己瞎改 |

## 十、总结：值不值得用？

### 适合你的情况

- 你已经在用龙虾，想要更好的记忆和进化能力
- 你有 Claude 订阅，龙虾用不了，Hermes 可以
- 你需要 Agent 自动沉淀工作流，不想手写 Skill
- 你有命令行基础，不怕折腾

### 不适合的情况

- 你是纯小白，没用过命令行
- 你只需要简单的 AI 对话，不需要 Agent
- 你不想折腾，龙虾的 GUI 够用了

### 我的观点

Hermes 不是龙虾的替代品，是升级版的大脑。龙虾是出色的执行者，Hermes 是自我进化者。

**最佳实践是两者结合**：
- Hermes 当指挥位（记忆、进化、决策）
- 龙虾当执行位（Skill 多、平台全、干活快）

如果你从"用 AI 干活"进化到"让 AI 替你进化"这个阶段，Hermes 值得一试。

## 资源汇总

- **GitHub**：https://github.com/NousResearch/hermes-agent
- **官方文档**：仓库内 README.md 和 AGENTS.md
- **Skill 标准**：agentskills.io
- **飞书配置参考**：腾讯云/阿里云开发者社区有详细教程

---

tags: #AI-Agent #Hermes #OpenClaw #工具测评
