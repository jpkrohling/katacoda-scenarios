As in Lesson 3, first start the `formatter` and `publisher` in separate terminals, then run the client with two arguments, e.g. `Bryan Bonjour`. The `publisher` should print `Bonjour, Bryan!`.

For each terminal window, run: `cd opentracing-tutorial/java`{{execute}} and then `export JAEGER_ENDPOINT=http://host01:14268/api/traces`{{execute}}

Formatter: `./run.sh lesson04.exercise.Formatter server`{{execute}}
Publisher: `./run.sh lesson04.exercise.Publisher server`{{execute}}
Client:    `./run.sh lesson04.exercise.Hello Bryan Bonjour`{{execute}}

Once the client has finished running, check the trace [in the UI](https://[[HOST_SUBDOMAIN]]-16686-[[KATACODA_HOST]].environments.katacoda.com/search?service=hello-world).