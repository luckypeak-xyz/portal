---
title: 模板定制化
slug: /docs/tutorials/customization/template
---

## 概述

goctl 代码生成是基于 go 的模板去实现数据驱动的，虽然目前 goctl 的代码生成可以满足一部分代码生成功能，但是模板的自定义可以更加丰富代码生成。

模板指令可参考 <a href="/docs/tutorials/cli/template" target="_blank">goctl template</a>

## 示例

## 场景

实现统一格式的 body 响应，格式如下：

```json
{
  "code": 0,
  "msg": "OK",
  "data": {}
  // ①
}
```

① 实际响应数据

:::tip
`go-zero`生成的代码没有对其进行处理
:::

### 准备工作

我们提前在 `module` 为 `greet` 的工程下的 `response` 包中写一个 `Response` 方法，目录树类似如下：

```text
greet
├── response
│   └── response.go
└── xxx...
```

代码如下

```go
package response

import (
	"net/http"

	"github.com/zeromicro/go-zero/rest/httpx"
)

type Body struct {
	Code int         `json:"code"`
	Msg  string      `json:"msg"`
	Data interface{} `json:"data,omitempty"`
}

func Response(w http.ResponseWriter, resp interface{}, err error) {
    var body Body
    if err != nil {
        body.Code = -1
        body.Msg = err.Error()
    } else {
        body.Msg = "OK"
        body.Data = resp
    }
    httpx.OkJson(w, body)
}
```

### 修改 `handler` 模板

```shell
$ vim ~/.goctl/${goctl版本号}/api/handler.tpl
```

将模板替换为以下内容

```go
package handler

import (
	"net/http"
	"greet/response"// ①
	{{.ImportPackages}}
)

func {{.HandlerName}}(svcCtx *svc.ServiceContext) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		{{if .HasRequest}}var req types.{{.RequestType}}
		if err := httpx.Parse(r, &req); err != nil {
			httpx.Error(w, err)
			return
		}{{end}}

		l := logic.New{{.LogicType}}(r.Context(), svcCtx)
		{{if .HasResp}}resp, {{end}}err := l.{{.Call}}({{if .HasRequest}}&req{{end}})
		{{if .HasResp}}response.Response(w, resp, err){{else}}response.Response(w, nil, err){{end}}//②

	}
}
```

① 替换为你真实的`response`包名，仅供参考

② 自定义模板内容

:::tip 1.如果本地没有`~/.goctl/${goctl版本号}/api/handler.tpl`文件，可以通过模板初始化命令`goctl template init`进行初始化
:::

### 修改模板前后对比

- 修改前

```go
func GreetHandler(svcCtx *svc.ServiceContext) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		var req types.Request
		if err := httpx.Parse(r, &req); err != nil {
			httpx.Error(w, err)
			return
		}

		l := logic.NewGreetLogic(r.Context(), svcCtx)
		resp, err := l.Greet(&req)
		// 以下内容将被自定义模板替换
		if err != nil {
			httpx.Error(w, err)
		} else {
			httpx.OkJson(w, resp)
		}
	}
}
```

- 修改后

```go
func GreetHandler(svcCtx *svc.ServiceContext) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		var req types.Request
		if err := httpx.Parse(r, &req); err != nil {
			httpx.Error(w, err)
			return
		}

		l := logic.NewGreetLogic(r.Context(), svcCtx)
		resp, err := l.Greet(&req)
		response.Response(w, resp, err)
	}
}
```

### 修改模板前后响应体对比

- 修改前

```json
{
  "message": "Hello go-zero!"
}
```

- 修改后

```json
{
  "code": 0,
  "msg": "OK",
  "data": {
    "message": "Hello go-zero!"
  }
}
```

## 模板自定义规则

1. 在 goctl 提供的有效数据范围内修改，即不支持外部变量
2. 不支持新增模板文件
3. 不支持变量修改

## 模板变量

### goctl api -o 代码生成模板

模板默认目录 `~/.goctl/${goctl-version}/newapi`

#### api.tpl

```go
syntax = "v1"

info (
	title: // TODO: add title
	desc: // TODO: add description
	author: "{{.gitUser}}"
	email: "{{.gitEmail}}"
)

type request {
	// TODO: add members here and delete this comment
}

type response {
	// TODO: add members here and delete this comment
}

service {{.serviceName}} {
	@handler GetUser // TODO: set handler name and delete this comment
	get /users/id/:userId(request) returns(response)

	@handler CreateUser // TODO: set handler name and delete this comment
	post /users/create(request)
}

```

