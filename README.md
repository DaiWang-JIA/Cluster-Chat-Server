Markdown

# Cluster-Chat-Server (C++ 集群聊天服务器)

![C++11](https://img.shields.io/badge/C++-11-blue.svg)
![Platform](https://img.shields.io/badge/Platform-Linux-green.svg)
![Tech](https://img.shields.io/badge/Tech-Muduo%20%7C%20Redis%20%7C%20MySQL-orange.svg)
![Infra](https://img.shields.io/badge/Infra-Nginx%20LB-lightgrey.svg)

**Cluster-Chat-Server** 是一个基于 C++11 的高可用、可水平扩展的集群聊天系统。

本项目采用 `Nginx` + `Muduo` + `Redis` + `MySQL` 的核心技术架构，实现了一个分层解耦的后端服务，并配备了功能完整的 C++ 命令行客户端。

---

## 🚀 核心功能

* **基础功能**：用户注册、登录。
* **私聊功能**：一对一聊天、好友添加。
* **群聊功能**：创建群组、加入群组、群组聊天。
* **离线消息**：支持私聊和群聊的离线消息存储与拉取。
* **集群架构**：支持 `ChatServer` 业务服务器的水平扩展与负载均衡。
* **跨服通信**：支持跨服务器实例的消息转发（私聊和群聊）。

---

## 🛠️ 技术栈

* **编程语言**：`C++11`
* **网络库**：`Muduo` (基于`one loop per thread`模型的 C++ Reactor 网络库)
* **负载均衡**：`Nginx` (使用 `stream` 模块进行 TCP 负载均衡)
* **消息队列**：`Redis` (使用 `Pub/Sub` 功能实现跨服消息转发)
* **数据库**：`MySQL` (存储用户信息、好友关系、离线消息等)
* **数据格式**：`JSON` (使用 `nlohmann/json` 库进行客户端与服务器的数据序列化)
* **构建工具**：`CMake`
* **操作系统**：`Linux`

---

## 📐 架构设计

本项目在架构上分为**客户端**、**Nginx负载均衡** 和 **C++后端服务集群** 三大块。

### 1. 服务端核心设计 (`chatservice.cpp`)

服务端采用分层设计，`ChatServer`（网络层） 与 `ChatService`（业务层） 完全解耦。

* **消息路由表**：`ChatService` 在启动时，会初始化一个 `std::unordered_map<int, std::function>`。
* **工作流**：
    1.  `chatserver.cpp` 的 `onMessage` 回调收到 `json` 数据包。
    2.  它解析出 `msgid`。
    3.  通过 `msgid` 从 `ChatService` 的 `map` 中 `O(1)` 查找到对应的业务处理器 `handler`。
    4.  执行 `handler`，`ChatServer` 自身不关心任何业务逻辑，实现了高度解耦。

### 2. 客户端核心设计 (`main2.cpp`)

客户端 采用了**UI线程与网络线程分离**的设计：

* **主线程**：负责 `send` 数据和接收用户键盘输入（如`chat:1:hello`）。
* **子线程 (`readTaskHandler`)**：独立的工作线程，**唯一职责**是在 `recv` 处**阻塞**，专门等待服务器的数据。
* **线程同步**：主线程在发送“登录”或“注册”请求后，会调用 `sem_wait(&rwsem)` 阻塞等待。子线程在收到对应的`_ACK`响应并处理完毕后，调用 `sem_post(&rwsem)` 唤醒主线程，实现了精确的同步。

### 3. 集群通信 (Redis Pub/Sub)

本项目使用 Redis 的发布/订阅功能实现跨服务器的消息转发。

* **场景**：用户A（连接在服务器A）给用户B（连接在服务器B）发送消息。
* **流程**：
    1.  **订阅**：用户B登录服务器B时，服务器B的 `ChatService` 会订阅（`SUBSCRIBE`）`userid_B` 频道。
    2.  **发布**：服务器A的 `oneChat` 发现用户B不在本地（`_userConnMap` 未命中），且数据库状态为 `online`，于是向 Redis 发布（`PUBLISH`）`userid_B` 频道，内容为消息 `json`。
    3.  **转发**：服务器B的 Redis 订阅线程收到消息，调用 `handleRedisSubscribeMessage` 回调，将消息转发给用户B的客户端。
* **非阻塞订阅**：`hiredis` 的 `redisGetReply` 订阅是阻塞的。本项目通过创建**两个 `redisContext`**（一个 `publish`，一个 `subscribe`）并启动**独立的订阅线程**（`observer_channel_message`），解决了订阅操作对 `ChatService` 业务线程（Muduo线程）的阻塞问题。

### 4. 高可用部署 (Nginx)

* 项目使用 Nginx 的 `stream` 模块 作为集群的**TCP流量入口**。
* Nginx 负责对后端的多个 `ChatServer` 实例（如 6000, 6001, 6002 端口）进行**负载均衡**。
* 同时，Nginx 会自动进行**健康检查**，如果某台 `ChatServer` 实例崩溃，Nginx 会自动将其从转发表中移除，实现**故障转移**，保证了业务服务器的高可用性。

---

## 🚀 如何运行

### 1. 依赖
* `cmake`
* `mysql-server` / `mysql-client` / `libmysqlclient-dev`
* `redis-server` / `libhiredis-dev`
* `muduo` (需提前编译安装)
* `nginx` (编译时需带 `--with-stream` 模块)

### 2. 数据库配置
1.  启动 `mysql`。
2.  创建 `chat` 数据库：`create database chat;`
3.  在 `db.cpp` 中修改数据库连接信息（`server`, `user`, `password`, `dbname`）。
4.  执行 `db.sql`（需自行根据 `*.model.cpp` 创建）以创建 `user`, `friend`, `groupuser`, `allgroup`, `offlinemessage` 表。

### 3. 编译
```bash
# 假设已在根目录
mkdir build
cd build
cmake ..
make

```


# 演示：
<img width="1272" height="282" alt="image" src="https://github.com/user-attachments/assets/45b588fc-d414-4051-bbcc-03a336fa2a6c" />
<img width="1275" height="246" alt="image" src="https://github.com/user-attachments/assets/2837d860-daea-4e9c-a63a-d5869f1e661b" />
<img width="387" height="141" alt="image" src="https://github.com/user-attachments/assets/83a8f6d1-c11f-4fff-90f3-b704a0ee13bd" />


<img width="628" height="523" alt="image" src="https://github.com/user-attachments/assets/bfde89cd-081f-42b5-844d-04ae57e16e53" />
<img width="677" height="546" alt="image" src="https://github.com/user-attachments/assets/abb726c9-240d-422f-9c33-352fa1f058a3" />




