# mcp-server-mysql

[@benborla29/mcp-server-mysql](https://smithery.ai/server/@benborla29/mcp-server-mysql/api)

## 1 本地启动

```shell
git clone https://github.com/benborla/mcp-server-mysql.git
cd mcp-server-mysql
npm install
# or
pnpm install
npm run build
# or
pnpm run build
pnpm run start
```

ts-node 本地调试：

```shell
# 没有安装 typescript 和 ts-node 的话先安装
npm install -g typescript ts-node
# VSCode 
# 创建 launch.json, 内容参考 ts-node 官方文档：https://typestrong.org/ts-node/docs/usage/
# 
```

launch.json:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "ts-node",
            "type": "node",
            "request": "launch",
            "args": [
                "${relativeFile}"
            ],
            // "runtimeArgs": [
            //     "-r",
            //     "ts-node/register"
            // ],            
            "runtimeArgs": [
                "--loader",
                "ts-node/esm"
            ],
            "cwd": "${workspaceRoot}",
            "protocol": "inspector",
            "internalConsoleOptions": "openOnSessionStart",
            "program": "${workspaceFolder}/index.ts",
            // "outFiles": ["${workspaceFolder}/out/**/*.js"]
        }
    ]
}
```

## 2 工作原理

1. 配置加载（dotenv)

   支持从 .env 文件加载配置，

2. 创建连接池、启动 MCP Server 并注册4种请求处理器、注册进程关闭钩子

   MCP Server 启动时使用的 Stdio 通信协议，暂时不支持 SSE。

   TODO: 自己拓展 SSE 功能。

3. 4种请求处理器（MCP工具）的处理逻辑

   