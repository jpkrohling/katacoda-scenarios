We'll start this lesson based on where we left the last lesson, plus a small refactoring to avoid duplicating code. We'll also change the `formatString` and `printHello` methods to make RPC calls to two downstream services, `formatter` and `publisher`.

<pre class="file" data-filename="opentracing-tutorial/java/src/main/java/lesson03/exercise/Hello.java" data-target="replace">package lesson03.exercise;

import com.google.common.collect.ImmutableMap;
import com.uber.jaeger.Tracer;
import io.opentracing.Scope;
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

Let's add a `formatter` service, which is a Dropwizard-based HTTP server that responds to a request like `GET 'http://localhost:8081/format?helloTo=Bryan'` and returns `"Hello, Bryan!"` string

<pre class="file" data-filename="opentracing-tutorial/java/src/main/java/lesson03/exercise/Formatter.java" data-target="replace">package lesson03.exercise;

import io.dropwizard.Application;
import io.dropwizard.Configuration;
import io.dropwizard.setup.Environment;

import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.QueryParam;
import javax.ws.rs.core.MediaType;

public class Formatter extends Application< Configuration> {

    @Path("/format")
    @Produces(MediaType.TEXT_PLAIN)
    public class FormatterResource {

        @GET
        public String format(@QueryParam("helloTo") String helloTo) {
            String helloStr = String.format("Hello, %s!", helloTo);
            return helloStr;
        }
    }

    @Override
    public void run(Configuration configuration, Environment environment) throws Exception {
        environment.jersey().register(new FormatterResource());
    }

    public static void main(String[] args) throws Exception {
        System.setProperty("dw.server.applicationConnectors[0].port", "8081");
        System.setProperty("dw.server.adminConnectors[0].port", "9081");
        new Formatter().run(args);
    }
}</pre>

And finally, a `publisher` service, that is another HTTP server that responds to requests like `GET 'http://localhost:8082/publish?helloStr=hi%20there'` and prints `"hi there"` string to stdout:

<pre class="file" data-filename="opentracing-tutorial/java/src/main/java/lesson03/exercise/Publisher.java" data-target="replace">package lesson03.exercise;

import io.dropwizard.Application;
import io.dropwizard.Configuration;
import io.dropwizard.setup.Environment;

import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.QueryParam;
import javax.ws.rs.core.MediaType;

public class Publisher extends Application< Configuration> {

    @Path("/publish")
    @Produces(MediaType.TEXT_PLAIN)
    public class PublisherResource {

        @GET
        public String format(@QueryParam("helloStr") String helloStr) {
            System.out.println(helloStr);
            return "published";
        }
    }

    @Override
    public void run(Configuration configuration, Environment environment) throws Exception {
        environment.jersey().register(new PublisherResource());
    }

    public static void main(String[] args) throws Exception {
        System.setProperty("dw.server.applicationConnectors[0].port", "8082");
        System.setProperty("dw.server.adminConnectors[0].port", "9082");
        new Publisher().run(args);
    }
}</pre>

With all that in place, let's switch to the Java version of the tutorial: `cd opentracing-tutorial/java`{{execute}}.

To test it out, run the formatter and publisher services in separate terminals. On the first, run `./run.sh lesson03.exercise.Formatter server`{{execute}}. Then, click on the "+" sign close to the terminal tab title and open a new terminal. On this new terminal, change to the Java version of the tutorial as well `cd opentracing-tutorial/java`{{execute}} and run `./run.sh lesson03.exercise.Publisher server`{{execute}}.

NOTE: for each new terminal, don't forget to change to the Java version of the tutorial

On a third terminal, execute an HTTP request against the `formatter`: `curl 'http://localhost:8081/format?helloTo=Bryan'`{{execute}}

And then, execute and HTTP request against the `publisher`: `curl 'http://localhost:8082/publish?helloStr=hi%20there'`{{execute}}

The `publisher` stdout will show `"hi there"`.

As our client is already instrumented, we need to set the Jaeger's endpoint: `export JAEGER_ENDPOINT=http://host01:14268/api/traces`{{execute}}

NOTE: for each new terminal, we need to set the env var `JAEGER_ENDPOINT` if we are running an instrumented server/client

Finally, let's run the client app as we did in the previous lessons: `./run.sh lesson03.exercise.Hello Bryan`{{execute}}

We will see the `publisher` printing the line `"Hello, Bryan!"`.

If we open this trace [in the UI](https://[[HOST_SUBDOMAIN]]-16686-[[KATACODA_HOST]].environments.katacoda.com/search?service=hello-world), we should see three spans, just like the one we had at the end of lesson 2.