对应指令 `goctl api -o`

模板注入对象为 `map[string]string`

```go
map[string]string{
    "gitUser":     getGitName(),
    "gitEmail":    getGitEmail(),
    "serviceName": baseName + "-api",
}
```

| pipeline变量   | 类型     | 说明     |
|--------------|--------|--------|
| .gitUser     | string | Git用户名 |
| .gitEmail    | string | Git邮箱  |
| .serviceName | string | 服务名称   |

### goctl api go 代码生成模板

对应指令 `goctl api go ...`

模板默认目录 `~/.goctl/${goctl-version}/api`

#### config.tpl

```go
package config

import {{.authImport}}

type Config struct {
	rest.RestConf
	{{.auth}}
	{{.jwtTrans}}
}

```

模板注入对象为 `map[string]string`

```go
map[string]string{
    "authImport": authImportStr,
    "auth":       strings.Join(auths, "\n"),
    "jwtTrans":   strings.Join(jwtTransList, "\n"),
}
```

| pipeline变量  | 类型     | 说明    |
|-------------|--------|-------|
| .authImport | string | 认证导入  |
| .auth       | string | 认证配置  |
| .jwtTrans   | string | JWT配置 |

#### etc.tpl

```go
Name: {{.serviceName}}
Host: {{.host}}
Port: {{.port}}
```

模板注入对象为 map[string]string

```go
map[string]string{
    "serviceName": service.Name,
    "host":        host,
    "port":        port,
}
```

| pipeline变量   | 类型     | 说明   |
|--------------|--------|------|
| .serviceName | string | 服务名称 |
| .host        | string | 主机地址 |
| .port        | string | 端口号  |

#### handler.tpl

```go
package {{.PkgName}}

import (
	"net/http"

	"github.com/zeromicro/go-zero/rest/httpx"
	{{.ImportPackages}}
)

{{if .HasDoc}}{{.Doc}}{{end}}
func {{.HandlerName}}(svcCtx *svc.ServiceContext) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		{{if .HasRequest}}var req types.{{.RequestType}}
		if err := httpx.Parse(r, &req); err != nil {
			httpx.ErrorCtx(r.Context(), w, err)
			return
		}

		{{end}}l := {{.LogicName}}.New{{.LogicType}}(r.Context(), svcCtx)
		{{if .HasResp}}resp, {{end}}err := l.{{.Call}}({{if .HasRequest}}&req{{end}})
		if err != nil {
			httpx.ErrorCtx(r.Context(), w, err)
		} else {
			{{if .HasResp}}httpx.OkJsonCtx(r.Context(), w, resp){{else}}httpx.Ok(w){{end}}
		}
	}
}

```

模板注入对象为 `map[string]any`

```go
map[string]any{
    "PkgName":        pkgName,
    "ImportPackages": genHandlerImports(group, route, rootPkg),
    "HandlerName":    handler,
    "RequestType":    util.Title(route.RequestTypeName()),
    "LogicName":      logicName,
    "LogicType":      strings.Title(getLogicName(route)),
    "Call":           strings.Title(strings.TrimSuffix(handler, "Handler")),
    "HasResp":        len(route.ResponseTypeName()) > 0,
    "HasRequest":     len(route.RequestTypeName()) > 0,
    "HasDoc":         len(route.JoinedDoc()) > 0,
    "Doc":            getDoc(route.JoinedDoc()),
}
```

| pipeline变量      | 类型     | 说明                               |
|-----------------|--------|----------------------------------|
| .PkgName        | string | 包名                               |
| .ImportPackages | string | 导入包                              |
| .HasDoc         | bool   | 是否有文档注释                          |
| .Doc            | string | 文档注释                             |
| .HandlerName    | string | handler 名称                       |
| .HasRequest     | bool   | 是否有请求体                           |
| .RequestType    | string | 请求类型                             |
| .LogicName      | string | 逻辑包名称，默认为logic,如果有分组时为具体的group名称 |
| .LogicType      | string | 逻辑对象type名称                       |
| .Call           | string | logic 对象调用方法名称                   |
| .HasResp        | bool   | 是否有响应体                           |

