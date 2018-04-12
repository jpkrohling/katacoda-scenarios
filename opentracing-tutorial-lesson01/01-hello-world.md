First, let's switch to the Java version of the tutorial: `cd opentracing-tutorial/java`{{execute}}.

Then, let's create a simple Java program `lesson01/exercise/Hello.java` that takes an argument and prints `Hello, {arg}!`.

<pre class="file" data-filename="opentracing-tutorial/java/src/main/java/lesson01/exercise/Hello.java" data-target="replace">package lesson01.exercise;

public class Hello {

    private void sayHello(String helloTo) {
        String helloStr = String.format("Hello, %s!", helloTo);
        System.out.println(helloStr);
    }

    public static void main(String[] args) {
        if (args.length != 1) {
            throw new IllegalArgumentException("Expecting one argument");
        }
        String helloTo = args[0];
        new Hello().sayHello(helloTo);
    }
}
</pre>

And now run it with `./run.sh lesson01.exercise.Hello Bryan`{{execute}}. Here we're using a simple helper script `run.sh` that executes a class via Maven, as well as strips out some of it diagnostic logging, so, it might take a while for the first run to complete.