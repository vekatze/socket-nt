# socket-nt

`socket-nt` is a thin layer over network sockets.

## Installation

```sh
neut get socket https://github.com/vekatze/socket-nt/raw/main/archive/0-3-18.tar.zst
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
    protocol: int, // the third argument of socket(2)
    port: int16,
    address: &text,
    backlog: int, // the second argument of listen(2)
    threads: int, // the number of worker threads
    interpreter: (socket-address, text) -> text, // how to interpret responses
  )
}

// Starts a server.
inline start-server(c: config): system(unit)
```

## Example

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
      threads = 4,
      interpreter = {
        // interpreter: (request) -> response
        function (t: text): text {
          printf("request:\n{}\n", [t]);
          *"HTTP/1.1 200 OK\r\nContent-Length: 24\r\n\r\n{\"key\": \"Hello, world!\"}"
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

Then run `neut build --execute`:

```text
❯ neut build --execute
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
