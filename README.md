# 课程

课程：[Go 语言编写简单分布式系统（完结）](https://www.bilibili.com/video/BV1ZU4y1577q?p=1&vd_source=78aed90022f9db1abfe1d828237d3862)

最近学习 `go` 时，在 b 站看到杨旭老师用 `go` 写了一个简单的分布式系统，感觉很有意思，就跟着学习了一下

下面做的笔记是对课程的内容进行梳理，方便后续查阅

# NOTE

## 分布式

1. 注册服务：`RegistryService`
2. 日志服务：`LogService`
3. 其他服务：`GradingService`、`portal`

### RegistryService

`RegistryService` 提供的服务：

1. 提供 `/services` 接口，用于其他服务在启动或者停止时告知
   - `POST`：告诉 `RegistryService`，我启动了一个服务，调用 `add` 方法
   - `DELETE`：告诉 `RegistryService`，我停止了一个服务，调用 `remove` 方法
2. 通过 `add` 函数将服务添加到 `registrations` 列表中
   - `r.registrations = append(r.registrations, reg)`
3. 通过 `remove` 函数将服务从 `registrations` 列表中移除
   - `r.registrations = append(r.registrations[:i], r.registrations[i+1:]...)`
4. 这里需要注意的是：要保证线程安全，也就是在 `append` 时，需要使用到锁
   ```go
   mutex.Lock()
   append(xxx, xxx)
   mutex.UnLock()
   ```
5. 服务发现：
   1. 比如说 `GradingService` 依赖 `LogService`，那么 `GradingService` 就需要知道 `LogService` 的地址
   2. 这个时候 `RegistryService` 就可以通过 `registrations` 列表来通知 `GradingService`，`LogService` 的地址
   3. `RegistryService` 是通过 `ServiceUpdateURL` 来通知的，`GradingService`，`LogService` 的地址
6. 服务发现需要分两步进行
   1. 如果 `GradingService` 启动时，如果 `LogService` 已经启动了，那么 `RegistryService` 就可以直接通知 `GradingService`，`LogService` 的地址(`r.sendRequiredServices(reg)` 方法)
   2. 如果 `GradingService` 启动时，如果 `LogService` 还没有启动，那么 `RegistryService` 就不会通知 `GradingService`，`LogService` 的地址，等到 `LogService` 启动后，`RegistryService` 才会通知 `GradingService`，`LogService` 的地址(`notify` 方法)

`RegistryService` 对外只需要提供 `RegisterService` 方法，其他服务调用这个函数，就能够获取 `RegistryService` 提供的服务

1. 调用 `RegisterService` 提供的接口 `/services`，将服务注册到 `RegistryService` 中
2. 为注册的服务添加路由：`ServiceUpdateURL`
3. 为注册的服务添加 `ServeHTTP` 方法，用于处理 `ServiceUpdateURL` 的请求，这个请求在方法 `sendRequiredServices` 调用时相应，更新 `providers` 中的 `service`
4. 为每个注册的服务提供健康检查

最后在提供一个 `ShutdownService` 用于像 `/services` 接口发送 `delete` 请求，告知 `RegistryService`，我停止了一个服务

### LogService

`LogService` 服务是对日志进行管理，将其他服务的日志进行收集、存储，提供 `/log` 接口，用于其他服务将日志发送给 `LogService`

### GradingService 和 Portal

这两个是业务服务

1. 在启动服务时调用方法 `RegistryService`，将自己注册到 `RegistryService`中
2. 在停止服务时调用方法 `ShutdownService`，将自己从 `RegistryService` 中移除

## api

### os.OpenFile

用于指定模式打开文件，并返回文件的指针

```go
func OpenFile(name string, flag int, perm FileMode) (*File, error)
```

`flag` 参数：

- `os.O_RDONLY`：只读模式打开文件
- `os.O_WRONLY`：只写模式打开文件
- `os.O_RDWR`：读写模式打开文件
- `os.O_APPEND`：追加模式，写入内容时将数据附加到文件尾部
- `os.O_CREATE`：如果文件不存在，则创建一个新文件

`perm` 参数：

- `0`：无权限
- `1`：执行权限
- `2`：写权限
- `3`：写和执行权限
- `4`：读权限
- `5`：读和执行权限
- `6`：读和写权限
- `7`：读、写和执行权限

- `0644`：表示文件的所有者可以读取和写入文件，文件所属组和其他用户只能读取文件。这是比较常见的设置
- `0600`：表示文件的所有者可以读取和写入文件，但是文件所属组和其他用户不能访问该文件。这种权限安全性较高

### ioutil.ReadAll

可以将整个文件内容读取到内存中，可以将请求体的内容读取到内存中

ps：将整个文件的内容或者请求体一次性读取到内存中，对于非常大的文件或者请求体，内存占用过高

### fmt.Scanln

会阻塞程序的执行，直到用户在终端输入一行内容并按下回车键，然后它会将用户输入的值存储到传入的参数中

它主要用于读取并解析简单的基本类型数据

```go
func main(){
  var name string
	var age int

	fmt.Print("Enter your name: ")
	fmt.Scanln(&name)

	fmt.Print("Enter your age: ")
	fmt.Scanln(&age)

	fmt.Printf("Hello, %s! You are %d years old.\n", name, age)
}
```

## http

### http.Server

1. `ListenAndServe`：启动服务，并监听指定的地址和端口，会阻塞
2. `Shutdown`：优雅地关闭服务，可以保证正在处理的服务不会被中断

```go
var srv htto.Server
go func(){
  srv.ListenAndServe()
}()
go func(){
  srv.Shutdown()
}()
```

### ServeHTTP

当一个结构体实现了 `ServeHTTP` 方法后，那么这个结构体就实现了 `http.Handler` 接口

实现了 `http.Handler` 接口的结构体，就可以作为 `http.Handle` 方法的第二个参数

然后调用 `http.ListenAndServe` 方法就可以启动一个服务，会自动调用 `ServeHTTP` 方法来处理请求

```go
func main() {
	http.Handle("/ping", &MyHandler{})
	http.ListenAndServe(":8080", nil)
}

type MyHandler struct{}

func (mh MyHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	switch r.Method {
	case http.MethodGet:
		w.WriteHeader(http.StatusOK)
		w.Write([]byte("pong"))
	default:
		w.WriteHeader(http.StatusMethodNotAllowed)
	}
}
```

### 将结构体序列化

1. `buf := new(bytes.Buffer)` 创建了一个新的 `bytes.Buffer` 对象，用于存储编码后的 `JSON` 数据
2. `enc := json.NewEncoder(buf)` 创建了一个新的 `JSON` 编码器 `enc`，并将其关联到 `buf` 对象。这意味着编码后的 `JSON` 数据将被写入到 `buf` 中
3. `err := enc.Encode(r)` 使用 `JSON` 编码器 `enc` 将结构体 `r` 编码为 `JSON` 数据，并将结果写入到 `buf` 中。`Encode` 方法返回一个可能的错误 `err`

```go
type Registration struct {
	ServiceName string
	ServiceURL  string
}
r := Registration{
  ServiceName: "LogService",
  ServiceURL:  "http://localhost:3000/services",
}
buf := new(bytes.Buffer)
enc := json.NewEncoder(buf)
err := enc.Encode(r)

res, err := http.Post(ServicesURL, "application/json", buf)
```

### 使用 http 默认请求

1. `http.DefaultClient` 是标准库中提供的默认 `HTTP` 请求。它已经预先配置好了一些默认的设置，例如超时时间、重试机制等
2. `Do(req)` 是 `http.Client` 类型的方法，用于执行一个 `HTTP` 请求并返回响应
   - 它接受一个 `http.Request` 对象作为参数，表示要发送的请求

```go
req, _ := http.NewRequest(http.MethodDelete, "http://localhost:3000/services", bytes.NewBuffer([]byte("http://localhost:4000/log")))
req.Header.Add("Content-Type", "text/plain")
res, err := http.DefaultClient.Do(req)
```

## log

### `log.New`

`log.New` 用于创建一个新的日志记录器实例，用于将日志消息写入指定的输出地，并可选择性地添加前缀字符串

1. 以文件的形式记录日志，用 `log.New` 创建一个新的 `log` 实例，然后调用 `log.Printf` 方法将日志写入文件

它接收 `io.Writer` 类型的参数，`os.OpenFile` 返回的文件指针类型 `*os.File` 实现了 `io.Writer` 接口，所以可以将文件指针传入 `log.New` 方法中

代码参考如下：

```go
import (
	"fmt"
	stlog "log"
	"os"
)
func main() {
	file, err := os.OpenFile("./logs", os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0600)
	if err != nil {
		fmt.Println(err)
	}
	defer file.Close()
	log := stlog.New(file, "[go] -  ", stlog.LstdFlags)
	log.Println("hello world")
}
```

2. 重写 `log` 的 `Write` 方法，也能实现将日志写入文件

在重写 `Write` 方法时，需要定义一个类型别名，然后在类型别名上实现 `Write` 方法，那么这个类型别名就能够传入 `log.New` 方法中

代码参考如下：

```go
import (
	stlog "log"
	"os"
)
type filelog string

func (fl filelog) Write(data []byte) (int, error) {
	file, err := os.OpenFile(string(fl), os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0600)
	if err != nil {
		return 0, err
	}
	defer file.Close()
	file.Write(data)
	return len(data), nil
}

func main() {
	log := stlog.New(filelog("./logs"), "[go] -  ", stlog.LstdFlags)
	log.Println("hello world")
}
```
