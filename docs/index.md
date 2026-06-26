# protoc-gen-grpc-py

Code generator plugin to generate idiomatic, fully-typed stubs for [gRPC-Python](https://grpc.io/docs/languages/python/)
using messages from [protobuf-py](https://protobufpy.com).

We believe [Connect](https://connectrpc.com/docs/python/getting-started/) is the definitive library for building RPC
applications in Python. But often you can't just swap out existing production RPC infrastructure just like that.
That doesn't mean you have to be stuck with gRPC's untyped, effectively opaque generated stubs. `protoc-gen-grpc-py`
provides the definitive experience for gRPC-Python users, with fully typed stubs based on `protobuf-py` messages.

## Usage

`protoc-gen-grpc-py` is a Protocol Buffers code generator plugin, so it can be used with a compiler like
[Buf](https://buf.build/product/cli). Add it to your `buf.gen.yaml` next to `protobuf-py` and regenerate.

```yaml
version: v2
plugins:
  - remote: buf.build/bufbuild/py
    out: src
  - remote: buf.build/bufbuild/grpc-py
    out: src
```

```shellsession
buf generate
```

You will see `_pb_grpc.py` files next to the `_pb.py` message definitions.

## Migrating

`protoc-gen-grpc-py` stubs are not a drop-in replacement for gRPC-Python's. Official stubs generate single
classes for both sync and async, with the appropriate mode used based on duck typing. It is impossible
to map this to fully-typed stubs, so we generate four fully-typed classes corresponding to gRPC-Python's two.

gRPC's `FooServicer` is replaced by two classes, `FooService` and `FooServiceSync` for sync and async respectively.
For existing servicer implementations, you just swap to the corresponding base class, and replace usage of the
free function `add_FooServicer_to_server` with calling the `add_to_server` method on the service class.

=== "Before (gRPC-Python)"

    ```python
    from haberdasher_pb2 import Hat
    from haberdasher_pb2_grpc import HaberdasherServicer, add_HaberdasherServicer_to_server


    class MyHaberdasher(HaberdasherServicer):
        async def MakeHat(self, request, context):
            return Hat(size=request.inches, color="blue")


    add_HaberdasherServicer_to_server(MyHaberdasher(), server)
    ```

=== "After (protoc-gen-grpc-py)"

    ```python
    import grpc

    from haberdasher_pb import Hat, Size
    from haberdasher_pb_grpc import HaberdasherService


    class MyHaberdasher(HaberdasherService):
        async def make_hat(self, request: Size, context: grpc.aio.ServicerContext) -> Hat:
            return Hat(size=request.inches, color="blue")


    MyHaberdasher().add_to_server(server)
    ```

gRPC's `FooStub` is replaced by two classes, `FooClient` and `FooClientSync` for sync and async respectively.
For existing client usage, swap constructing the stub with the appropriate client class.

=== "Before (gRPC-Python)"

    ```python
    from haberdasher_pb2 import Size
    from haberdasher_pb2_grpc import HaberdasherStub

    stub = HaberdasherStub(channel)
    hat = await stub.MakeHat(Size(inches=12))
    ```

=== "After (protoc-gen-grpc-py)"

    ```python
    from haberdasher_pb import Size
    from haberdasher_pb_grpc import HaberdasherClient

    client = HaberdasherClient(channel)
    hat = await client.make_hat(Size(inches=12))
    ```

Note that the `protobuf-py` message classes themselves have significant differences from `google.protobuf`,
especially related to nested messages, repeated elements, and oneofs. Take in the migration cost for the
message usage when switching to this plugin. The result will hopefully be worth it, fully typed gRPC usage
across your codebase, with no changes to servers, `Dockerfile`s, or any other infrastructure.
