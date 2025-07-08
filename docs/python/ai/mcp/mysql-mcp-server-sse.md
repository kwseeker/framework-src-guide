# mysql-mcp-server-sse

[mangooer](https://github.com/mangooer)/[mysql-mcp-server-sse](https://github.com/mangooer/mysql-mcp-server-sse)

## 源码启动

```shell
pip install -r requirements.txt
# 复制一份配置并按需修改，需要将配置项同一行后面的注释去掉否则会作为值的一部分
cp .env.example .env
# 启动 MCP Server (server.py)
python -m src.server
# 调试启动，添加 .vscode/launch.json, 配置主文件
```

launch.json

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Python: Launch Server",
            "type": "debugpy",
            "request": "launch",
            "program": "${workspaceFolder}/src/server.py",
            "args": [],  // 可传递命令行参数，如 ["--port", "8000"]
            "cwd": "${workspaceFolder}",  // 工作目录设为项目根目录
            "env": {
                "PYTHONPATH": "${workspaceFolder}"  // 确保能识别 src 模块
            },
            "console": "integratedTerminal",  // 在集成终端中运行
            "justMyCode": true  // 只调试用户代码，跳过库代码
        }
    ]
}
```

MCP 客户端测试：

```shell
# 参考 FastMCP 写一个 MCP Client 进行测试
```

## 工作原理

基于 Python、[FastMCP](https://gofastmcp.com/getting-started/welcome)、aiomysql 等库开发。

1. 从 .env 加载配置

2. 导入 aiomysql

3. 使用 FastMCP 创建 MCP Server 实例，并配置 /sse 端点

4. 注册 MCP 工具到 MCP Server 实例 (工具在 tools 包中定义)

   一共定义了5类MCP工具（@mcp.tool）：

5. 注册关闭信号处理

6. 启动 MCP Server