Just to refresh our memory, this is our client so far:
<pre class="file" data-filename="opentracing-tutorial/java/src/main/java/lesson04/exercise/Hello.java" data-target="replace">package lesson04.exercise;

import com.google.common.collect.ImmutableMap;
import com.uber.jaeger.Tracer;
import io.opentracing.Scope;
import io.opentracing.propagation.Format;
import io.opentracing.tag.Tags;
import lib.Tracing;
import okhttp3.HttpUrl;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.Response;

import java.io.IOException;

public class Hello {

    private final Tracer tracer;
    private final OkHttpClient client;

    private Hello(Tracer tracer) {
        this.tracer = tracer;
        this.client = new OkHttpClient();
    }

    private String getHttp(int port, String path, String param, String value) {
        try {
            HttpUrl url = new HttpUrl.Builder().scheme("http").host("localhost").port(port).addPathSegment(path)
                    .addQueryParameter(param, value).build();
            Request.Builder requestBuilder = new Request.Builder().url(url);

            Tags.SPAN_KIND.set(tracer.activeSpan(), Tags.SPAN_KIND_CLIENT);
            Tags.HTTP_METHOD.set(tracer.activeSpan(), "GET");
            Tags.HTTP_URL.set(tracer.activeSpan(), url.toString());
            tracer.inject(tracer.activeSpan().context(), Format.Builtin.HTTP_HEADERS, new RequestBuilderCarrier(requestBuilder));

            Request request = requestBuilder.build();
            Response response = client.newCall(request).execute();
            if (response.code() != 200) {
                throw new RuntimeException("Bad HTTP result: " + response);
            }
            return response.body().string();
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

    private void sayHello(String helloTo) {
        try (Scope scope = tracer.buildSpan("say-hello").startActive(true)) {
            scope.span().setTag("hello-to", helloTo);

            String helloStr = formatString(helloTo);
            printHello(helloStr);
        }
    }

    private String formatString(String helloTo) {
        try (Scope scope = tracer.buildSpan("formatString").startActive(true)) {
            String helloStr = getHttp(8081, "format", "helloTo", helloTo);
            scope.span().log(ImmutableMap.of("event", "string-format", "value", helloStr));
            return helloStr;
        }
    }

    private void printHello(String helloStr) {
        try (Scope scope = tracer.buildSpan("printHello").startActive(true)) {
            getHttp(8082, "publish", "helloStr", helloStr);
            scope.span().log(ImmutableMap.of("event", "println"));
        }
    }

    public static void main(String[] args) {
        if (args.length != 1) {
            throw new IllegalArgumentException("Expecting one argument");
        }
        String helloTo = args[0];
        Tracer tracer = Tracing.init("hello-world");
        new Hello(tracer).sayHello(helloTo);
        tracer.close();
        System.exit(0); // okhttpclient sometimes hangs maven otherwise
    }
}</pre>

And this is how our Formatter:
<pre class="file" data-filename="opentracing-tutorial/java/src/main/java/lesson04/exercise/Formatter.java" data-target="replace">package lesson04.exercise;

import com.google.common.collect.ImmutableMap;
import io.dropwizard.Application;
import io.dropwizard.Configuration;
import io.dropwizard.setup.Environment;
import io.opentracing.Scope;
import io.opentracing.SpanContext;
import io.opentracing.Tracer;
import io.opentracing.propagation.Format;
import io.opentracing.propagation.TextMapExtractAdapter;
import io.opentracing.tag.Tags;
import lib.Tracing;

import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.QueryParam;
import javax.ws.rs.core.Context;
import javax.ws.rs.core.HttpHeaders;
import javax.ws.rs.core.MediaType;
import javax.ws.rs.core.MultivaluedMap;
import java.util.HashMap;

public class Formatter extends Application< Configuration> {
    private final Tracer tracer;

    private Formatter(Tracer tracer) {
        this.tracer = tracer;
    }

    @Path("/format")
    @Produces(MediaType.TEXT_PLAIN)
    public class FormatterResource {

        @GET
        public String format(@QueryParam("helloTo") String helloTo, @Context HttpHeaders httpHeaders) {
            try (Scope scope = startServerSpan(tracer, httpHeaders, "format")) {
                String helloStr = String.format("Hello, %s!", helloTo);
                scope.span().log(ImmutableMap.of("event", "string-format", "value", helloStr));
                return helloStr;
            }
        }

    }

    @Override
    public void run(Configuration configuration, Environment environment) throws Exception {
        environment.jersey().register(new FormatterResource());
    }

    public static void main(String[] args) throws Exception {
        System.setProperty("dw.server.applicationConnectors[0].port", "8081");
        System.setProperty("dw.server.adminConnectors[0].port", "9081");

        Tracer tracer = Tracing.init("formatter");
        new Formatter(tracer).run(args);
    }

    public static Scope startServerSpan(Tracer tracer, javax.ws.rs.core.HttpHeaders httpHeaders, String operationName) {
        // format the headers for extraction
        MultivaluedMap< String, String> rawHeaders = httpHeaders.getRequestHeaders();
        final HashMap< String, String> headers = new HashMap< String, String>();
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
}</pre>

Our Publisher is like this:
<pre class="file" data-filename="opentracing-tutorial/java/src/main/java/lesson04/exercise/Publisher.java" data-target="replace">package lesson04.exercise;

import com.google.common.collect.ImmutableMap;
import io.dropwizard.Application;
import io.dropwizard.Configuration;
import io.dropwizard.setup.Environment;
import io.opentracing.Scope;
import io.opentracing.SpanContext;
import io.opentracing.Tracer;
import io.opentracing.propagation.Format;
import io.opentracing.propagation.TextMapExtractAdapter;
import io.opentracing.tag.Tags;
import lib.Tracing;

import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.QueryParam;
import javax.ws.rs.core.Context;
import javax.ws.rs.core.HttpHeaders;
import javax.ws.rs.core.MediaType;
import javax.ws.rs.core.MultivaluedMap;
import java.util.HashMap;

public class Publisher extends Application< Configuration> {

    private final Tracer tracer;

    private Publisher(Tracer tracer) {
        this.tracer = tracer;
    }

    @Path("/publish")
    @Produces(MediaType.TEXT_PLAIN)
    public class PublisherResource {

        @GET
        public String format(@QueryParam("helloStr") String helloStr, @Context HttpHeaders httpHeaders) {
            try (Scope scope = startServerSpan(tracer, httpHeaders, "publish")) {
                System.out.println(helloStr);
                scope.span().log(ImmutableMap.of("event", "println", "value", helloStr));
                return "published";
            }
        }
    }

    @Override
    public void run(Configuration configuration, Environment environment) throws Exception {
        environment.jersey().register(new PublisherResource());
    }

    public static void main(String[] args) throws Exception {
        System.setProperty("dw.server.applicationConnectors[0].port", "8082");
        System.setProperty("dw.server.adminConnectors[0].port", "9082");
        Tracer tracer = Tracing.init("publisher");
        new Publisher(tracer).run(args);
    }

    public static Scope startServerSpan(Tracer tracer, javax.ws.rs.core.HttpHeaders httpHeaders, String operationName) {
        // format the headers for extraction
        MultivaluedMap< String, String> rawHeaders = httpHeaders.getRequestHeaders();
        final HashMap< String, String> headers = new HashMap< String, String>();
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
}</pre>

And finally, our RequestBuilderCarrier:
<pre class="file" data-filename="opentracing-tutorial/java/src/main/java/lesson04/exercise/RequestBuilderCarrier.java" data-target="replace">package lesson04.exercise;

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

Let's add a new parameter to our Hello's main method, so that it accepts a `greeting` in addition to a name. This is how the main method would look like in the end:

<pre class="file" data-target="clipboard">
public static void main(String[] args) {
    if (args.length != 2) {
        throw new IllegalArgumentException("Expecting two arguments, helloTo and greeting");
    }
    String helloTo = args[0];
    String greeting = args[1];
    Tracer tracer = Tracing.init("hello-world");
    new Hello(tracer).sayHello(helloTo, greeting);
    tracer.close();
    System.exit(0); // okhttpclient sometimes hangs maven otherwise
}
</pre>

Don't forget to change the signature of the method `sayHello()` to accept this new parameter:
<pre class="file" data-target="clipboard">
private void sayHello(String helloTo, String greeting) {
</pre>

And add this instruction to `sayHello` method after starting the span:

<pre class="file" data-target="clipboard">
scope.span().setBaggageItem("greeting", greeting);
</pre>

By doing this we read a second command line argument as a "greeting" and store it in the baggage under `"greeting"` key.
