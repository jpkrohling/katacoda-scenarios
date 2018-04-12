Right now the trace we created is very basic. If we call our program with `Hello Susan` instead of `Hello Bryan`, the resulting traces will be nearly identical. It would be nice if we could capture the program arguments in the traces to distinguish them.

One naive way is to use the string `"Hello, Bryan!"` as the _operation name_ of the span, instead of `"say-hello"`. However, such practice is highly discouraged in distributed tracing, because the operation name is meant to represent a _class of spans_, rather than a unique instance. For example, in Jaeger UI you can select the operation name from a dropdown when searching for traces. It would be very bad user experience if we ran the program to say hello to a 1000 people and the dropdown then contained 1000 entries. Another reason for choosing more general operation names is to allow the tracing systems to do aggregations. For example, Jaeger tracer has an option of emitting metrics for all the traffic going through the application. Having a unique operation name for each span would make the metrics useless.

The recommended solution is to annotate spans with tags or logs. A _tag_ is a key-value pair that provides certain metadata about the span. A _log_ is similar to a regular log statement, it contains a timestamp and some data, but it is associated with span from which it was logged.

When should we use tags vs. logs?  The tags are meant to describe attributes of the span that apply to the whole duration of the span. For example, if a span represents an HTTP request, then the URL of the request should be recorded as a tag because it does not make sense to think of the URL as something that's only relevant at different points in time on the span. On the other hand, if the server responded with a redirect URL, logging it would make more sense since there is a clear timestamp associated with such event. The OpenTracing Specification provides guidelines called [Semantic Conventions][semantic-conventions] for recommended tags and log fields.

Using Tags
----------

In the case of `Hello Bryan`, the string `"Bryan"` is a good candidate for a span tag, since it applies to the whole span and not to a particular moment in time. We can record it like this:

<pre class="file" data-target="clipboard">
Span span = tracer.buildSpan("say-hello").startManual();
span.setTag("hello-to", helloTo);
</pre>

Using Logs
----------

Our hello program is so simple that it's difficult to find a relevant example of a log, but let's try. Right now we're formatting the `helloStr` and then printing it. Both of these operations take certain time, so we can log their completion:

<pre class="file" data-target="clipboard">
String helloStr = String.format("Hello, %s!", helloTo);
span.log(ImmutableMap.of("event", "string-format", "value", helloStr));

System.out.println(helloStr);
span.log(ImmutableMap.of("event", "println"));
</pre>

And let's not forget the `import` statement for the `ImmutableMap`:

<pre class="file" data-target="clipboard">
import com.google.common.collect.ImmutableMap;
</pre>

The log statements might look a bit strange if you have not previosuly worked with a structured logging API. Rather than formatting a log message into a single string that is easy for humans to read, structured logging APIs encourage you to separate bits and pieces of that message into key-value pairs that can be automatically processed by log aggregation systems. The idea comes from the realization that today most logs are processed by machines rather than humans. Just [google "structured-logging"][google-logging] for many articles on this topic.

The OpenTracing API for Java exposes structured logging API by accepting a collection of key-value pairs in the form of a `Map<String, ?>`. Here we are using Guava's `ImmutableMap.of()` to construct such a map, which takes an alternating list of `key1,value1,key2,value2` pairs.

The OpenTracing Specification also recommends all log statements to contain an `event` field that describes the overall event being logged, with other attributes of the event provided as additional fields.

If you run the program with these changes, then find the trace in the UI and expand its span (by clicking on it), you will be able to see the tags and logs.

<pre class="file" data-filename="opentracing-tutorial/java/src/main/java/lesson01/exercise/Hello.java" data-target="replace">package lesson01.exercise;

import io.opentracing.Span;
import com.google.common.collect.ImmutableMap;
import com.uber.jaeger.Configuration;
import com.uber.jaeger.Configuration.ReporterConfiguration;
import com.uber.jaeger.Configuration.SamplerConfiguration;
import com.uber.jaeger.Tracer;

public class Hello {

    private final Tracer tracer;

    private Hello(Tracer tracer) {
        this.tracer = tracer;
    }

    private void sayHello(String helloTo) {
        Span span = tracer.buildSpan("say-hello").startManual();
        span.setTag("hello-to", helloTo);
        
        String helloStr = String.format("Hello, %s!", helloTo);
        span.log(ImmutableMap.of("event", "string-format", "value", helloStr));

        System.out.println(helloStr);
        span.log(ImmutableMap.of("event", "println"));

        span.finish();
    }

    public static void main(String[] args) {
        if (args.length != 1) {
            throw new IllegalArgumentException("Expecting one argument");
        }
        String helloTo = args[0];

        com.uber.jaeger.Tracer tracer = initTracer("hello-world");
        new Hello(tracer).sayHello(helloTo);
        tracer.close();
    }

    public static com.uber.jaeger.Tracer initTracer(String service) {
        SamplerConfiguration samplerConfig = new SamplerConfiguration("const", 1);
        ReporterConfiguration reporterConfig = ReporterConfiguration.fromEnv();
        Configuration config = new Configuration(service, samplerConfig, reporterConfig);
        return (com.uber.jaeger.Tracer) config.getTracer();
    }
}</pre>

Try it out: `./run.sh lesson01.exercise.Hello Bryan`{{execute}}

[semantic-conventions]: https://github.com/opentracing/specification/blob/master/semantic_conventions.md
[google-logging]: https://www.google.com/search?q=structured-logging