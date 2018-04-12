We may ask - so what, we could've done the same thing by passing the `greeting` as an HTTP request parameter. However, that is exactly the point of this exercise - we did not have to change any APIs on the path from the root span in `Hello.java` all the way to the server-side span in `formatter`, three levels down. If we had a much larger application with much deeper call tree, say the `formatter` was 10 levels down, the exact code changes we made here would have worked, despite 8 more services being in the path. If changing the API was the only way to pass the data, we would have needed to modify 8 more services to get the same effect.

Some of the possible applications of baggage include:

* passing the tenancy in multi-tenant systems
* passing identity of the top caller
* passing fault injection instructions for chaos engineering
* passing request-scoped dimensions for other monitoring data, like separating metrics for prod vs. test traffic