#### logic.tpl

```go
package {{.pkgName}}

import (
	{{.imports}}
)

type {{.logic}} struct {
	logx.Logger
	ctx    context.Context
	svcCtx *svc.ServiceContext
}

{{if .hasDoc}}{{.doc}}{{end}}
func New{{.logic}}(ctx context.Context, svcCtx *svc.ServiceContext) *{{.logic}} {
	return &{{.logic}}{
		Logger: logx.WithContext(ctx),
		ctx:    ctx,
		svcCtx: svcCtx,
	}
}

func (l *{{.logic}}) {{.function}}({{.request}}) {{.responseType}} {
	// todo: add your logic here and delete this line

	{{.returnString}}
}

```

模板注入对象为 `map[string]any`

```go
map[string]any{
    "pkgName":      subDir[strings.LastIndex(subDir, "/")+1:],
    "imports":      imports,
    "logic":        strings.Title(logic),
    "function":     strings.Title(strings.TrimSuffix(logic, "Logic")),
    "responseType": responseString,
    "returnString": returnString,
    "request":      requestString,
    "hasDoc":       len(route.JoinedDoc()) > 0,
    "doc":          getDoc(route.JoinedDoc()),
}
```

| pipeline变量    | 类型     | 说明                   |
|---------------|--------|----------------------|
| .pkgName      | string | 包名                   |
| .imports      | string | 导入包                  |
| .logic        | string | 逻辑结构体名称              |
| .hasDoc       | bool   | 是否有文档注释              |
| .doc          | string | 文档注释                 |
| .function     | string | logic 函数名称           |
| .request      | string | 请求体表达式，包含参数名称，参数类型   |
| .responseType | string | 响应类型体表达式，包含参数名称，参数类型 |
| .returnString | string | 返回语句，返回的结构体          |

#### main.tpl

```go
package main

import (
	"flag"
	"fmt"

	{{.importPackages}}
)

var configFile = flag.String("f", "etc/{{.serviceName}}.yaml", "the config file")

func main() {
	flag.Parse()

	var c config.Config
	conf.MustLoad(*configFile, &c)

	server := rest.MustNewServer(c.RestConf)
	defer server.Stop()

	ctx := svc.NewServiceContext(c)
	handler.RegisterHandlers(server, ctx)

	fmt.Printf("Starting server at %s:%d...\n", c.Host, c.Port)
	server.Start()
}

```

模板注入对象为 `map[string]string`

```go
map[string]string{
    "importPackages": genMainImports(rootPkg),
    "serviceName":    configName,
}
```

| pipeline变量      | 类型     | 说明   |
|-----------------|--------|------|
| .importPackages | string | 导入包  |
| .serviceName    | string | 服务名称 |

#### middleware.tpl

```go
package middleware

import "net/http"

type {{.name}} struct {
}

func New{{.name}}() *{{.name}} {
	return &{{.name}}{}
}

func (m *{{.name}})Handle(next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		// TODO generate middleware implement function, delete after code implementation

		// Passthrough to next handler if need
		next(w, r)
	}
}

```

模板注入对象为 `map[string]string`

```go
map[string]string{
    "name": strings.Title(name),
}
```

| pipeline变量 | 类型     | 说明    |
|------------|--------|-------|
| .name      | string | 中间件名称 |

#### svc.tpl

```go
package svc

import (
	{{.configImport}}
)

type ServiceContext struct {
	Config {{.config}}
	{{.middleware}}
}

func NewServiceContext(c {{.config}}) *ServiceContext {
	return &ServiceContext{
		Config: c,
		{{.middlewareAssignment}}
	}
}

```

模板注入对象为 `map[string]string`

```go
map[string]string{
    "configImport":         configImport,
    "config":               "config.Config",
    "middleware":           middlewareStr,
    "middlewareAssignment": middlewareAssignment,
}
```

| pipeline变量            | 类型     | 说明      |
|-----------------------|--------|---------|
| .configImport         | string | pkg 导入  |
| .config               | string | 配置结构体名称 |
| .middleware           | string | 中间件字段   |
| .middlewareAssignment | string | 中间件赋值语句 |

#### type.tpl

```go
// Code generated by goctl. DO NOT EDIT.
package types{{if .containsTime}}
import (
	"time"
){{end}}
{{.types}}
```

