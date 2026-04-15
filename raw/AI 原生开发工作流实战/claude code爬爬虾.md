1. 一个新的项目使用/init 去分析
2. /compack合并信息
3. /clear清除信息
4. ！可以直接使用终端命令
5. `#` 进入记忆模式
6. /ide 可以选择ide是什么
7. claude -p "今天是几号了"，这个是非交互模式，开启临时的一次性对话
8. 安装mcp
   1. claude map add context7 -- npx -y @upstash/context7-mcp
9. 删除mcp
   1. claude  mcp remove context7
10. /permissions
    1. 定义内置工具例如 Bash(git commit:*)
    2. 定义mcp例如 mcp__context7
    3. 或者直接 claude --dangerously-skip-permissions

11 . 自定义命令commands

## 安装

官网：https://claude.com/product/claude-code

文档：https://code.claude.com/docs/en/overview

npm install -g @anthropic-ai/claude-code

查看版本：claude --version

更新：claude update  



