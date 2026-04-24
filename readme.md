# socket

`socket-nt` is a thin layer over network sockets.

## Installation

```sh
neut get socket https://github.com/vekatze/socket-nt/raw/main/archive/0-3-55.tar.zst
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

// A socket address.
data socket-address {
| Socket-Address(
    ip-address: string,
    port: int16,
  )
}

// Server configuration.
data config {
| Config(
    family: address-family,
    comm-type: socket-type,
    reuse-socket: bool, // whether to enable SO_REUSEADDR
    protocol: c-int, // the third argument of socket(2)
    port: int16,
    address: &string,
    backlog: c-int, // the second argument of listen(2)
    threads: int, // the number of worker threads
    interpreter: (socket-address, descriptor) -> string, // how to interpret requests
  )
}

// Starts a server.
inline start-server(c: config) -> system(unit)
```

## Example

```neut
import {
  core.file.descriptor {descriptor},
  core.int.io {print-int-line},
  core.int.show {show-int},
  core.string {format, string-byte-length},
  core.string.io {print-line},
  core.system {get-error-message},
  this.address-family {AF_INET},
  this.socket {Config, Socket-Address, socket-address, start-server},
  this.socket-type {SOCK_STREAM},
}

define main() -> unit {
  let server-config =
    Config{
      family := AF_INET,
      comm-type := SOCK_STREAM,
      reuse-socket := False,
      protocol := 0,
      port := 8080,
      address := "0.0.0.0",
      backlog := 128,
      threads := 10,
      interpreter := {
        (address: socket-address, _: descriptor) => {
          let Socket-Address{ip-address, port} = address;
          pin ip-address = ip-address;
          print("client: ");
          print(ip-address);
          print(":");
          print-int-line(magic cast(_, _, port));
          let body = *"hello";
          let body-len on body = string-byte-length(body);
          format("HTTP/1.1 200 OK\r\nContent-Length: {}\r\n\r\n{}", List[show-int(body-len), body])
        }
      },
    };
  match start-server(server-config) {
  | Left(errno) =>
    pin error = get-error-message(errno);
    print("error: ");
    print-line(error)
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
curl --silent http://127.0.0.1:8080/foo/bar -d "yo"

# ↓
#
# hello
```