模板注入对象为 `map[string]any`

```go
map[string]any{
    "types":        val,
    "containsTime": false,
}
```

| pipeline变量    | 类型     | 说明     |
|---------------|--------|--------|
| .containsTime | bool   | 是否包含时间 |
| .types        | string | 类型     |

### goctl api new 代码生成模板

对应指令为 `goctl api new ...`

模板默认目录 `~/.goctl/${goctl-version}/newapi`

#### api.tpl

```go
syntax = "v1"

type Request {
  Name string `path:"name,options=you|me"`
}

type Response {
  Message string `json:"message"`
}

service {{.name}}-api {
  @handler {{.handler}}Handler
  get /from/:name(Request) returns (Response)
}

```

模板注入对象 `map[string]string`

```go
map[string]string{
    "name":    dirName,
    "handler": strings.Title(dirName),
}
```

| pipeline变量 | 类型     | 说明    |
|------------|--------|-------|
| .name      | string | 服务名称  |
| .handler   | string | 处理器名称 |

### mongo 代码生成模板

模板默认目录 `~/.goctl/${goctl-version}/mongo`

#### error.tpl

```go
package model

import (
	"errors"

	"github.com/zeromicro/go-zero/core/stores/mon"
)

var (
	ErrNotFound        = mon.ErrNotFound
	ErrInvalidObjectId = errors.New("invalid objectId")
)

```

#### model.tpl

```go
// Code generated by goctl. DO NOT EDIT.
package model

import (
    "context"
    "time"

    {{if .Cache}}"github.com/zeromicro/go-zero/core/stores/monc"{{else}}"github.com/zeromicro/go-zero/core/stores/mon"{{end}}
    "go.mongodb.org/mongo-driver/bson"
    "go.mongodb.org/mongo-driver/bson/primitive"
    "go.mongodb.org/mongo-driver/mongo"
)

{{if .Cache}}var prefix{{.Type}}CacheKey = "cache:{{.lowerType}}:"{{end}}

type {{.lowerType}}Model interface{
    Insert(ctx context.Context,data *{{.Type}}) error
    FindOne(ctx context.Context,id string) (*{{.Type}}, error)
    Update(ctx context.Context,data *{{.Type}}) (*mongo.UpdateResult, error)
    Delete(ctx context.Context,id string) (int64, error)
}

type default{{.Type}}Model struct {
    conn {{if .Cache}}*monc.Model{{else}}*mon.Model{{end}}
}

func newDefault{{.Type}}Model(conn {{if .Cache}}*monc.Model{{else}}*mon.Model{{end}}) *default{{.Type}}Model {
    return &default{{.Type}}Model{conn: conn}
}


func (m *default{{.Type}}Model) Insert(ctx context.Context, data *{{.Type}}) error {
    if data.ID.IsZero() {
        data.ID = primitive.NewObjectID()
        data.CreateAt = time.Now()
        data.UpdateAt = time.Now()
    }

    {{if .Cache}}key := prefix{{.Type}}CacheKey + data.ID.Hex(){{end}}
    _, err := m.conn.InsertOne(ctx, {{if .Cache}}key, {{end}} data)
    return err
}

func (m *default{{.Type}}Model) FindOne(ctx context.Context, id string) (*{{.Type}}, error) {
    oid, err := primitive.ObjectIDFromHex(id)
    if err != nil {
        return nil, ErrInvalidObjectId
    }

    var data {{.Type}}
    {{if .Cache}}key := prefix{{.Type}}CacheKey + id{{end}}
    err = m.conn.FindOne(ctx, {{if .Cache}}key, {{end}}&data, bson.M{"_id": oid})
    switch err {
    case nil:
        return &data, nil
    case {{if .Cache}}monc{{else}}mon{{end}}.ErrNotFound:
        return nil, ErrNotFound
    default:
        return nil, err
    }
}

func (m *default{{.Type}}Model) Update(ctx context.Context, data *{{.Type}}) (*mongo.UpdateResult, error) {
    data.UpdateAt = time.Now()
    {{if .Cache}}key := prefix{{.Type}}CacheKey + data.ID.Hex(){{end}}
    res, err := m.conn.UpdateOne(ctx, {{if .Cache}}key, {{end}}bson.M{"_id": data.ID}, bson.M{"$set": data})
    return res, err
}

func (m *default{{.Type}}Model) Delete(ctx context.Context, id string) (int64, error) {
    oid, err := primitive.ObjectIDFromHex(id)
    if err != nil {
        return 0, ErrInvalidObjectId
    }
	{{if .Cache}}key := prefix{{.Type}}CacheKey +id{{end}}
    res, err := m.conn.DeleteOne(ctx, {{if .Cache}}key, {{end}}bson.M{"_id": oid})
	return res, err
}

```

