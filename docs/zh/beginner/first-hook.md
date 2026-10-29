---
next:
  text: '开发者指南：概述'
  link: '/zh/dev/overview'
---

# 7. 动手：第一个自定义 hook

**本页问题：** 怎样亲眼证明 hook 在目标函数前后执行了，而原来的结果没有改变？

我们为标准库 `net/http` 的 `(*Transport).RoundTrip` 写一个只打印日志的 hook。程序发出一次 HTTP 请求，分别用普通 `go build` 和 `otel go build` 构建，再比较输出。

开始前请先[安装 `otel`](../index.md#安装)，并确认 `otel version` 可以运行。

## 1. 准备目录

```text
hooklab/
├── go.mod
├── main.go        # 业务程序：发出一次 HTTP 请求
├── rule.json      # 规则：找 RoundTrip
└── rules/
    ├── go.mod
    └── hook.go    # hook：进入和返回时各打印一行
```

## 2. 业务程序

`main.go` 在本机启动一个测试服务器，对它发一次 GET 请求，再打印结果。代码中初始化的 OpenTelemetry SDK 与本实验无关，可以忽略。

```go
package main

import (
	"context"
	"fmt"
	"net/http"
	"net/http/httptest"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/sdk/trace"
)

func target(value int) int { return value + 1 }

func main() {
	provider := trace.NewTracerProvider()
	otel.SetTracerProvider(provider)
	defer provider.Shutdown(context.Background())
	_, root := otel.Tracer("guide").Start(context.Background(), "root")
	defer root.End()
	s := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) { w.WriteHeader(http.StatusNoContent) }))
	defer s.Close()
	res, err := http.Get(s.URL)
	if err != nil {
		panic(err)
	}
	defer res.Body.Close()
	fmt.Println("plain-result", target(41), "status", res.StatusCode)
}
```

在 `hooklab/` 中执行：

```bash
go mod init example.com/hooklab
go get go.opentelemetry.io/otel@v1.45.0 go.opentelemetry.io/otel/sdk@v1.45.0
go mod tidy
```

## 3. 规则

`rule.json`：

```json
[
  {
    "ImportPath": "net/http",
    "Function": "RoundTrip",
    "ReceiverType": "\\*Transport",
    "OnEnter": "guideHttpEnter",
    "OnExit": "guideHttpExit",
    "Path": "./rules"
  }
]
```

## 4. hook

`rules/hook.go`：

```go
package rules

import (
	"fmt"
	"net/http"
	_ "unsafe"

	"github.com/alibaba/loongsuite-go/pkg/api"
)

//go:linkname guideHttpEnter net/http.guideHttpEnter
func guideHttpEnter(call api.CallContext, t *http.Transport, req *http.Request) {
	fmt.Printf("hook-enter method=%s\n", req.Method)
}

//go:linkname guideHttpExit net/http.guideHttpExit
func guideHttpExit(call api.CallContext, res *http.Response, err error) {
	if err != nil {
		fmt.Printf("hook-error=%v\n", err)
		return
	}
	if res != nil {
		fmt.Printf("hook-exit status=%d\n", res.StatusCode)
	}
}
```

对照[编写 Hook 代码](../dev/hook.md)看三个约定：

- 第一个参数是 `api.CallContext`。
- OnEnter 其余参数对应目标函数的接收者和参数：`(t *Transport) RoundTrip(req *Request)`。
- OnExit 其余参数对应目标函数的返回值：`(*Response, error)`。

`//go:linkname` 把这两个函数与工具生成的调用点连接起来，所以要导入 `unsafe`。

`rules/go.mod` 声明 hook 依赖的 `pkg` 模块。版本应与你使用的 `otel` 工具一致，下面是 v1.15.0 对应的版本：

```text
module guidehook

go 1.25.0

require github.com/alibaba/loongsuite-go/pkg v0.0.0-20260928033626-cac400935add
```

然后在 `rules/` 中执行 `go mod download github.com/alibaba/loongsuite-go/pkg` 生成 `go.sum`。

## 5. 比较两次构建

普通构建：

```bash
$ go run .
plain-result 42 status 204
```

使用 `otel` 构建：

```bash
$ otel set -disable= -rule=$PWD/rule.json
$ otel go build -o instrumented .
$ ./instrumented
```

输出中会出现：

```text
hook-enter method=GET
hook-exit status=204
plain-result 42 status 204
```

两次构建的业务结果都是 `plain-result 42 status 204`，插桩后的程序多了 hook 的两行输出。这说明规则命中了 `RoundTrip`，hook 在请求前后执行，原来的结果没有改变。

> [!NOTE]
> - `-disable=` 表示保留全部默认规则，`-rule` 在此基础上追加你的规则。不要用 `-disable=all`，它会去掉工具自身需要的规则。参数说明见[命令行参考](../user/config.md)。
> - 如果本机没有运行 OpenTelemetry Collector，终端可能还会出现数据导出失败的日志，这不影响本实验。导出器的设置见 [OpenTelemetry 配置](../user/sdk-config.md)。

## 没有看到 hook 输出？

按构建时到运行时的顺序排查：

1. 规则有没有命中？在 `.otel-build/matched.json` 中搜索 `net/http`，确认你的规则在里面。其他调试方法见[调试](../hacking/debug.md)。
2. 构建有没有报错？检查 hook 签名是否与目标函数一致。
3. 运行的是不是 `./instrumented`，而不是 `go run .` 生成的普通程序？

**互动版**：社区维护的 [LoongSuite Go 中文学习指南 · 动手实验](https://qinghua362330.github.io/loongsuite-go-guide/#/labs) 提供本页实验的完整文件和一键运行脚本。

## 接下来

- [Hook 规则类型](../dev/rule_def.md)：给结构体加字段、给包加文件等其他规则。
- [注册Hook规则](../dev/register.md)：把规则贡献到官方仓库。
- [测试Hook代码](../dev/test.md)：为规则编写测试。
- [埋点阶段](../hacking/instrument.md)：hook 调用是怎样被插进原函数的。
