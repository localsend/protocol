# LocalSend Protocol v2

The goal is to have a simple REST protocol that does not rely on any external servers.<br>
主要为了实现一个不依赖于任何外部服务器的简单 REST 协议。

Because computer networks can be complicated, we cannot assume that every method is available.
Some devices might not support multicast or might not be allowed to have an HTTP server running.<br>
因为计算机网络比较复杂，因此我们不能假设每种方法都可用。某些设备可能不支持多点广播或不能运行 HTTP 服务器。

This is why this protocol tries to be "clever" and uses multiple methods to discover and to send files to other LocalSend members.<br>
就是这个原因，所以这协议尝试用多种方式去发现其他 LocalSend 成员，从而执行发送文件。

The protocol only needs one party to set up an HTTP server.<br>
该协议只需要一方建立 HTTP 服务器即可运行。

## Table of Contents
## 目录

- [1. Defaults](#1-defaults) 默认配置
- [2. Fingerprint](#2-fingerprint) 指纹
- [3. Discovery](#3-discovery) 搜寻发现
    - [3.1 Multicast](#31-multicast-udp-default) 广播
    - [3.2 HTTP](#32-http-legacy-mode) HTTP
- [4. File transfer](#4-file-transfer-http) 文件传输
  - [4.1 Send request](#41-send-request-metadata-only) 发送请求
  - [4.2 Send file](#42-send-file) 发送文件
  - [4.3 Cancel](#43-cancel) 取消
- [5. Reverse File transfer](#5-reverse-file-transfer-http) 反向文件传输
  - [5.1 Browser URL](#51-browser-url) 浏览器 URL
  - [5.2 Receive request](#52-receive-request-metadata-only) 接收请求
  - [5.3 Receive file](#53-receive-file) 接收文件
- [6. Additional API](#6-additional-api) 其他 API
  - [6.1 Info](#61-info) 关于

## 1. Defaults 默认配置

LocalSend does not require a specific port or multicast address but instead provides a default configuration.<br>
LocalSend 不需要特定的端口或广播地址，而是提供默认配置。

Everything can be configured in the app settings if the port / address is somehow unavailable.<br>
如果端口/地址不可用，可以在应用程序中修改配置。

The default multicast group is 224.0.0.0/24 because some Android devices reject any other multicast group.<br>
默认广播地址是 224.0.0.0/24，因为某些 Android 设备禁止其他广播组。

**Multicast (UDP) UDP广播**

- Port: 53317
- Address: 224.0.0.167

**HTTP (TCP)**
**HTTP（TCP）**
- Port: 53317

## 2. Fingerprint 指纹

The fingerprint is used to avoid self-discovery and to remember devices.<br>
指纹用于区分设备。

When encryption is on (HTTPS), then the fingerprint is the SHA-256 hash of the certificate.
使用 HTTPS 加密时，证书的 SHA-256 哈希值作为指纹。

When encryption is off (HTTP), then the fingerprint is a random generated string.
当关闭 HTTP 加密时，用随机字符串作为指纹。

## 3. Discovery 搜寻发现

### 3.1 Multicast UDP (Default) UDP 广播（默认）

**Announcement 通知**

At the start of the app, the following message will be sent to the multicast group:<br>
在应用程序启动时，会发送以下消息到广播地址：

```json5
{
  "alias": "Nice Orange",
  "version": "2.0", // protocol version (major.minor)
  "deviceModel": "Samsung", // nullable
  "deviceType": "mobile", // mobile | desktop | web | headless | server, nullable
  "fingerprint": "random string",
  "port": 53317,
  "protocol": "https", // http | https
  "download": true, // if the download API (5.2 and 5.3) is active (optional, default: false)
  "announce": true
}
```

**Response 返回**

Other LocalSend members will notice this message and will reply with their respective information.<br>
其他 LocalSend 成员会收到到此消息并回复各自的设备信息。

First, an HTTP/TCP request is sent to the origin:<br>
首先，发送 HTTP/TCP 请求：

`POST /api/localsend/v2/register`

```json5
{
  "alias": "Secret Banana",
  "version": "2.0",
  "deviceModel": "Windows",
  "deviceType": "desktop",
  "fingerprint": "random string", // ignored in HTTPS mode
  "port": 53317,
  "protocol": "https",
  "download": true, // if the download API (5.2 and 5.3) is active (optional, default: false)
}
```

As fallback, members can also respond with a Multicast/UDP message.<br>
另外，成员还可以使用 UDP 广播消息进行响应。

```json5
{
  "alias": "Secret Banana",
  "version": "2.0",
  "deviceModel": "Windows",
  "deviceType": "desktop",
  "fingerprint": "random string",
  "port": 53317,
  "protocol": "https",
  "download": true,
  "announce": false,
}
```

The `fingerprint` is only used to avoid self-discovering.<br>
“指纹”仅用于区分自身设备。

A response is only triggered when `announce` is `true`.<br>
仅当“announce”为“true”时才会响应。

### 3.2 HTTP (Legacy Mode) HTTP 传统模式

This method should be used when multicast was unsuccessful.<br>
当广播不成功时应使用此方法。

Devices are discovered by sending this request to all local IP addresses.<br>
向本地所有 IP 地址发送此请求来发现设备。

`POST /api/localsend/v2/register`

Request 请求

```json5
{
  "alias": "Secret Banana",
  "version": "2.0", // protocol version (major.minor)
  "deviceModel": "Windows",
  "deviceType": "desktop",
  "fingerprint": "random string", // ignored in HTTPS mode
  "port": 53317,
  "protocol": "https", // http | https
  "download": true, // if the download API (5.2 and 5.3) is active (optional, default: false)
}
```

Response 返回

```json5
{
  "alias": "Nice Orange",
  "version": "2.0",
  "deviceModel": "Samsung",
  "deviceType": "mobile",
  "fingerprint": "random string", // ignored in HTTPS mode
  "download": true, // if the download API (5.2 and 5.3) is active (optional, default: false)
}
```

## 4. File transfer (HTTP) HTTP 文件传输

This is the default method.<br>
这是默认方式。

The receiver setups the HTTP server.<br>
接收方作为 HTTP 服务器。

The sender (i.e. HTTP client) sends files to the HTTP server.<br>
发送方（即 HTTP 客户端）将文件发送到 HTTP 服务器。

### 4.1 Preparation (Metadata only) 准备工作（仅限元数据）

Sends only the metadata to the receiver.<br>
仅将元数据发送给接收方。

The receiver will decide if this request gets accepted, partially accepted or rejected.<br>
接收方将决定该请求是否被接受、部分接受或拒绝。

`POST /api/localsend/v2/prepare-upload`

Request 请求

```json5
{
  "info": {
    "alias": "Nice Orange",
    "version": "2.0", // protocol version (major.minor)
    "deviceModel": "Samsung", // nullable
    "deviceType": "mobile", // mobile | desktop | web | headless | server, nullable
    "fingerprint": "random string", // ignored in HTTPS mode
    "port": 53317,
    "protocol": "https", // http | https
    "download": true, // if the download API (5.2 and 5.3) is active (optional, default: false)
  },
  "files": {
    "some file id": {
      "id": "some file id",
      "fileName": "my image.png",
      "size": 324242, // bytes
      "fileType": "image/jpeg",
      "sha256": "*sha256 hash*", // nullable
      "preview": "*preview data*" // nullable
    },
    "another file id": {
      "id": "another file id",
      "fileName": "another image.jpg",
      "size": 1234,
      "fileType": "image/jpeg",
      "sha256": "*sha256 hash*",
      "preview": "*preview data*"
    }
  }
}
```

Response 返回

```json5
{
  "sessionId": "mySessionId",
  "files": {
    "someFileId": "someFileToken",
    "someOtherFileId": "someOtherFileToken"
  }
}
```

Errors 错误代码

| HTTP code | Message                            |
|-----------|------------------------------------|
| 204       | Finished (No file transfer needed) |
| 400       | Invalid body                       |
| 403       | Rejected                           |
| 500       | Unknown error by receiver          |
 
### 4.2 Send File 发送文件

The file transfer.<br>
文件传输。

Use the `sessionId`, the `fileId` and its file-specific `token` from `/prepare-upload`.<br>
使用“/prepare-upload”中的“sessionId”、“fileId”及其特定于文件的“token”。

This route can be called in parallel.<br>
这请求路径可以同时调用。

`POST /api/localsend/v2/upload?sessionId=mySessionId&fileId=someFileId&token=someFileToken`

Request 请求

```text
Binary data
```

Response 返回

```text
No body
```

Errors 错误代码

| HTTP code | Message                     |
|-----------|-----------------------------|
| 400       | Missing parameters          |
| 403       | Invalid token or IP address |
| 409       | Blocked by another session  |
| 500       | Unknown error by receiver   |

### 4.3 Cancel 取消

This route will be called when the sender wants to cancel the session.<br>
当发送方希望取消会话时，可用此请求路径。

Use the `sessionId` from `/send-request`.<br>
使用“/send-request”中的“sessionId”。

`POST /api/localsend/v2/cancel?sessionId=mySessionId`

Response 返回

```text
No body
```

## 5. Reverse file transfer (HTTP) 反向文件传输（HTTP）

This is an alternative method which should be used when LocalSend is not available on the receiver.<br>
这是当接收方无法使用 LocalSend 时应使用的备用方法。

The sender setups an HTTP server to send files to other members by providing a URL.<br>
发送方设置一个 HTTP 服务器，通过提供 URL 将文件发送给其他成员。

The receiver then opens the browser with the given URL and downloads the file.<br>
然后接收方使用给定的 URL 打开浏览器并下载文件。

It is important to note that the unencrypted HTTP protocol is used because browsers reject self-signed certificates.<br>
需要注意的是，由于浏览器拒绝自签名证书，因此使用未加密的 HTTP 协议。

### 5.1 Browser URL 浏览器 URL

The receiver can open the following URL in the browser to download the file.<br>
接收方可以在浏览器中打开以下网址来下载文件。

```text
http://<sender-ip>:<sender-port>
```

### 5.2 Receive Request (Metadata only) 接收请求（仅限元数据）

Send to the sender a request to get a list of file metadata.<br>
向发送方发送请求以获取文件元数据列表。

The downloader may add `?sessionId=mySessionId`. In this case, the request should be accepted if it is the same session.<br>
下载方可以添加`?sessionId=mySessionId`。 此时，如果是同一个会话，则接受该请求。

This is needed if the user refreshes the browser page.<br>
如果用户刷新浏览器页面，则需要这样操作。

`POST /api/localsend/v2/prepare-download`

Request 请求

```text
No body
```

Response 返回

```json5
{
  "info": {
    "alias": "Nice Orange",
    "version": "2.0",
    "deviceModel": "Samsung", // nullable
    "deviceType": "mobile", // mobile | desktop | web | headless | server, nullable
    "fingerprint": "random string", // ignored in HTTPS mode
    "download": true, // if the download API (5.2 and 5.3) is active (optional, default: false)
  },
  "sessionId": "mySessionId",
  "files": {
    "some file id": {
      "id": "some file id",
      "fileName": "my image.png",
      "size": 324242, // bytes
      "fileType": "image/jpeg",
      "sha256": "*sha256 hash*", // nullable
      "preview": "*preview data*" // nullable
    },
    "another file id": {
      "id": "another file id",
      "fileName": "another image.jpg",
      "size": 1234,
      "fileType": "image/jpeg",
      "sha256": "*sha256 hash*",
      "preview": "*preview data*"
    }
  }
}
```

### 5.3 Receive File 接收文件

The file transfer.<br>
文件传输。

Use the `sessionId`, the `fileId` from `/receive-request`.<br>
使用“/receive-request”中的“sessionId”、“fileId”。

This route can be called in parallel.<br>
这请求路径可以同时调用。

`GET /api/localsend/v2/download?sessionId=mySessionId&fileId=someFileId`

Request 请求

```text
No body
```

Response 返回

```text
Binary data
```

## 6. Additional API 其他 API

### 6.1 Info 信息

This was an old route previously used for discovery. This has been replaced with `/register` which is a two-way discovery.<br>
这是一条以前用于发现的旧连接。 它已被替换为“/register”，这是一种双向发现。

Now this route should be only used for debugging purposes.<br>
现在该路由应该仅用于调试。

`GET /api/localsend/v2/info`

Response 返回

```json5
{
  "alias": "Nice Orange",
  "version": "2.0",
  "deviceModel": "Samsung", // nullable
  "deviceType": "mobile", // mobile | desktop | web | headless | server, nullable
  "fingerprint": "random string",
  "download": true, // if the download API (5.2 and 5.3) is active (optional, default: false)
}
```

## 7. Enums 枚举

In this project, enums are used to define the possible values of some fields.<br>
在这项目中，枚举用于定义某些字段的可能值。

### 7.1 Device Type 设备类型

Device types are only used for UI purposes like showing an icon.<br>
设备类型仅用于 UI 使用，例如显示图标。

There is no difference in the protocol between the different device types.<br>
各种设备类型的协议都一样。

| Value    | Description                               |
|----------|-------------------------------------------|
| mobile   | mobile device (Android, iOS, FireOS)      |
| desktop  | desktop (Windows, macOS, Linux)           |
| web      | web browser (Firefox, Chrome)             |
| headless | program without GUI running on a terminal |
| server   | (self-hosted) cloud service running 24/7  |

The implementation handle unknown values. The official implementation falls back to `desktop`.<br>
如有枚举外的值，都返回“desktop”类型。
