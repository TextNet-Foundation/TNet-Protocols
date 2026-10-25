# TNet Protocols Specification

This document describes the three core protocols of **TNet** — a decentralized text network:

- **UTNP** — Universal Text Negotiation Protocol
- **TMUTP** — TMkUp Transfer Protocol
- **TBIP** — TNet Binary Information Protocol

All protocols operate over TCP and use plain-text request/response formats.

---

## TNet Status Codes

TMUTP and TBIP share a common status code grid — **TNet Status Codes**. The codes are identical for both protocols, though some codes may have no special use in one of them. If a code is received, it should be interpreted according to this table.

| Code | Name                  | Description                     |
|------|-----------------------|---------------------------------|
| 20   | OK                    | Successful response with body   |
| 21   | MOVED                 | Resource moved                  |
| 22   | OK EMPTY              | Success, empty body             |
| 23   | MULTIPLE CHOICES      | Multiple variants available     |
| 30   | BAD REQUEST           | Invalid request                 |
| 31   | NOT FOUND             | Document or resource not found  |
| 32   | FORBIDDEN             | Access denied                   |
| 40   | SERVER ERROR          | Internal server error           |
| 41   | NOT IMPLEMENTED       | Method or functionality not implemented |
| 42   | SERVICE UNAVAILABLE   | Service temporarily unavailable |
| 50   | METHOD NOT ALLOWED    | Method not allowed              |
| 51   | VERSION NOT SUPPORTED | Unsupported protocol version    |

---

## 1. UTNP — Universal Text Negotiation Protocol

**Version:** 1.0  
**Port:** 34  
**Transport:** TCP  
**Purpose:** Service discovery and protocol negotiation. UTNP does not transfer content and cannot be used in URLs.

### Requests

```
UTNP/1.0 NEGOTIATE <protocol1>,<protocol2>,...
UTNP/1.0 LIST
```

**Examples:**
```
UTNP/1.0 NEGOTIATE tmutp
UTNP/1.0 NEGOTIATE tmutp,tbip
UTNP/1.0 LIST
```

### Responses

**Successful negotiation:**
```
UTNP/1.0 START PROTOCOL <name> <version> <host> <port>
```

**Available protocols list:**
```
UTNP/1.0 AVAILABLE
<name1> <version1> <host1> <port1>
<name2> <version2> <host2> <port2>
...
```

**Error:**
```
UTNP/1.0 ERROR <message>
```

**Error with available protocols (if requested protocol is unsupported):**
```
UTNP/1.0 ERROR Unsupported protocol
UTNP/1.0 AVAILABLE
<protocol1> <version1> <host1> <port1>
...
```

**UTNP version error:**
```
UTNP ERROR - UNSUPPORTED UTNP VERSION, SUPPORTED UTNP/1.0
```

### Server Behavior

1. Listen on port 34.
2. Accept a request line.
3. If the UTNP version is not `1.0`, return the version error and close the connection.
4. If the request is `LIST`, return the list of all available protocols.
5. If the request is `NEGOTIATE`:
   - Check whether at least one of the listed protocols is supported.
   - If yes, return `START PROTOCOL` with the parameters of the first supported protocol.
   - If no, return `ERROR Unsupported protocol` and the `AVAILABLE` list.
6. Close the connection after responding.

### Standard TNet Protocols

| Name    | Version | Default Port |
|---------|---------|--------------|
| `tmutp` | 1.0     | 100          |
| `tbip`  | 1.0     | 81           |

---

## 2. TMUTP — TMkUp Transfer Protocol

**Version:** 1.0  
**Port:** 100  
**Transport:** TCP  
**Purpose:** Transfer of TMkUp documents (text only).

### Request Format

**GET:**
```
TMUTP/1.0 GET <path>
```

**POST:**
```
TMUTP/1.0 POST <path>
Content-Length: <length>
<body>
```

**Examples:**
```
TMUTP/1.0 GET /index.tmkup
TMUTP/1.0 GET /docs/
TMUTP/1.0 GET /about
TMUTP/1.0 POST /feedback
Content-Length: 25

name=John&message=Hello
```

### Status Codes

TMUTP uses the **TNet Status Codes** grid (see above).

### Response Format

```
TMUTP/1.0 <code> <name>
Content-Type: text/tmkup
Content-Length: <length>
[Last-Modified: <date>]
[Location: <new-path>]

<TMkUp body>
```

**Examples:**

```
TMUTP/1.0 20 OK
Content-Type: text/tmkup
Content-Length: 100
Last-Modified: 2024-06-10

---tmk-meta
title: "Home"
---tmk-meta
# Welcome!
```

```
TMUTP/1.0 21 MOVED
Location: /index.tmkup
Content-Type: text/tmkup
Content-Length: 0
```

```
TMUTP/1.0 31 NOT FOUND
Content-Type: text/tmkup
Content-Length: 80

---tmk-meta
title: "31 — Not Found"
---tmk-meta
# 31 — Document not found
```

