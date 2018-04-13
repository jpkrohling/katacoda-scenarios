Our servers are currently not instrumented for tracing. We need to do the following:

#### Create an instance of a Tracer, similar to how we did it in `Hello.java`

Add a member variable and a constructor to the Formatter:

<pre class="file" data-target="clipboard">
private final Tracer tracer;

private Formatter(Tracer tracer) {
    this.tracer = tracer;
}
</pre>

And import the newly used class:
<pre class="file" data-target="clipboard">
import io.opentracing.Tracer;
</pre>

Replace the call to `Formatter.run()` with this:

<pre class="file" data-target="clipboard">
Tracer tracer = Tracing.init("formatter");
new Formatter(tracer).run(args);
</pre>

Note that we've moved the code that initializes a tracer to its own class, located under `lib`. We'll need to import this class:
<pre class="file" data-target="clipboard">
import lib.Tracing;
</pre>

#### Extract the span context from the incoming request using `tracer.extract`

First, let's add a helper function on the Hello class:

<pre class="file" data-target="clipboard">
public static Scope startServerSpan(Tracer tracer, javax.ws.rs.core.HttpHeaders httpHeaders, String operationName) {
    // format the headers for extraction
    MultivaluedMap&lt;String, String&gt; rawHeaders = httpHeaders.getRequestHeaders();
    final HashMap&lt;String, String&gt; headers = new HashMap&lt;String, String&gt;();
    for (String key : rawHeaders.keySet()) {
        headers.put(key, rawHeaders.get(key).get(0));
    }

    Tracer.SpanBuilder spanBuilder;
    try {
        SpanContext parentSpan = tracer.extract(Format.Builtin.HTTP_HEADERS, new TextMapExtractAdapter(headers));
        if (parentSpan == null) {
            spanBuilder = tracer.buildSpan(operationName);
        } else {
            spanBuilder = tracer.buildSpan(operationName).asChildOf(parentSpan);
        }
    } catch (IllegalArgumentException e) {
        spanBuilder = tracer.buildSpan(operationName);
    }
    return spanBuilder.withTag(Tags.SPAN_KIND.getKey(), Tags.SPAN_KIND_SERVER).startActive(true);
}
</pre>

And import the newly used classes:
<pre class="file" data-target="clipboard">
import io.opentracing.SpanContext;
import io.opentracing.propagation.Format;
import io.opentracing.propagation.TextMapExtractAdapter;
import io.opentracing.tag.Tags;
import javax.ws.rs.core.MultivaluedMap;
import java.util.HashMap;
</pre>

The logic here is similar to the client side instrumentation, except that we are using `tracer.extract` and tagging the span as `span.kind=server`. Instead of using a dedicated adapter class to convert JAXRS `HttpHeaders` type into `io.opentracing.propagation.TextMap`, we are copying the headers to a plain `HashMap<String, String>` and using a standard adapter `TextMapExtractAdapter`.

Now change the `FormatterResource` handler method to use `startServerSpan`:

<pre class="file" data-target="clipboard">
@GET
public String format(@QueryParam("helloTo") String helloTo, @Context HttpHeaders httpHeaders) {
    try (Scope scope = Tracing.startServerSpan(tracer, httpHeaders, "format")) {
        String helloStr = String.format("Hello, %s!", helloTo);
        scope.span().log(ImmutableMap.of("event", "string-format", "value", helloStr));
        return helloStr;
    }
}
</pre>

And import the newly used classes:
<pre class="file" data-target="clipboard">
import io.opentracing.Scope;
import com.google.common.collect.ImmutableMap;
import javax.ws.rs.core.Context;
import javax.ws.rs.core.HttpHeaders;
</pre>

It would be better to have this in a more appropriate place. We've prepared a `Tracing` class under the `lib` package: that's what we'll be using in the future.

#### Apply the same to the Publisher

Now, just apply the same changes to the publisher.