模板注入对象为 `map[string]any`

```go
map[string]any{
    "Type":      stringx.From(t).Title(),
    "lowerType": stringx.From(t).Untitle(),
    "Cache":     ctx.Cache,
}
```

| pipeline变量 | 类型     | 说明       |
|------------|--------|----------|
| .Cache     | bool   | 是否启用缓存   |
| .Type      | string | 模型类型名称   |
| .lowerType | string | 小写模型类型名称 |

#### model_custom.tpl

```go
package model

{{if .Cache}}import (
    "github.com/zeromicro/go-zero/core/stores/cache"
    "github.com/zeromicro/go-zero/core/stores/monc"
){{else}}import "github.com/zeromicro/go-zero/core/stores/mon"{{end}}

{{if .Easy}}
const {{.Type}}CollectionName = "{{.snakeType}}"
{{end}}

var _ {{.Type}}Model = (*custom{{.Type}}Model)(nil)

type (
    // {{.Type}}Model is an interface to be customized, add more methods here,
    // and implement the added methods in custom{{.Type}}Model.
    {{.Type}}Model interface {
        {{.lowerType}}Model
    }

    custom{{.Type}}Model struct {
        *default{{.Type}}Model
    }
)


// New{{.Type}}Model returns a model for the mongo.
{{if .Easy}}func New{{.Type}}Model(url, db string{{if .Cache}}, c cache.CacheConf{{end}}) {{.Type}}Model {
    conn := {{if .Cache}}monc{{else}}mon{{end}}.MustNewModel(url, db, {{.Type}}CollectionName{{if .Cache}}, c{{end}})
    return &custom{{.Type}}Model{
        default{{.Type}}Model: newDefault{{.Type}}Model(conn),
    }
}{{else}}func New{{.Type}}Model(url, db, collection string{{if .Cache}}, c cache.CacheConf{{end}}) {{.Type}}Model {
    conn := {{if .Cache}}monc{{else}}mon{{end}}.MustNewModel(url, db, collection{{if .Cache}}, c{{end}})
    return &custom{{.Type}}Model{
        default{{.Type}}Model: newDefault{{.Type}}Model(conn),
    }
}{{end}}

```

模板注入对象为 `map[string]any`

```go
map[string]any{
    "Type":      stringx.From(t).Title(),
    "lowerType": stringx.From(t).Untitle(),
    "snakeType": stringx.From(t).ToSnake(),
    "Cache":     ctx.Cache,
    "Easy":      ctx.Easy,
}
```

| pipeline变量 | 类型     | 说明       |
|------------|--------|----------|
| .Cache     | bool   | 是否启用缓存   |
| .Easy      | bool   | 是否简易模式   |
| .Type      | string | 模型类型名称   |
| .snakeType | string | 蛇形模型类型名称 |
| .lowerType | string | 小写模型类型名称 |

#### types.tpl

```go
package model

import (
	"time"

	"go.mongodb.org/mongo-driver/bson/primitive"
)

type {{.Type}} struct {
	ID primitive.ObjectID `bson:"_id,omitempty" json:"id,omitempty"`
	// TODO: Fill your own fields
	UpdateAt time.Time `bson:"updateAt,omitempty" json:"updateAt,omitempty"`
	CreateAt time.Time `bson:"createAt,omitempty" json:"createAt,omitempty"`
}

```

模板注入对象为 `map[string]any`

```go
map[string]any{
    "Type": stringx.From(t).Title(),
}
```

| pipeline变量 | 类型     | 说明     |
|------------|--------|--------|
| .Type      | string | 模型类型名称 |

## 参考文献

- <a href="/docs/tutorials/cli/template" target="_blank">《goctl template》</a>
- <a href="https://golang.org/pkg/text/template/" target="_blank">《text/template》</a>
