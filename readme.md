# socket-nt

`socket-nt` is a thin layer over network sockets.

## Installation

```sh
neut get socket https://github.com/vekatze/socket-nt/raw/main/archive/0-3-21.tar.zst
```

## Types

```neut
// Only AF_INET is supported for now.
data address-family {
| AF_INET
}

// See socket(2).
data socket-type {
| SOCK_STREAM
| SOCK_DGRAM
| SOCK_RAW
}

// Server configuration.
data config {
| Config(
    family: address-family,
    comm-type: socket-type,
    reuse-socket: bool, // whether to enable SO_REUSEADDR
    protocol: c-int, // the third argument of socket(2)
    port: int16,
    address: &text,
    backlog: c-int, // the second argument of listen(2)
    threads: int, // the number of worker threads
    interpreter: (socket-address, text) -> text, // how to interpret requests
  )
}

// Starts a server.
inline start-server(c: config): system(unit)
```

## Example

`source/test.nt` contains something like the following:

```neut
define main(): unit {
  let server-config =
    Config of {
      family = AF.AF_INET,
      comm-type = SOCK_STREAM,
      reuse-socket = True,
      protocol = 0,
      port = 8080,
      address = "0.0.0.0",
      backlog = 128,
      threads = 10,
      interpreter = {
        function (address: socket-address, t: text): text {
          let Socket-Address of {ip-address, port} = address in
          printf("client: {}:{}\n", [ip-address, show-int(magic cast(_, _, port))]);
          let len on t = text-byte-length(t) in
          let body = format("{\"key\": \"Hello, world!\", \"request-length\": {}}", [show-int(len)]) in
          let body-len on body = text-byte-length(body) in
          printf("request:\n{}\n", [t]);
          format("HTTP/1.1 200 OK\r\nContent-Length: {}\r\n\r\n{}", [show-int(body-len), body])
        }
      },
    }
  in
  match start-server(server-config) {
  | Left(errno) =>
    printf("error: {}\n", [get-error-message(errno)])
  | Right(_) =>
    Unit
  }
}
```

You can run this by `neut build test --execute`:

```text
❯ neut build test --execute
listening 0.0.0.0:8080
```

Now open a new terminal and run:

```sh
curl --silent http://127.0.0.1:8080/foo/bar -d "whatever" | jq

# ↓
#
# {
#   "key": "Hello, world!"
# }
```

The server logs the request like the following:

```text
❯ neut build --execute
listening 0.0.0.0:8080
request:
POST /foo/bar HTTP/1.1
Host: 127.0.0.1:8080
User-Agent: curl/8.4.0
Accept: */*
Content-Length: 8
Content-Type: application/x-www-form-urlencoded

whatever
```
