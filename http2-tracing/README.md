# HTTP/2 Tracing With eBPF Uprobes Demo

Use eBPF uprobes to trace HTTP/2 headers, without any changes to the application code.

## What is this demo?

This demo provides the gRPC client and server, and the uprobe tracer for
[Observing HTTP/2 Traffic is Hard, but eBPF Can Help](https://blog.px.dev/http2-tracing)
blog post. Follow the instructions provided below to trace HTTP/2 message headers with eBPF uprobes.

## Prerequisites

* This demo only works on Linux. The code was tested on Ubuntu 20.04.3 LTS with 5.4 kernel.
* Install [BCC](https://github.com/iovisor/bcc/blob/master/INSTALL.md).
* All of the code is written in [Golang](https://go.dev/doc/install).
* Various Go packages might need to be installed, follow the directives to install them when any of
  the `go build` commands failed.

## Trace HTTP/2 headers with wireshark

First launch wireshark, and then launch gRPC client & server. When launching wireshark, you'll need
to set the port for HTTP/2 protocol, follow the menu `Edit > Preferences > Protocols > HTTP2`, set
`HTTP2 TCP port` to `50051`, as shown in the picture below:

<img src=".readme_assets/wireshark_http2_setting.png" alt="Wireshark HTTP2 setting">

```
# Launch wireshark with display filter on the gRPC server's listening port 50051.
sudo wireshark -Y "tcp.port == 50051 && http2" -i any -k

# Build and launch gRPC server.
go build -o /tmp/grpc_server server/main.go && /tmp/grpc_server

# Build and launch gRPC client.
go build -o /tmp/grpc_client client/main.go && /tmp/grpc_client --count 10
```

Click the packet in the capture window containing the text `200 OK`:

<img src=".readme_assets/wireshark_packets.png" alt="Wireshark packets">

The decoded HTTP/2 headers can be seen as below:

<img src=".readme_assets/wireshark_http2_headers.png" alt="Wireshark HTTP/2 headers" width="500">

Now launch wireshark after launching gRPC client & server, such that **wireshark only capture
the traffic in the middle of the connection**. This time, wireshark cannot decode the headers
as shown in the screenshot below.

```
go build -o /tmp/grpc_server server/main.go && /tmp/grpc_server
go build -o /tmp/grpc_client client/main.go && /tmp/grpc_client --count 20
sudo wireshark -Y "tcp.port == 50051 && http2" -i any -k
```

<img src=".readme_assets/wireshark_http2_headers_failed.png" alt="Wireshark HTTP/2 headers failure" width="500">

## Trace HTTP/2 headers with uprobe tracer

Run the commands below to see the uprobe tracer tracing the HTTP/2 messages from the gRPC server to
client:

```
go build -o /tmp/grpc_server server/main.go && /tmp/grpc_server

# Build and launch the uprobe tracer.
# If failed, see instructions in the Development section below to change gobpf version.
go build -o /tmp/uprobe_trace ./uprobe_trace && \
 sudo -E /tmp/uprobe_trace --binary=/tmp/grpc_server

go build -o /tmp/grpc_client client/main.go && /tmp/grpc_client --count 10
```

The output of the uprobe tracer looks like:

<img src=".readme_assets/uprobe_tracer_output.png" alt="Uprobe tracer output">

## Development

* If you want to modify the gRPC protobuf, you'll need to install:
  [protocol buffer compiler](https://grpc.io/docs/protoc-installation/) and
  [go protobuf plugin](https://grpc.io/docs/languages/go/quickstart/).
* If you have modified the gRPC protobuf file, you'll need to run the following command to update
  the generated go source files:
  ```
  protoc --go_out=. --go-grpc_out=. --go-grpc_opt=require_unimplemented_servers=false proto/greet.proto
  ```
* Change `gobpf` version in go.mod if `go build` failed when building uprobe tracer with error
  like below:
  ```
  github.com/iovisor/gobpf/bcc ../../../pkg/mod/github.com/iovisor/gobpf@v0.0.0-20200614202714-e6b321d32103/bcc/module.go:261:33:
  not enough arguments in call to _C2func_bpf_attach_uprobe
        have (_Ctype_int, uint32, *_Ctype_char, *_Ctype_char, _Ctype_ulong, _Ctype_int)
        want (_Ctype_int, uint32, *_Ctype_char, *_Ctype_char, _Ctype_ulong, _Ctype_int, _Ctype_uint)
  ```
  In `go.mod`, uncomment any of the gobpf versions listed to replace the exiting one.
  Until the build succeeds. Gobpf is a wrapper of BCC's libbpf APIs, but gobpf is not released
  through the same channel as BCC, so the build failed when the versions are not compatible.

## Bugs & Features

Feel free to file a bug or an issue for a feature request. You can also join our
[Slack](https://slackin.px.dev/) community.
