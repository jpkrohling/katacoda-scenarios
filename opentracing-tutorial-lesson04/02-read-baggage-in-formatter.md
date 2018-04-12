On the formatter side, we want to retrieve the baggage item and use it in our `helloStr`. To accomplish that, let's add the following code to the `formatter`'s HTTP handler:

<pre class="file" data-target="clipboard">
String greeting = scope.span().getBaggageItem("greeting");
if (greeting == null) {
    greeting = "Hello";
}
String helloStr = String.format("%s, %s!", greeting, helloTo);
</pre>