```
TMUTP/1.0 51 VERSION NOT SUPPORTED
Content-Type: text/tmkup
Content-Length: 90

---tmk-meta
title: "51 — Unsupported Version"
---tmk-meta
# Server supports only TMUTP/1.0
```

### Headers

**Request:**
| Header           | Description              | Example              |
|------------------|--------------------------|----------------------|
| `Content-Length` | Body length (for POST)   | `Content-Length: 25` |

**Response:**
| Header           | Description              | Example                     |
|------------------|--------------------------|-----------------------------|
| `Content-Type`   | Always `text/tmkup`      | `Content-Type: text/tmkup`  |
| `Content-Length` | Body length              | `Content-Length: 1024`      |
| `Last-Modified`  | Last modification date   | `Last-Modified: 2024-06-10` |
| `Location`       | New path (for 21 MOVED)  | `Location: /index.tmkup`    |

### Server Behavior

1. Accept a request.
2. Check the version. If not `1.0`, return `51 VERSION NOT SUPPORTED`.
3. If `GET`:
   - If the path ends with `/`, look for `index.tmkup` in the directory.
   - Otherwise, look for a file **exactly by name** (no extension is appended).
   - If found, return `20 OK` with the content.
   - If not found, return `31 NOT FOUND`.
   - If the path contains parameters (e.g., `/search?q=...`), the server handles them at its discretion.
4. If `POST`:
   - Read the request body (format `application/x-www-form-urlencoded`).
   - Process the data.
   - Return `20 OK` with confirmation or `22 OK EMPTY`.
5. If the method is neither `GET` nor `POST`, return `50 METHOD NOT ALLOWED`.
6. If the body format is unsupported, return `41 NOT IMPLEMENTED`.

---

## 3. TBIP — TNet Binary Information Protocol

**Version:** 1.0  
**Port:** 81  
**Transport:** TCP  
**Purpose:** Transfer of binary information (images, audio, video).  
**Exclusion:** Executable files (.exe, .bin) are not considered binary information.

### Request Format

**GET:**
```
TBIP/1.0 GET <path>
```

**HEAD (optional):**
```
TBIP/1.0 HEAD <path>
```

**Examples:**
```
TBIP/1.0 GET /logo.png
TBIP/1.0 GET /audio/song.mp3
TBIP/1.0 HEAD /video/clip.mp4
```

### Status Codes

TBIP uses the **TNet Status Codes** grid (see above). Code `41 NOT IMPLEMENTED` is part of the grid, though it may not be used by every TBIP server.

### Response Format

```
TBIP/1.0 <code> <name>
Content-Type: <type>
Content-Length: <length>
[Last-Modified: <date>]
[Location: <new-path>]

<binary data>
```

**Examples:**

```
TBIP/1.0 20 OK
Content-Type: image/png
Content-Length: 12345
Last-Modified: 2024-06-12

<binary PNG data>
```

```
TBIP/1.0 31 NOT FOUND
Content-Type: text/plain
Content-Length: 20

File not found
```

### Content-Types

| Type                     | Description                    |
|--------------------------|--------------------------------|
| `image/png`              | PNG                            |
| `image/jpeg`             | JPEG                           |
| `image/gif`              | GIF                            |
| `image/webp`             | WebP                           |
| `audio/mpeg`             | MP3                            |
| `audio/ogg`              | OGG                            |
| `video/mp4`              | MP4                            |
| `video/webm`             | WebM                           |
| `application/octet-stream` | Any other binary file (client may offer to save) |

### Server Behavior

1. Accept a request.
2. Check the version. If not `1.0`, return `51 VERSION NOT SUPPORTED`.
3. If `GET` or `HEAD`:
   - Look for a file **exactly by name** (no extension is appended).
   - If found, determine the MIME type by extension (or by magic bytes).
   - Return `20 OK` with headers and body (for HEAD — headers only).
   - If not found, return `31 NOT FOUND`.
   - If the path is a directory, return `31 NOT FOUND` (by default, no index).
4. If the method is neither `GET` nor `HEAD`, return `50 METHOD NOT ALLOWED`.

**Content-Length:** The server always sends the full size of the binary data in bytes. Chunked transfer is not used. The client reads exactly `Content-Length` bytes.

---

## TNet Workflow

```
Client connects to port 34 (UTNP)
  |
  |-- UTNP/1.0 NEGOTIATE tmutp
  |<-- UTNP/1.0 START PROTOCOL tmutp 1.0 example.com 100
  |
  |-- closes connection
  |
  |-- connects to example.com:100 (TMUTP)
  |-- TMUTP/1.0 GET /index.tmkup
  |<-- TMUTP/1.0 20 OK + TMkUp
  |
  |-- (if an image is needed)
  |-- again UTNP to port 34
  |-- UTNP/1.0 NEGOTIATE tbip
  |<-- UTNP/1.0 START PROTOCOL tbip 1.0 example.com 81
  |
  |-- connects to example.com:81 (TBIP)
  |-- TBIP/1.0 GET /logo.png
  |<-- TBIP/1.0 20 OK + binary PNG data
```
