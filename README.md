**中文** | [English](README.en.md)

# PlayerServer（易播服务器）

基于 HTTP 协议的加密播放器服务器，提供用户登录校验、视频密钥分发等服务。请求通过 HTTP 承载，响应体使用 JSON 传递业务数据。

服务器采用 **多进程 + 线程池 + epoll** 的架构：

- 独立的 **日志进程**（Unix 域套接字接收日志，统一落盘）
- **用户接入进程**（监听端口，accept 后把客户端 socket 通过 `sendmsg` 传给业务进程）
- **业务处理进程**（epoll 收包，线程池解析 HTTP / 校验签名 / 访问数据库）
- 数据库层抽象出统一接口，内置 **MySQL** 与 **SQLite3** 两种实现

模块之间尽量用无锁队列和 fd 传递来同步，避免直接使用互斥锁。

## 目录结构

```
PlayerServer/
├── CMakeLists.txt              # Linux 下用 CMake 构建
├── EPlayerServer.vcxproj       # Visual Studio「Linux 远程调试」工程
├── EPlayerServer.vcxproj.filters
├── include/                    # 项目头文件
│   ├── Public.h                #   Buffer（std::string 派生的字节缓冲）
│   ├── Function.h              #   可变参数回调封装
│   ├── Thread.h / ThreadPool.h #   线程、线程池（epoll 驱动）
│   ├── Process.h               #   子进程创建、fd/socket 跨进程传递
│   ├── Epoll.h / Socket.h      #   epoll 封装、TCP/Unix 域 socket 封装
│   ├── Logger.h                #   日志客户端/服务端、TRACEx/LOGx 宏
│   ├── HttpParser.h            #   HTTP 请求解析、URL 解析
│   ├── Crypto.h                #   MD5
│   ├── DatabaseHelper.h        #   表/字段声明宏，数据库客户端抽象接口
│   ├── MysqlClient.h           #   MySQL 实现
│   ├── Sqlite3Client.h         #   SQLite3 实现
│   ├── CServer.h               #   接入服务器（监听 + 转发给业务进程）
│   └── EdoyunPlayerServer.h    #   播放器业务：登录校验、JSON 响应
├── src/                        # 对应实现 + main.cpp
└── third_party/                # 以源码形式内置的第三方库
    ├── http_parser/            #   Node.js http-parser（C）
    ├── jsoncpp/                #   JsonCpp 1.9.4（精简合并版）
    └── sqlite3/                #   SQLite 3.31.1 合并源码
```

## 环境依赖

仅支持 Linux（依赖 epoll、`sendmsg` 传 fd 等接口）。

| 依赖 | 说明 |
|------|------|
| g++ 7+ / CMake 3.10+ | 使用 C++14 |
| libmysqlclient-dev | MySQL 客户端库 |
| libssl-dev | OpenSSL，用于 MD5 |
| MySQL 5.7+ / MariaDB | 业务数据库，建表语句由代码自动执行 |

Ubuntu / Debian：

```bash
sudo apt install build-essential cmake libmysqlclient-dev libssl-dev
```

## 构建

```bash
git clone https://github.com/hatmore/PlayerServer.git
cd PlayerServer
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
```

也可以直接用 Visual Studio 打开 `EPlayerServer.vcxproj`，通过「Linux 远程调试」连接到一台 Linux 主机编译运行。

## 配置与运行

数据库连接参数位于 `include/EdoyunPlayerServer.h` 的 `BusinessProcess()` 中（host / user / password / port / db），首次运行会自动创建 `edoyun` 库下的用户表。监听地址与端口在 `CServer::Init` 的默认参数中（`0.0.0.0:9999`）。

```bash
cd build
mkdir -p log          # 日志进程把日志写到 ./log/
./EPlayerServer
```

## HTTP 接口

### 登录校验

```
GET /login?time=<时间戳>&salt=<随机盐>&user=<用户名>&sign=<签名>
```

服务器根据用户名查出密码后计算

```
sign = MD5(time + MD5_KEY + password + salt)
```

与请求中的 `sign` 比对。响应为 `HTTP/1.1 200 OK`，正文为 JSON：

```json
{ "status": 0, "message": "success" }
```

`status` 非 0 表示失败，`message` 给出失败原因。

## 日志

`TRACEI / TRACEE / TRACED / TRACEW` 宏以及 `LOGI / LOGE` 流式宏把日志通过 Unix 域套接字 `./log/server.sock` 发到日志进程，日志进程按启动时间写入 `./log/<时间>.log`。`DUMPD(data, size)` 用于以十六进制 dump 一段内存。

## 许可

本项目暂未声明开源许可证，第三方库各自遵循其原始许可（`third_party/` 下各库自带）。
