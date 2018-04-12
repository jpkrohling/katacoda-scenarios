In the `formatString` function we already create a child span. In order to pass its context over the HTTP request we need to call `tracer.inject` before building the HTTP request:

<pre class="file" data-target="clipboard">
Tags.SPAN_KIND.set(tracer.activeSpan(), Tags.SPAN_KIND_CLIENT);
Tags.HTTP_METHOD.set(tracer.activeSpan(), "GET");
Tags.HTTP_URL.set(tracer.activeSpan(), url.toString());
tracer.inject(tracer.activeSpan().context(), Format.Builtin.HTTP_HEADERS, new RequestBuilderCarrier(requestBuilder));
</pre>

And the import statements for the newly used classes:

<pre class="file" data-target="clipboard">
import io.opentracing.propagation.Format;
import io.opentracing.tag.Tags;
</pre>

In this case the `carrier` is HTTP request headers object, which we adapt to the carrier API by wrapping in `RequestBuilderCarrier` helper class.

<pre class="file" data-filename="opentracing-tutorial/java/src/main/java/lesson03/exercise/RequestBuilderCarrier.java" data-target="replace">package lesson03.exercise;

import okhttp3.Request;

import java.util.Iterator;
import java.util.Map;

public class RequestBuilderCarrier implements io.opentracing.propagation.TextMap {
    private final Request.Builder builder;

    RequestBuilderCarrier(Request.Builder builder) {
        this.builder = builder;
    }

    @Override
    public Iterator< Map.Entry< String, String>> iterator() {
        throw new UnsupportedOperationException("carrier is write-only");
    }

    @Override
    public void put(String key, String value) {
        builder.addHeader(key, value);
    }
}</pre>

Notice that we also add a couple additional tags to the span with some metadata about the HTTP request, and we mark the span with a `span.kind=client` tag, as recommended by the OpenTracing [Semantic Conventions][semantic-conventions]. There are other tags we could add.

[semantic-conventions]: https://github.com/opentracing/specification/blob/master/semantic_conventions.md