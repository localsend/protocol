# LocalSend Protocol v1

## Table of Contents

- [1. Defaults](#1-defaults)
- [2. Discovery](#2-discovery)
    - [2.1 Multicast](#21-multicast-udp-default)
    - [2.2 HTTP](#22-http-legacy-mode)
- [3. File transfer](#3-file-transfer-http)
    - [3.1 Send request](#31-send-request-metadata-only)
    - [3.2 Send file](#32-send-file)
    - [3.3 Cancel](#33-cancel)

## 1. Defaults

LocalSend does not require a specific port or multicast address but instead provides a default configuration.

Everything can be configured in the app settings if the port / address is somehow unavailable.

The default multicast group is 224.0.0.0/24 because some Android devices reject any other multicast group.

**Multicast (UDP)**
- Port: 53317
- Address: 224.0.0.167

**HTTP (TCP)**
- Port: 53317

## 2. Discovery

### 2.1 Multicast UDP (Default)

At the start of the app, the following message will be sent to the multicast group:

```json5
{
  "alias": "Nice Orange",
  "deviceModel": "Samsung", // nullable
  "deviceType": "mobile", // mobile | desktop | web
  "fingerprint": "random string",
  "announcement": true
}
```

Other LocalSend members will notice this message and will reply with their respective information:

```json5
{
  "alias": "Secret Banana",
  "deviceModel": "Windows",
  "deviceType": "desktop",
  "fingerprint": "random string",
  "announcement": false
}
```

The `fingerprint` is only used to avoid self-discovering.

A response is only triggered when `announcement` is `true`.

### 2.2 HTTP (Legacy Mode)

This method should be used when multicast was unsuccessful.

Devices are discovered by sending this request to all local IP addresses.

The `fingerprint` parameter is optional and is only used to avoid self-discovering.

In this case, the server may respond with an error code if sender and receiver fingerprints match.

The fingerprint is generated on each app start randomly.

`GET /api/localsend/v1/info?fingerprint=abc`

Response

```json5
{
  "alias": "Nice Orange",
  "deviceModel": "Samsung", // nullable
  "deviceType": "mobile" // mobile | desktop | web
}
```

## 3. File transfer (HTTP)

### 3.1 Send Request (Metadata only)

Sends only the metadata to the receiver.

The receiver will decide if this request gets accepted, partially accepted or rejected.

Be aware that only one active session is allowed, i.e. the server will reject all requests when another session is ongoing.

`POST /api/localsend/v1/send-request`

Request

```json5
{
  "info": {
    "alias": "Nice Orange",
    "deviceModel": "Samsung", // nullable
    "deviceType": "mobile" // mobile | desktop | web
  },
  "files": {
    "some file id": {
      "id": "some file id",
      "fileName": "my image.png",
      "size": 324242, // bytes
      "fileType": "image", // image | video | pdf | text | other
      "preview": "*preview data*" // nullable
    },
    "another file id": {
      "id": "another file id",
      "fileName": "another image.jpg",
      "size": 1234,
      "fileType": "image",
      "preview": "*preview data*"
    }
  }
}
```

Response

```json5
{
  "some file id": "some token",
  "another file id": "some other token"
}
```

### 3.2 Send File

The file transfer.

Use the `fileId` and its file-specific `token` from `/send-request`.

This route can be called in parallel.

`POST /api/localsend/v1/send?fileId=some file id&token=some token`

Request

```text
Binary data
```

Response

```text
No body
```

### 3.3 Cancel

This route will be called when the sender wants to cancel the session.

The server will use the IP address to determine if the request is legitimate.

`POST /api/localsend/v1/cancel`

Response

```text
No body
```
