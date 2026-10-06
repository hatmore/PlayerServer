[中文](README.md) | **English**

# PlayerServer

An HTTP-based license server for an encrypted video player. It authenticates users and hands out video decryption keys. Requests travel over plain HTTP; responses carry business data as JSON.

The server is built on a **multi-process + thread-pool + epoll** design:

- a dedicated **logging process** (receives log records over a Unix domain socket and writes them to disk)
- an **accept process** (listens on the TCP port and passes accepted client sockets to the business process with `sendmsg`)
- a **business process** (epoll-driven receive loop, thread pool for HTTP parsing, signature checking and database access)
- a database abstraction with **MySQL** and **SQLite3** back ends

Processes synchronise through fd passing and lock-free queues rather than mutexes wherever possible.

## Layout

```
PlayerServer/
├── CMakeLists.txt              # CMake build for Linux
├── EPlayerServer.vcxproj       # Visual Studio "Linux remote" project
├── EPlayerServer.vcxproj.filters
├── include/                    # project headers
│   ├── Public.h                #   Buffer (byte buffer derived from std::string)
│   ├── Function.h              #   variadic callback wrapper
│   ├── Thread.h / ThreadPool.h #   thread and epoll-driven thread pool
│   ├── Process.h               #   child processes, fd / socket passing
│   ├── Epoll.h / Socket.h      #   epoll wrapper, TCP / Unix socket wrapper
│   ├── Logger.h                #   log client/server, TRACEx / LOGx macros
│   ├── HttpParser.h            #   HTTP request and URL parsing
│   ├── Crypto.h                #   MD5
│   ├── DatabaseHelper.h        #   table/field declaration macros, DB client interface
│   ├── MysqlClient.h           #   MySQL implementation
│   ├── Sqlite3Client.h         #   SQLite3 implementation
│   ├── CServer.h               #   accept server (listen + hand off to business process)
│   └── EdoyunPlayerServer.h    #   player business logic: login check, JSON response
├── src/                        # implementations + main.cpp
└── third_party/                # vendored sources
    ├── http_parser/            #   Node.js http-parser (C)
    ├── jsoncpp/                #   JsonCpp 1.9.4 (trimmed)
    └── sqlite3/                #   SQLite 3.31.1 amalgamation
```

## Requirements

Linux only (epoll, `sendmsg` fd passing).

| Dependency | Notes |
|------------|-------|
| g++ 7+ / CMake 3.10+ | C++14 |
| libmysqlclient-dev | MySQL client library |
| libssl-dev | OpenSSL, used for MD5 |
| MySQL 5.7+ / MariaDB | business database; tables are created by the server |

Ubuntu / Debian:

```bash
sudo apt install build-essential cmake libmysqlclient-dev libssl-dev
```

## Build

```bash
git clone https://github.com/hatmore/PlayerServer.git
cd PlayerServer
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
```

Alternatively open `EPlayerServer.vcxproj` in Visual Studio and build/debug on a Linux host through the "Linux remote" workflow.

## Configure and run

Database connection parameters live in `BusinessProcess()` in `include/EdoyunPlayerServer.h` (host / user / password / port / db). The user table in the `edoyun` database is created on first run. The listen address and port are the default arguments of `CServer::Init` (`0.0.0.0:9999`).

```bash
cd build
mkdir -p log          # the logging process writes to ./log/
./EPlayerServer
```

## HTTP API

### Login

```
GET /login?time=<timestamp>&salt=<random salt>&user=<user name>&sign=<signature>
```

The server looks up the user's password and computes

```
sign = MD5(time + MD5_KEY + password + salt)
```

and compares it with the `sign` parameter. The response is `HTTP/1.1 200 OK` with a JSON body:

```json
{ "status": 0, "message": "success" }
```

A non-zero `status` means failure; `message` explains why.

## Logging

The `TRACEI / TRACEE / TRACED / TRACEW` macros and the stream-style `LOGI / LOGE` macros send records over the Unix socket `./log/server.sock` to the logging process, which writes `./log/<start time>.log`. `DUMPD(data, size)` hex-dumps a memory block.

## License

No license has been declared for this project yet. Third-party code under `third_party/` keeps its original license.
