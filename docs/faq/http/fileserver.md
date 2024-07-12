# File Server

## 如何用 go-zero 服务同时提供文件服务？

可以通过 `go-zero` 的 `rest.WithFileServer(path, dir)` 来给 `restful` 服务增加文件服务能力。

示例代码如下：

```go
package main

import (
	"net/http"

	"github.com/zeromicro/go-zero/rest"
)

func main() {
    // 在 `html` 目录下有需要对外提供的文件，比如有个文件 `index.html`，
    // 以 `/static/index.html` 这样的路径就可以访问该文件了。
	server := rest.MustNewServer(rest.RestConf{
		Host: "localhost",
		Port: 4000,
	}, rest.WithFileServer("/static", "html"))
	defer server.Stop()

	server.AddRoute(rest.Route{
		Method:  http.MethodGet,
		Path:    "/hello",
		Handler: helloHandler,
	})

	server.Start()
}

func helloHandler(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	w.Write([]byte("Hello, World!"))
}
```

这仅仅是个示例，一般不用做生产服务，或者当生产服务很简单的时候可以考虑使用，但不是最佳实践，一般会通过 nginx 或者云存储提供。

go-zero 版本：>= v1.7.0