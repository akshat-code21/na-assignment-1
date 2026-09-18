# Assignment: HTTP/1.1 Persistent-Connection Calculator

> **Course:** CN & Scaler — Session 5 (Evolution of HTTP)
> **Due:** Before Session 7
> **Source:** Lesson 5 slides, "Early HTTP/1.1 · Assignment"

---

## 1. Goal

Build a small calculator **HTTP server** whose entire feature set is four arithmetic
operations — but the arithmetic is *not the point*. The point is **HTTP/1.1 connection
persistence (keep-alive)**: your server must serve **many requests over one socket**,
unlike an HTTP/1.0 server that hangs up after every response.

**Constraints:**
- Any language.
- **No framework.** Just a raw socket (e.g., Python's `socket` module).
- You may look at `02-http11/server11.py` — but only **after you have failed at least once**.

---

## 2. Functional Requirements

The server listens on **port 8080** and implements four GET endpoints with query
parameters `a` and `b`:

| Request                | Expected response |
|------------------------|-------------------|
| `GET /add?a=2&b=3`     | `200` with body `5`  |
| `GET /sub?a=10&b=4`    | `200` with body `6`  |
| `GET /mul?a=6&b=7`     | `200` with body `42` |
| `GET /div?a=9&b=3`     | `200` with body `3`  |

### Error handling

| Case                          | Expected response | Meaning |
|-------------------------------|-------------------|---------|
| `GET /div?a=1&b=0`            | `400`             | Division by zero |
| `GET /add?a=x&b=3`           | `400`             | Non-numeric operand |
| `GET /pow?a=2&b=8`           | `404`             | Unknown route |
| `POST /add`                  | `405`             | Method not allowed |
| `GET /add` with **no `Host:` header** | `400`     | HTTP/1.1 requires a Host header |

---

## 3. The Grading Harness (the hard requirement)

The marker will use **one single TCP connection** for *all* requests:

```python
import socket

s = socket.create_connection(("localhost", 8080))

# then, over this same socket:
#   GET /add?a=2&b=3   -> 200  5
#   GET /sub?a=10&b=4  -> 200  6
#   GET /mul?a=6&b=7   -> 200  42
#   GET /div?a=1&b=0   -> 400
#   GET /pow?a=2&b=8   -> 404
#   POST /add          -> 405
#
# socket still open: True
# 1 TCP handshake, 6 responses
```

**Pass criteria:**
- All 6 responses returned **in order** over the **same socket**.
- The socket is **still open** after all responses (`socket still open: True`).
- Exactly **1 TCP handshake** for **6 responses**.

> If the socket dies before the marker is done, you have written a 1996 (HTTP/1.0) server.

---

## 4. Why It Is Actually Hard

Keeping the connection open forces a question HTTP/1.0 never asked you:
**"Where does this request end and the next one begin?"**

- In HTTP/1.0 the answer was "at EOF" — for free.
- In HTTP/1.1 you must **consume exactly `Content-Length` bytes** (for request bodies,
  e.g. the `POST /add` case) and **not one more** — byte `n+1` belongs to somebody else.
- You must correctly frame *both* directions: parse request headers to find request
  boundaries, and emit responses that let the client find response boundaries
  (e.g., `Content-Length` on your responses).

---

## 5. Stretch Goals (all optional)

1. **Honour `Connection: close`** — client asks to close, you close after responding.
2. **An idle timeout you can defend** — close connections that go quiet for too long.
3. **Chunked transfer encoding** (`Transfer-Encoding: chunked`).
4. **Pipelining** — take all six requests *at once* (sent back-to-back before reading
   any response) and answer **in order**.

---

## 6. Suggested Deliverables / Checklist

- [ ] `GET /add`, `/sub`, `/mul`, `/div` return correct results with `200`
- [ ] `/div` by zero → `400`
- [ ] Non-integer operands → `400`
- [ ] Unknown path → `404`
- [ ] Non-GET method on a valid route → `405`
- [ ] Missing `Host` header on HTTP/1.1 request → `400`
- [ ] All 6 graded requests succeed over **one** socket, in order
- [ ] Socket remains open afterwards (server does not close after each response)
- [ ] `Content-Length` body framing handled exactly (no over/under-read)
- [ ] (Stretch) `Connection: close`, idle timeout, chunked encoding, pipelining

---

## 7. Language Choice: Java ✅

**Constraint reminder:** any language is allowed, but **no frameworks / no built-in HTTP
server** — raw sockets only. In Java that means:

| Use | Don't use |
|-----|-----------|
| `java.net.ServerSocket`, `java.net.Socket` | `com.sun.net.httpserver.HttpServer` |
| `java.io.InputStream` / `OutputStream` | `HttpURLConnection`, Apache HttpClient |
| `java.nio.charset.StandardCharsets` | Spring, Jetty, Spark, etc. |

### Java-specific pitfalls to watch for

1. **Buffered readers eat bytes.** If you wrap the socket's `InputStream` in a
   `BufferedReader`, the buffer may swallow bytes of the *next* request (or a POST
   body) that you then can't read back. This is exactly the framing failure the
   assignment is designed to catch.
   → Prefer reading the raw `InputStream` and parsing header lines from bytes
   yourself, then reading **exactly** `Content-Length` body bytes from the same
   stream.
2. **`readLine()` doesn't tell you it ate `\r\n`.** Fine for headers, but never use
   it for the body.
3. **One connection, many requests** ⇒ after writing a response, **loop back and
   read the next request on the same `Socket`**. Do not call `socket.close()`
   after each response. Only close when the client closes or you decide to
   (e.g. `Connection: close`, idle timeout via `Socket.setSoTimeout()`).
4. **Write bytes, not strings.** `OutputStream.write(str.getBytes(StandardCharsets.UTF_8))`.
5. **Testing pipelining:** `printf` + `nc` closes after one send; to test multiple
   requests on one socket, write a tiny test client that sends all 6 requests
   back-to-back before reading (see §3), or use `nc` interactively.

### Useful APIs for the stretch goals

- Idle timeout → `socket.setSoTimeout(ms)` + catching `SocketTimeoutException`
- `Connection: close` → check the request header, close after responding
- Chunked encoding → decode `Transfer-Encoding: chunked` bodies yourself
- TCP no-delay (snappier responses) → `socket.setTcpNoDelay(true)`

---

*Reference implementation to consult only after a first failed attempt:*
`02-http11/server11.py` *(in the course repo `http-evolution.tar.gz`)*

