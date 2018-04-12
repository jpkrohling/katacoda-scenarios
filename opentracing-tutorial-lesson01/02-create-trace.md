A trace is a directed acyclic graph of spans. A span is a logical representation of some work done in your application. Each span has these minimum attributes: an operation name, a start time, and a finish time.

Let's create a trace that consists of just a single span. To do that we need an instance of the `io.opentracing.Tracer`. We can use a global instance returned by `io.opentracing.util.GlobalTracer.get()`.

<pre class="file" data-filename="opentracing-tutorial/java/src/main/java/lesson01/exercise/Hello.java" data-target="replace">package lesson01.exercise;

import io.opentracing.Span;
import io.opentracing.util.GlobalTracer;

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

    public static void main(String[] args) {
        if (args.length != 1) {
            throw new IllegalArgumentException("Expecting one argument");
        }
        String helloTo = args[0];
        new Hello(GlobalTracer.get()).sayHello(helloTo);
    }
}</pre>

We are using the following basic features of the OpenTracing API:

* a tracer instance is used to create a span builder via buildSpan()
* each span is given an operation name, "say-hello" in this case
* builder is used to create a span via startManual()
* each span must be finished by calling its finish() function
* the start and end timestamps of the span will be captured automatically by the tracer implementation

However, if we run this program, we will see no difference, and no traces in the tracing UI. That's because the function `GlobalTracer.get()` returns a no-op tracer by default. Try it out: `./run.sh lesson01.exercise.Hello Bryan`{{execute}}
