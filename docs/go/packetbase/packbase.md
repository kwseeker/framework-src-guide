# PacketBase

一个开源的 Go 后端应用。包括：具有实时订阅的嵌入式数据库 （SQLite）、内置文件和用户管理、方便的 Admin dashboard UI

和简单的 REST 式 API，另外**还可以使用 Go / JS 进行业务拓展**（比如自定义路由、接入数据库、事件钩子、任务调度、模板引擎、实时消息、文件系统、日志等等）。可以作为标准的后端应用、开发框架、工具包。

> 感觉定位上更接近是一个支持灵活拓展的低代码平台。

## 功能

```go
require (
	github.com/disintegration/imaging v1.6.2		//图像处理库（缩放、裁剪、滤镜等）
	github.com/domodwyer/mailyak/v3 v3.6.2			//邮件构建和发送库
	github.com/dop251/goja v0.0.0-20250309171923-bcd7cc6bf64c	//Go 实现的 JavaScript 解释器
	github.com/dop251/goja_nodejs v0.0.0-20250314160716-c55ecee183c0  //为 goja 提供 Node.js 兼容 API
	github.com/fatih/color v1.18.0					//命令行彩色输出
	github.com/fsnotify/fsnotify v1.7.0				//文件系统监控
	github.com/gabriel-vasile/mimetype v1.4.8		//文件 MIME 类型检测
	github.com/ganigeorgiev/fexpr v0.5.0			//
	github.com/go-ozzo/ozzo-validation/v4 v4.3.0	//数据验证库
	github.com/golang-jwt/jwt/v5 v5.2.2
	github.com/pocketbase/dbx v1.11.0				//数据库抽象层
	github.com/pocketbase/tygoja v0.0.0-20250103200817-ca580d8c5119
	github.com/spf13/cast v1.7.1	//安全的类型转换
	github.com/spf13/cobra v1.9.1	//CLI 应用框架
	golang.org/x/crypto v0.37.0		//加密算法
	golang.org/x/image v0.26.0		//Go 官方图像处理扩展
	golang.org/x/net v0.39.0
	golang.org/x/oauth2 v0.29.0		//OAuth2 客户端
	golang.org/x/sync v0.13.0
	modernc.org/sqlite v1.37.0
)

// 间接依赖
require (
	github.com/asaskevich/govalidator v0.0.0-20230301143203-a9d515a09cc2 // indirect
	github.com/dlclark/regexp2 v1.11.5 // indirect
	github.com/dop251/base64dec v0.0.0-20231022112746-c6c9f9a96217 // indirect
	github.com/dustin/go-humanize v1.0.1 // indirect
	github.com/go-sourcemap/sourcemap v2.1.4+incompatible // indirect
	github.com/google/pprof v0.0.0-20250317173921-a4b03ec1a45e // indirect
	github.com/google/uuid v1.6.0 // indirect
	github.com/inconshreveable/mousetrap v1.1.0 // indirect
	github.com/mattn/go-colorable v0.1.14 // indirect
	github.com/mattn/go-isatty v0.0.20 // indirect
	github.com/ncruces/go-strftime v0.1.9 // indirect
	github.com/remyoudompheng/bigfft v0.0.0-20230129092748-24d4a6f8daec // indirect
	github.com/spf13/pflag v1.0.6 // indirect
	golang.org/x/exp v0.0.0-20250408133849-7e4ce0ab07d0 // indirect
	golang.org/x/mod v0.24.0 // indirect
	golang.org/x/sys v0.32.0 // indirect
	golang.org/x/text v0.24.0 // indirect
	golang.org/x/tools v0.32.0 // indirect
	modernc.org/libc v1.62.1 // indirect
	modernc.org/mathutil v1.7.1 // indirect
	modernc.org/memory v1.9.1 // indirect
)
```

