# LocalSend Protocol v1

## Discovery

> Devices are discovered by sending this request to all local IP addresses.

`GET /api/localsend/v1/info`

Response

```json5
{
  "alias": "Nice Orange",
  "deviceModel": "Samsung", // nullable
  "deviceType": "mobile" // mobile | desktop | web
}
```

## Send Request

> Sends the metadata to the receiver.
> 
> The receiver will decide if this request gets accepted, partially accepted or rejected.
> 
> Be aware that only one active session is allowed, i.e. the server will reject all requests when another session is ongoing.

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
    }
  }
}
```

Response

```json5
{
  "some file id": "some token"
}
```

## Send Data

> The file transfer.
> 
> Use the `fileId` and its file-specific `token` from `/send-request`.
> 
> This route can be called in parallel.

`POST /api/localsend/v1/send?fileId=some file id&token=some token`

Request

```text
Binary data
```

Response

```text
No body
```

## Cancel

> This route will be called when the sender wants to cancel the session.
>
> The server will use the IP address to determine if the request is legitimate.

`POST /api/localsend/v1/cancel`

Response

```text
No body
```
