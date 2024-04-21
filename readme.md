# socket-nt

`socket-nt` is a thin layer over network sockets.

## Installation

```sh
neut get socket https://github.com/vekatze/socket/raw/main/archive/0-1.tar.zst
```

## Example Code

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
  | Fail(Errno(i)) =>
    printf("failed to start a server. error code: {}\n", [show(i)])
  | Pass(_) =>
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

The server logs the request like the below:

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
