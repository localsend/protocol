# LocalSend 协议 v2

[English](./README.md) | **简体中文**

主要为了实现一个不依赖于任何外部服务器的简单 REST 协议。

因为计算机网络比较复杂，因此我们不能假设每种方法都可用。某些设备可能不支持多点广播或不能运行 HTTP 服务器。

就是这个原因，所以这协议尝试用多种方式去发现其他 LocalSend 成员，从而执行发送文件。

该协议只需要一方建立 HTTP 服务器即可运行。

## 目录

- [LocalSend 协议 v2](#localsend-协议-v2)
  - [目录](#目录)
  - [1. 默认配置](#1-默认配置)
  - [2. 指纹](#2-指纹)
  - [3. 搜寻发现](#3-搜寻发现)
    - [3.1 UDP 广播](#31-udp-广播)
    - [3.2 HTTP (传统模式)](#32-http-传统模式)
  - [4. 文件传输 (HTTP)](#4-文件传输-http)
    - [4.1 准备工作 (仅元数据)](#41-准备工作-仅元数据)
    - [4.2 发送文件](#42-发送文件)
    - [4.3 取消](#43-取消)
  - [5. 反向文件传输 (HTTP)](#5-反向文件传输-http)
    - [5.1 浏览器 URL](#51-浏览器-url)
    - [5.2 接收请求](#52-接收请求)
    - [5.3 接收文件](#53-接收文件)
  - [6. 其他 API](#6-其他-api)
    - [6.1 信息](#61-信息)
  - [7. 枚举](#7-枚举)
    - [7.1 设备类型](#71-设备类型)

## 1. 默认配置

LocalSend 不需要特定的端口或广播地址，而是提供默认配置。

如果端口/地址不可用，可以在应用程序中修改配置。

默认广播地址是 224.0.0.0/24，因为某些 Android 设备禁止其他广播组。

**UDP广播**

- 端口: 53317
- 地址: 224.0.0.167

**HTTP (TCP)**

- 端口: 53317

## 2. 指纹

指纹用于区分设备。

使用 HTTPS 加密时，证书的 SHA-256 哈希值作为指纹。

关闭 HTTPS 加密时，用随机字符串作为指纹。

## 3. 搜寻发现

### 3.1 UDP 广播

**通知**

在应用程序启动时，会发送以下消息到广播地址：

```json5
{
  "alias": "Nice Orange",
  "version": "2.0", // 协议版本（major.minor）
  "deviceModel": "Samsung", // nullable
  "deviceType": "mobile", // mobile | desktop | web | headless | server, nullable
  "fingerprint": "随机字符串",
  "port": 53317,
  "protocol": "https", // http | https
  "download": true, // 下载 API（5.2 和 5.3）是否激活（可选，默认为 false）
  "announce": true
}
```

**返回**

其他 LocalSend 成员会收到到此消息并回复各自的设备信息。

首先，向原设备发送 HTTP/TCP 请求：

`POST /api/localsend/v2/register`

```json5
{
  "alias": "Secret Banana",
  "version": "2.0",
  "deviceModel": "Windows",
  "deviceType": "desktop",
  "fingerprint": "随机字符串", // 在 HTTPS 模式下被忽略
  "port": 53317,
  "protocol": "https",
  "download": true, // 下载 API（5.2 和 5.3）是否激活（可选，默认为 false）
}
```

另外，成员还可以使用 UDP 广播消息进行响应。

```json5
{
  "alias": "Secret Banana",
  "version": "2.0",
  "deviceModel": "Windows",
  "deviceType": "desktop",
  "fingerprint": "随机字符串",
  "port": 53317,
  "protocol": "https",
  "download": true,
  "announce": false,
}
```

`fingerprint` 仅用于区分自身设备。

只有当 `announce` 为 `true` 时，才会触发响应。

### 3.2 HTTP (传统模式)

当广播不成功时应使用此方法。

向本地所有 IP 地址发送此请求来发现设备。

`POST /api/localsend/v2/register`

请求：

```json5
{
  "alias": "Secret Banana",
  "version": "2.0", // 协议版本（major.minor）
  "deviceModel": "Windows",
  "deviceType": "desktop",
  "fingerprint": "随机字符串", // 在 HTTPS 模式下被忽略
  "port": 53317,
  "protocol": "https", // http | https
  "download": true, // 下载 API（5.2 和 5.3）是否激活（可选，默认为 false）
}
```

返回：

```json5
{
  "alias": "Nice Orange",
  "version": "2.0",
  "deviceModel": "Samsung",
  "deviceType": "mobile",
  "fingerprint": "随机字符串", // 在 HTTPS 模式下被忽略
  "download": true, // 下载 API（5.2 和 5.3）是否激活（可选，默认为 false）
}
```

## 4. 文件传输 (HTTP)

HTTP传输是默认方式。

接收方作为 HTTP 服务器。

发送方（即 HTTP 客户端）将文件发送到 HTTP 服务器。

### 4.1 准备工作 (仅元数据)

仅将元数据发送给接收方。

接收方将决定该请求是否被接受、部分接受或拒绝。

`POST /api/localsend/v2/prepare-upload`

请求：

```json5
{
  "info": {
    "alias": "Nice Orange",
    "version": "2.0", // 协议版本 (major.minor)
    "deviceModel": "Samsung", // 可为空
    "deviceType": "mobile", // mobile | desktop | web | headless | server, 可为空
    "fingerprint": "随机字符串", // 在 HTTPS 模式下被忽略
    "port": 53317,
    "protocol": "https", // http | https
    "download": true, // 下载 API (5.2 和 5.3) 是否激活 (可选，默认为 false)
  },
  "files": {
    "文件ID": {
      "id": "文件ID",
      "fileName": "我的图片.png",
      "size": 324242, // bytes
      "fileType": "image/jpeg",
      "sha256": "*sha256哈希值*", // 可为空
      "preview": "*预览数据*" // 可为空
    },
    "另一个文件ID": {
      "id": "另一个文件ID",
      "fileName": "另一个图片.jpg",
      "size": 1234,
      "fileType": "image/jpeg",
      "sha256": "*sha256哈希值*",
      "preview": "*预览数据*"
    }
  }
}
```

返回：

```json5
{
  "sessionId": "我的会话ID",
  "files": {
    "文件ID": "文件Token",
    "另一个文件ID": "另一个文件Token"
  }
}
```

Errors 错误代码

| HTTP 代码 | 信息                |
| --------- | ------------------- |
| 204       | 完成 (无需传输文件) |
| 400       | 无效请求体          |
| 403       | 拒绝                |
| 500       | 接收器未知错误      |

### 4.2 发送文件

文件传输。

使用来自 `/prepare-upload` 的 `sessionId`、`fileId` 及其特定于文件的 `token`。

这请求路径可以同时调用。

`POST /api/localsend/v2/upload?sessionId=我的会话ID&fileId=文件ID&token=文件Token`

请求：

```text
二进制数据
```

返回：

```text
无响应体
```

Errors 错误代码

| HTTP 代码 | 信息               |
| --------- | ------------------ |
| 400       | 参数缺失           |
| 403       | 无效令牌或 IP 地址 |
| 409       | 被其他会话阻止     |
| 500       | 接收器未知错误     |

### 4.3 取消

当发送方希望取消会话时，可用此请求路径。

使用 `/send-request` 中的 `sessionId`。

`POST /api/localsend/v2/cancel?sessionId=我的会话ID`

返回：

```text
无响应体
```

## 5. 反向文件传输 (HTTP)

HTTP方式是当接收方无法使用 LocalSend 时应使用的备用方法。

发送方设置一个 HTTP 服务器，通过提供 URL 将文件发送给其他成员。

然后接收方使用给定的 URL 打开浏览器并下载文件。

需要注意的是，由于浏览器拒绝自签名证书，因此使用未加密的 HTTP 协议。

### 5.1 浏览器 URL

接收方可以在浏览器中打开以下网址来下载文件。

```text
http://<发送者ip>:<发送者端口>
```

### 5.2 接收请求

向发送方发送请求以获取文件元数据列表。

下载方可以添加 `?sessionId=我的会话ID`。此时，如果是同一个会话，则接受该请求。

如果用户刷新浏览器页面，则需要这样操作。

`POST /api/localsend/v2/prepare-download`

请求：

```text
无请求体
```

返回：

```json5
{
  "info": {
    "alias": "Nice Orange",
    "version": "2.0",
    "deviceModel": "Samsung", // 可为空
    "deviceType": "mobile", // mobile | desktop | web | headless | server, 可为空
    "fingerprint": "随机字符串", //  在 HTTPS 模式下被忽略
    "download": true, // 下载 API (5.2 和 5.3) 是否激活 (可选，默认为 false)
  },
  "sessionId": "mySessionId",
  "files": {
    "文件ID": {
      "id": "文件ID",
      "fileName": "我的图片.png",
      "size": 324242, // bytes
      "fileType": "image/jpeg",
      "sha256": "*sha256哈希值*", // 可为空
      "preview": "*预览数据*" // 可为空
    },
    "另一个文件ID": {
      "id": "另一个文件ID",
      "fileName": "另一个图片.jpg",
      "size": 1234,
      "fileType": "image/jpeg",
      "sha256": "*sha256哈希值*",
      "preview": "*预览数据*"
    }
  }
}
```

### 5.3 接收文件

文件传输。

使用 `/receive-request` 中的 `sessionId` 和 `fileId`。

这请求路径可以同时调用。

`GET /api/localsend/v2/download?sessionId=我的会话ID&fileId=文件ID`

请求：

```text
无请求体
```

返回：

```text
二进制数据
```

## 6. 其他 API

### 6.1 信息

这是一条以前用于发现的旧连接。 它已被替换为“/register”，这是一种双向发现。

现在该路由应该仅用于调试。

`GET /api/localsend/v2/info`

返回：

```json5
{
  "alias": "Nice Orange",
  "version": "2.0",
  "deviceModel": "Samsung", // 可为空
  "deviceType": "mobile", // mobile | desktop | web | headless | server, 可为空
  "fingerprint": "随机字符串",
  "download": true, // 下载 API (5.2 和 5.3) 是否激活 (可选，默认为 false)
}
```

## 7. 枚举

在这项目中，枚举用于定义某些字段的可能值。

### 7.1 设备类型

设备类型仅用于 UI 使用，例如显示图标。

各种设备类型的协议都一样。

| 值       | 描述                            |
| -------- | ------------------------------- |
| mobile   | 移动设备 (Android, iOS, FireOS) |
| desktop  | 桌面 (Windows, macOS, Linux)    |
| web      | 网页浏览器 (Firefox, Chrome)    |
| headless | 在终端运行的无图形用户界面程序  |
| server   | 全天候运行的（自托管）云服务    |

如有枚举外的值，都返回 `desktop` 类型。
