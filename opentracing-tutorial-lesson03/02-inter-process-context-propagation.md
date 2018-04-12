Since the only change we made in the `Hello.java` app was to replace two operations with HTTP calls, the tracing story remains the same - we get a trace with three spans, all from `hello-world` service. But now we have two more microservices participating in the transaction and we want to see them in the trace as well. In order to continue the trace over the process boundaries and RPC calls, we need a way to propagate the span context over the wire. The OpenTracing API provides two functions in the Tracer interface to do that, `inject(spanContext, format, carrier)` and `extract(format, carrier)`.

The `format` parameter refers to one of the three standard encodings the OpenTracing API defines:
* `TEXT_MAP` where span context is encoded as a collection of string key-value pairs,
* `BINARY` where span context is encoded as an opaque byte array,
* `HTTP_HEADERS`, which is similar to `TEXT_MAP` except that the keys must be safe to be used as HTTP headers.

The `carrier` is an abstraction over the underlying RPC framework. For example, a carrier for `TEXT_MAP` format is an interface that allows the tracer to write key-value pairs via `put(key, value)` method, while a carrier for Binary format is simply a `ByteBuffer`.

The tracing instrumentation uses `inject` and `extract` to pass the span context through the RPC calls.