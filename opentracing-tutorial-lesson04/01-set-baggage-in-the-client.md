We'll start this lesson where we left the previous one. We are providing the `Publisher` and the `RequestBuilderCarrier` on the workspace, as there's no need to change anything on them during this lesson, but refer to the previous versions of the `Hello` client and `Formatter` when working on this lesson.

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
