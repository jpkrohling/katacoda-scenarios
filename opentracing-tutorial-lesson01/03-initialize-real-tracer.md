Let's create an instance of a real tracer, such as Jaeger (http://github.com/uber/jaeger-client-java). Our `pom.xml` already imports Jaeger:

```xml
<dependency>
    <groupId>com.uber.jaeger</groupId>
    <artifactId>jaeger-core</artifactId>
    <version>0.26.0</version>
</dependency>
```

We need to add some imports:

<pre class="file" data-target="clipboard">
import com.uber.jaeger.Configuration;
import com.uber.jaeger.Configuration.ReporterConfiguration;
import com.uber.jaeger.Configuration.SamplerConfiguration;
import com.uber.jaeger.Tracer;
import com.uber.jaeger.samplers.ConstSampler;
</pre>

And we define a helper function that will create a tracer.

<pre class="file" data-target="clipboard">
public static com.uber.jaeger.Tracer initTracer(String service) {
    SamplerConfiguration samplerConfig = SamplerConfiguration.fromEnv()
            .withType(ConstSampler.TYPE)
            .withParam(1);

    ReporterConfiguration reporterConfig = ReporterConfiguration.fromEnv()
            .withLogSpans(true);

    Configuration config = new Configuration(service)
            .withSampler(samplerConfig)
            .withReporter(reporterConfig);

    return (com.uber.jaeger.Tracer) config.getTracer();
}
</pre>

To use this instance, let's change the main function:

<pre class="file" data-target="clipboard">
Tracer tracer = initTracer("hello-world");
new Hello(tracer).sayHello(helloTo);
tracer.close();
</pre>

Note that we are passing a string `hello-world` to the init method. It is used to mark all spans emitted by the tracer as originating from a `hello-world` service.

This is how our `Hello` class looks like now:

<pre class="file" data-filename="java/src/main/java/lesson01/exercise/Hello.java" data-target="replace">package lesson01.exercise;

import com.uber.jaeger.Configuration;
import com.uber.jaeger.Configuration.ReporterConfiguration;
import com.uber.jaeger.Configuration.SamplerConfiguration;
import com.uber.jaeger.Tracer;
import com.uber.jaeger.samplers.ConstSampler;
import io.opentracing.Span;

public class Hello {

    private final io.opentracing.Tracer tracer;

    private Hello(io.opentracing.Tracer tracer) {
        this.tracer = tracer;
    }

    private void sayHello(String helloTo) {
        Span span = tracer.buildSpan("say-hello").startManual();

        String helloStr = String.format("Hello, %s!", helloTo);
        System.out.println(helloStr);

        span.finish();
    }

    public static com.uber.jaeger.Tracer initTracer(String service) {
        SamplerConfiguration samplerConfig = SamplerConfiguration.fromEnv()
                .withType(ConstSampler.TYPE)
                .withParam(1);

        ReporterConfiguration reporterConfig = ReporterConfiguration.fromEnv()
                .withLogSpans(true);

        Configuration config = new Configuration(service)
                .withSampler(samplerConfig)
                .withReporter(reporterConfig);

        return (com.uber.jaeger.Tracer) config.getTracer();
    }

    public static void main(String[] args) {
        if (args.length != 1) {
            throw new IllegalArgumentException("Expecting one argument");
        }
        String helloTo = args[0];

        Tracer tracer = initTracer("hello-world");
        new Hello(tracer).sayHello(helloTo);
        tracer.close();

    }
}</pre>

NOTE: as this scenario runs on Docker, we need to specify the Agent's hostname to the Jaeger client. We can do that by setting this: `export JAEGER_ENDPOINT=http://host01:14268/api/traces`{{execute}}.

Running the program now, we see a span logged. Try it out: `./run.sh lesson01.exercise.Hello Bryan`{{execute}}.

Check also the [Jaeger UI](https://[[HOST_SUBDOMAIN]]-16686-[[KATACODA_HOST]].environments.katacoda.com/search?service=hello-world) for the newly created trace!