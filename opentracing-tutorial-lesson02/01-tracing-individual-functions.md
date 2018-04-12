In Lesson 1 we wrote a program that creates a trace that consists of a single span. That single span combined two operations performed by the program, formatting the output string and printing it. Let's use it as the base for this lesson.

<pre class="file" data-filename="opentracing-tutorial/java/src/main/java/lesson02/exercise/Hello.java" data-target="replace">package lesson02.exercise;

import io.opentracing.Span;
import com.google.common.collect.ImmutableMap;
import com.uber.jaeger.Tracer;
import lib.Tracing;

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
        Tracer tracer = Tracing.init("hello-world");
        new Hello(tracer).sayHello(helloTo);
        tracer.close();
    }
}</pre>

And let's switch to the Java version of the tutorial: `cd opentracing-tutorial/java`{{execute}}.

Let's move those operations into standalone functions first:

<pre class="file" data-target="clipboard">
String helloStr = formatString(span, helloTo);
printHello(span, helloStr);
</pre>

and the functions:

<pre class="file" data-target="clipboard">
private String formatString(Span span, String helloTo) {
    String helloStr = String.format("Hello, %s!", helloTo);
    span.log(ImmutableMap.of("event", "string-format", "value", helloStr));
    return helloStr;
}

private void printHello(Span span, String helloStr) {
    System.out.println(helloStr);
    span.log(ImmutableMap.of("event", "println"));
}
</pre>

Of course, this does not change the outcome. What we really want to do is to wrap each function into its own span.

<pre class="file" data-target="clipboard">
private String formatString(Span rootSpan, String helloTo) {
    Span span = tracer.buildSpan("formatString").startManual();
    try {
        String helloStr = String.format("Hello, %s!", helloTo);
        span.log(ImmutableMap.of("event", "string-format", "value", helloStr));
        return helloStr;
    } finally {
        span.finish();
    }
}

private void printHello(Span rootSpan, String helloStr) {
    Span span = tracer.buildSpan("printHello").startManual();
    try {
        System.out.println(helloStr);
        span.log(ImmutableMap.of("event", "println"));
    } finally {
        span.finish();
    }
}
</pre>

Let's run it: `./run.sh lesson02.exercise.Hello Bryan`{{execute}}

We got three spans, but there is a problem here. [If we search for the spans in the UI](https://[[HOST_SUBDOMAIN]]-16686-[[KATACODA_HOST]].environments.katacoda.com/search?service=hello-world) each one will represent a standalone trace with a single span. That's not what we wanted!

What we really wanted was to establish causal relationship between the two new spans to the root span started in `main()`. We can do that by passing an additional option `asChildOf` to the span builder:

<pre class="file" data-target="clipboard">
Span span = tracer.buildSpan("formatString").asChildOf(rootSpan).startManual();
</pre>

If we think of the trace as a directed acyclic graph where nodes are the spans and edges are the causal relationships between them, then the `ChildOf` option is used to create one such edge between `span` and `rootSpan`. In the API the edges are represented by `SpanReference` type that consists of a `SpanContext` and a label. The `SpanContext` represents an immutable, thread-safe portion of the span that can be used to establish references or to propagate it over the wire. The label, or `ReferenceType`, describes the nature of the relationship. `ChildOf` relationship means that the `rootSpan` has a logical dependency on the child `span` before `rootSpan` can complete its operation. Another standard reference type in OpenTracing is `FollowsFrom`, which means the `rootSpan` is the ancestor in the DAG, but it does not depend on the completion of the child span, for example if the child represents a best-effort, fire-and-forget cache write.

If we modify the `printHello` function accordingly and run the app, we'll see that all reported spans now belong to the same trace: `./run.sh lesson02.exercise.Hello Bryan`{{execute}}. [In the UI](https://[[HOST_SUBDOMAIN]]-16686-[[KATACODA_HOST]].environments.katacoda.com/search?service=hello-world), the trace will show a proper parent-child relationship between the spans.

For reference, here's how our final code looks like:

<pre class="file" data-filename="opentracing-tutorial/java/src/main/java/lesson02/exercise/Hello.java" data-target="replace">package lesson02.exercise;

import io.opentracing.Span;
import io.opentracing.util.GlobalTracer;

import com.google.common.collect.ImmutableMap;
import com.uber.jaeger.Tracer;
import lib.Tracing;

public class Hello {

    private final Tracer tracer;

    private Hello(Tracer tracer) {
        this.tracer = tracer;
    }

    private void sayHello(String helloTo) {
        Span span = tracer.buildSpan("say-hello").startManual();
        span.setTag("hello-to", helloTo);

        String helloStr = formatString(span, helloTo);
        printHello(span, helloStr);

        span.finish();
    }

    private  String formatString(Span rootSpan, String helloTo) {
        Span span = tracer.buildSpan("formatString").asChildOf(rootSpan).startManual();
        try {
            String helloStr = String.format("Hello, %s!", helloTo);
            span.log(ImmutableMap.of("event", "string-format", "value", helloStr));
            return helloStr;
        } finally {
            span.finish();
        }
    }

    private void printHello(Span rootSpan, String helloStr) {
        Span span = tracer.buildSpan("printHello").asChildOf(rootSpan).startManual();
        try {
            System.out.println(helloStr);
            span.log(ImmutableMap.of("event", "println"));
        } finally {
            span.finish();
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
    }
}</pre>
