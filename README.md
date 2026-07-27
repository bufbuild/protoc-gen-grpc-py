<div align="center">

![The Buf logo](https://raw.githubusercontent.com/bufbuild/protoc-gen-grpc-py/main/.github/buf-logo.svg)

# protoc-gen-grpc-py

[![PyPI version](https://img.shields.io/pypi/v/protoc-gen-grpc-py?style=flat-square)](https://pypi.org/project/protoc-gen-grpc-py)
[![License](https://img.shields.io/pypi/l/protoc-gen-grpc-py?style=flat-square)](https://github.com/bufbuild/protoc-gen-grpc-py/blob/main/LICENSE)
[![Slack](https://img.shields.io/badge/slack-buf-%23e01e5a?style=flat-square)](https://buf.build/links/slack)

`protoc-gen-grpc-py` generates **well-typed, idiomatic [gRPC](https://grpc.io) stubs** for [protobuf-py](https://github.com/bufbuild/protobuf-py), the ergonomic and modern Protobuf library for Python.

Fully typed clients and servicers you can read. asyncio and sync variants. <br />
Plugs into your existing [grpcio](https://pypi.org/project/grpcio/) channels and servers.

[Quickstart](#quickstart) •
[Example](https://github.com/bufbuild/protoc-gen-grpc-py/tree/main/example) •
[protobuf-py](https://github.com/bufbuild/protobuf-py)

</div>

It's a compatibility layer for projects already built on [grpcio](https://pypi.org/project/grpcio/). The generated clients and servicers plug into your existing gRPC channels and servers, but serialize `protobuf-py` messages directly. This lets you upgrade to a better Protobuf package without touching your RPC stack.

> [!TIP]
> For new projects, use [Connect for Python](https://github.com/connectrpc/connect-py) instead. Connect speaks the gRPC and gRPC-Web protocols in addition to its own, so existing gRPC clients can call a Connect server unchanged. But you also get plain HTTP APIs you can `curl`, first-class streaming, and generated clients for **every major language**, including your frontend. `protoc-gen-grpc-py` only exists so current gRPC codebases can get the improvements of `protobuf-py` today without a full rewrite.

## Quickstart

Generated code requires the runtime libraries [protobuf-py](https://pypi.org/project/protobuf-py/) and [grpcio](https://pypi.org/project/grpcio/). The plugin works with Protobuf compilers like [buf](https://github.com/bufbuild/buf).

```shellsession
$ uv add protobuf-py grpcio
$ uv add --dev protoc-gen-py protoc-gen-grpc-py buf-bin
```

Add `protoc-gen-grpc-py` alongside `protoc-gen-py` in your `buf.gen.yaml`:

```yaml
version: v2
inputs:
  - directory: proto
plugins:
  # Generates message types (*_pb.py).
  - local: protoc-gen-py
    out: src/gen
  # Generates gRPC service stubs (*_pb_grpc.py).
  - local: protoc-gen-grpc-py
    out: src/gen
```

To generate code for all Protobuf files within your project, run:

```shellsession
$ uv run -- buf generate
```

A `*_pb_grpc.py` file is generated for each proto file that declares a service. Message types come from [`protoc-gen-py`](https://pypi.org/project/protoc-gen-py/), which also manages `__init__.py` files.

## Feature highlights

### Well-typed clients you can read

For each service, the plugin emits a client whose methods are fully typed, so editors and type checkers understand every RPC method, its request and response types, and its streaming shape:

```python
async with grpc.aio.insecure_channel("localhost:50051") as channel:
    client = ElizaServiceClient(channel)
    response = await client.say(SayRequest(sentence="Hello!"))
    print(response.sentence)
```

### Servicers that register themselves

Each service also gets a servicer base class with typed method signatures and an `add_to_server` method:

```python
class ElizaService(ElizaServiceServicer):
    async def say(
        self, request: SayRequest, context: grpc.aio.ServicerContext
    ) -> SayResponse:
        return SayResponse(sentence=f"You said: {request.sentence}")


async def serve() -> None:
    server = grpc.aio.server()
    ElizaService().add_to_server(server)
    server.add_insecure_port("[::]:50051")
    await server.start()
    await server.wait_for_termination()
```

### asyncio and sync, side by side

Every client and servicer is generated in both asyncio (`grpc.aio`) and synchronous flavors:

```python
with grpc.insecure_channel("localhost:50051") as channel:
    client = ElizaServiceClientSync(channel)  # synchronous servicer
    response = client.say(SayRequest(sentence="Hello!"))
```

### protobuf-py messages end to end

Requests and responses are [protobuf-py](https://github.com/bufbuild/protobuf-py) messages. They have readable generated code, typed oneofs with pattern matching, real `IntEnum` enums, and a high-performance Rust encoder/decoder, with zero dependency on the legacy `google-protobuf` runtime.

<details>
<summary><h2>Generating with protoc</h2></summary>

The plugin also works with [protoc](https://github.com/protocolbuffers/protobuf/releases) as a standard `protoc` plugin:

```shellsession
$ uv run protoc --proto_path proto \
    --py_out src/gen \
    --grpc-py_out src/gen \
    proto/a.proto proto/b.proto proto/c.proto
```

</details>

## Example

See the [example](https://github.com/bufbuild/protoc-gen-grpc-py/tree/main/example) for a complete client and server using the generated stubs, including server, client, and bidirectional streaming.
