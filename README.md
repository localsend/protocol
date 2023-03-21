# LocalSend Protocol v3

## Table of Contents

- [1. Defaults](#1-defaults)
- [2. Fingerprint](#2-fingerprint)
- [3. Discovery](#3-discovery)
    - [3.1 Multicast](#31-multicast-udp-default)
    - [3.2 HTTP](#32-http-legacy-mode)
- [4. File transfer](#4-file-transfer-http)
    - [4.1 Send request](#41-send-request-metadata-only)
    - [4.2 Send file](#42-send-file)
    - [4.3 Cancel](#43-cancel)
- [5. Additional API](#5-additional-api)
  - [5.1 Info](#51-info)
- [6. Login authentication ways](6-login-authentication-ways)
  - [6.1 Workflow](#51-info)
  - [6.1.1 OpenId](#51-info)
  - [6.1.2 OAuth](#51-info)
  - [6.1.3 Automatic Public/Private Key](#51-info)
  - [6.1.4 TouchId](#51-info)
  - [6.2 Info](#51-info)
  - [6.3 Use Case](#51-info)

## 1. Defaults

LocalSend does not require a specific port or multicast address but instead provides a default configuration.

Everything can be configured in the app settings if the port / address is somehow unavailable.

The default multicast group is 224.0.0.0/24 because some Android devices reject any other multicast group.

**Multicast (UDP)**
- Port: 53317
- Address: 224.0.0.167

**HTTP (TCP)**
- Port: 53317

## 2. Fingerprint

The fingerprint is used to avoid self-discovery and to remember devices.

When encryption is on (HTTPS), then the fingerprint is the SHA-256 hash of the certificate.

When encryption is off (HTTP), then the fingerprint is a random generated string.

## 3. Discovery

### 3.1 Multicast UDP (Default)

**Announcement**

At the start of the app, the following message will be sent to the multicast group:

```json5
{
  "alias": "Nice Orange",
  "deviceModel": "Samsung", // nullable
  "deviceType": "mobile", // mobile | desktop | web
  "fingerprint": "random string",
  "announce": true
}
```

**Response**

Other LocalSend members will notice this message and will reply with their respective information.

First, an HTTP/TCP request is sent to the origin:

`POST /api/localsend/v2/register`

```json5
{
  "alias": "Secret Banana",
  "deviceModel": "Windows",
  "deviceType": "desktop",
  "fingerprint": "random string" // ignored in HTTPS mode
}
```

As fallback, members can also respond with a Multicast/UDP message.

```json5
{
  "alias": "Secret Banana",
  "deviceModel": "Windows",
  "deviceType": "desktop",
  "fingerprint": "random string",
  "announce": false
}
```

The `fingerprint` is only used to avoid self-discovering.

A response is only triggered when `announce` is `true`.

### 3.2 HTTP (Legacy Mode)

This method should be used when multicast was unsuccessful.

Devices are discovered by sending this request to all local IP addresses.

`POST /api/localsend/v2/register`

Request

```json5
{
  "alias": "Secret Banana",
  "deviceModel": "Windows",
  "deviceType": "desktop",
  "fingerprint": "random string" // ignored in HTTPS mode
}
```

Response

```json5
{
  "alias": "Nice Orange",
  "deviceModel": "Samsung",
  "deviceType": "mobile",
  "fingerprint": "random string" // ignored in HTTPS mode
}
```

## 4. File transfer (HTTP)

### 4.1 Send Request (Metadata only)

Sends only the metadata to the receiver.

The receiver will decide if this request gets accepted, partially accepted or rejected.

`POST /api/localsend/v2/send-request`

Request

```json5
{
  "info": {
    "alias": "Nice Orange",
    "deviceModel": "Samsung", // nullable
    "deviceType": "mobile", // mobile | desktop | web
    "fingerprint": "random string" // ignored in HTTPS mode
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
  "sessionId": "mySessionId",
  "files": {
    "someFileId": "someFileToken",
    "someOtherFileId": "someOtherFileToken"
  }
}
```

### 4.2 Send File

The file transfer.

Use the `sessionId`, the `fileId` and its file-specific `token` from `/send-request`.

This route can be called in parallel.

`POST /api/localsend/v2/send?sessionId=mySessionId&fileId=someFileId&token=someFileToken`

Request

```text
Binary data
```

Response

```text
No body
```

### 4.3 Cancel

This route will be called when the sender wants to cancel the session.

Use the `sessionId` from `/send-request`.

`POST /api/localsend/v2/cancel?sessionId=mySessionId`

Response

```text
No body
```

## 5. Additional API

### 5.1 Info

This was an old route previously used for discovery. This has been replaced with `/register` which is a two-way discovery.

Now this route should be only used for debugging purposes.

`GET /api/localsend/v2/info`

Response

```json5
{
  "alias": "Nice Orange",
  "deviceModel": "Samsung", // nullable
  "deviceType": "mobile" // mobile | desktop | web
}
```

## 6. Login authentication way

There are multiple ways and means to authenticate a device besides "accept" and "decline". I'll explain this in the part where you ask about the login process workflow.  

But if I need to try to convince I would say something like: "There are several ways to authenticate in addition to "accept" or "decline", the proposal here aims to say that the localsend protocol can use these forms of authentication to give greater control, security to its users, collaborators, partners or even general solutions that uses the localsend network protocol daily to send and receive your files locally."

All the ways of login I mentioned can be complementary or alternative to "accept" or "decline". 

## 6.1 workflow
IMHO(In my humble opinion), it would be interesting to provide more forms of login or authentication for the user. Bear in mind that there may be more or less technical users who would like a more diversified login process, with greater possibilities.

For example, you can accept the device by logging into your Google account, Facebook, etc. Most users use social networks, it would be easier for most users to use authentication method like OAuthn2 or OAuthn3 to authenticate a certain device than a simple "accept" or "decline".

Another example, localsend could use a public and private key. For example, localsend generates a public key and requests a generated private key. Whenever the user needs to authenticate a connection to another device, he looks up that device with a public key, and when he needs to connect to that device, the user signs with a private key. This example is something similar to what nostr does to send and receive messages from the user, the user has a public key where anyone sends a message and when he writes the message, he signs it with a private key. Something localsend could be inspired by and have, this can increases greater transparency for user and network protocol.

But today the process of various types of authentication factors or ways is what makes authentication more secure. Today there are things like passwordless, web-authn, OpenId, TouchId, 2 or more factor authentication etc. Below I will leave examples of json format with the authentication method to make it easier to understand. Initially, I will only show initial code examples and then the use case for each one.

### 6.1.1 OpenId
If you work in any company, and your company uses OpenId protocol to authenticate login. It would make sense for localsend to use OpenId for authentication.

```json5
{
  "info": {
    "alias": "Nice Orange",
    "device-type-auth": "OpenId", 
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

### 6.1.2 OAuth
If you want to connect to a local device with people you know locally, an interesting option is to use the OAuth protocol. Most of the social sites you access already use the OAuthn standard in version OAuthn1, OAuthn2 or OAuthn3. The interesting thing about this option is that you could have a list of devices that you have accessed recently or in the past. For authentication to work, both devices must have some social network for login authentication. This social network must use the OAuth protocol to validate the login.

```json5
{
  "info": {
    "alias": "Nice Orange",
    "device-type-auth": "OAuth", 
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

### 6.1.3 Automatic Public/Private Key
If you would like to connect to a device locally with greater security, control could use a public key and a private key. Both the public and private key are automatically generated by localsend, the private key is for you to authenticate the device you want to authenticate. The public key is for your device to be found on the local network.

```json5
{
  "info": {
    "alias": "Nice Orange",
    "device-type-auth": "automatic public/private key"
    "device-id-public": "random string", 
    "device-id-private": "random string", 
    "deviceModel": "Samsung", // nullable
    "deviceType": "mobile" // mobile | desktop | web
  },
  "files": {
    "some file id": {
      "id": "some file id",
      "fileName": "my image.png",
      "size": 324242, // bytes
      "fileType": "image", // image | video | pdf | text | other
      "preview": "*preview data*", // nullable
      "source-file": device-id-public,
      "auth-file": device-id-private
    },
    "another file id": {
      "id": "another file id",
      "fileName": "another image.jpg",
      "size": 1234,
      "fileType": "image",
      "preview": "*preview data*",
       "source-file": device-id-public,
       "auth-file": device-id-private
    }
  }
}
```

### 6.1.4 TouchId
Sometimes you just want to authenticate the device with something biometric for some personal security reason. There are bluetooth versions that work with an auto-generated password or touch-id.
```json5
{
  "info": {
    "alias": "Nice Orange",
    "device-type-auth": "TouchId", 
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

### 6.2 Info
"device-type-authn" it describes which form of authentication is used: OAuth, OpenId, TouchId, Automatic Private/Public key. In the case of the "Private/Public key" authentication type, a "device-id-public", "device-id-private" which is information that the user must save locally for later use to authenticate files.

But in case of elevated control, you don't need "device-id-public", "device-id-private" to authenticate files, as this is sent remotely to the local network with the information of the device you connected to with OAuth, OpenId, TouchId or Standard Auth ("accept" or "decline").

### 6.3 use case
OAuth, OpenId and TouchId rely on a third party service for verification. This may be considered in the future but LocalSend should be self reliant for now. 

That's true, things like OAuth, OpenId and TouchId depend on a third-party service. But, I would like to comment briefly that although they are authentication protocols that need third party services to work, some people can create "login extensions" in localsend. For example, if there is any way to use these libraries (react-native-touch-id, OpenID/OAuth2) in localsend and generate a custom way or schema of login that would be very interesting and feasible.

Having some public/private key mechanism is good. Because LocalSend already uses a self-signed certificate for HTTPS encryption, we can just use the certificate to remember devices. This is a good idea. Perhaps this resource can help with this issue: [Sent history](https://github.com/localsend/localsend/issues/215)
