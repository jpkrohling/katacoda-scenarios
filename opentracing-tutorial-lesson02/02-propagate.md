You may have noticed a few unpleasant side effects of our recent changes
* we had to pass the Span object as the first argument to each function
* we also had to write somewhat verbose try/finally code to finish the spans

OpenTracing API for Java provides a better way. Using thread-locals and the notion of an "active span", we can avoid passing the span through our code and just access it via `tracer`.

<pre class="file" data-target="clipboard">
private void sayHello(String helloTo) {
    try (Scope scope = tracer.buildSpan("say-hello").startActive(true)) {
        scope.span().setTag("hello-to", helloTo);
        
        String helloStr = formatString(helloTo);
        printHello(helloStr);
    }
}

private  String formatString(String helloTo) {
    try (Scope scope = tracer.buildSpan("formatString").startActive(true)) {
        String helloStr = String.format("Hello, %s!", helloTo);
        scope.span().log(ImmutableMap.of("event", "string-format", "value", helloStr));
        return helloStr;
    }
}

private void printHello(String helloStr) {
    try (Scope scope = tracer.buildSpan("printHello").startActive(true)) {
        System.out.println(helloStr);
        scope.span().log(ImmutableMap.of("event", "println"));
    }
}
</pre>

And let's not forget to import the newly used classes:

<pre class="file" data-target="clipboard">
import io.opentracing.Scope;
</pre>

In the above code we're making the following changes:
* We use `startActive(boolean)` method of the span builder instead of `startManual()`, which makes the span "active" by storing it in a thread-local storage.
* `startActive(boolean)` returns a `Scope` object instead of a `Span`. Scope is a container of the currently active span. We access the active span via `scope.span()`. Once the scope is closed, the previous scope becomes current, thus re-activating previously active span in the current thread.
* `Scope` is auto-closable, which allows us to use try-with-resource syntax.
* The boolean parameter in `startActive(boolean)` tells the Scope that once it is closed it should finish the span it represents.
* `startActive(boolean)` automatically creates a `ChildOf` reference to the previously active span, so that we don't have to use `asChildOf()` builder method explicitly.

If we run this program, we will see that all three reported spans have the same trace ID: `./run.sh lesson02.exercise.Hello Bryan`{{execute}}. Make sure to check the trace [in the UI](https://[[HOST_SUBDOMAIN]]-16686-[[KATACODA_HOST]].environments.katacoda.com/search?service=hello-world) as well!