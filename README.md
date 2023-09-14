So far, we have started the application with the easy-to-use function startStandaloneServer, thanks to which the application has not had to be configured that much

Unfortunately, startStandaloneServer does not allow adding subscriptions to the application, so in this branch we switched to the more robust expressMiddleware function. As the name of the function already suggests, it is an Express middleware, which means that Express must also be configured for the application, with the GraphQL server acting as middleware.

ApolloServerPluginDrainHttpServer has now been added to the configuration of the GraphQL server according to the recommendations of the documentation
The GraphQL server in the server variable is now connected to listen to the root of the server, i.e. to the / route, using the expressMiddleware object. Information about the logged-in user is set in the context using the function we defined earlier. Since it is an Express server, the middlewares express-json and cors are also needed so that the data included in the requests is correctly parsed and so that CORS problems do not appear.

Since the GraphQL server must be started before the Express application can start listening to the specified port, the entire initialization has had to be placed in an async function, which allows waiting for the GraphQL server to start.

When queries and mutations are used, GraphQL uses the HTTP protocol in the communication. In case of subscriptions, the communication between client and server happens with WebSockets.

 The resolver of the personAdded subscription registers and saves info about all the clients that do the subscription. The clients are saved to an "iterator object" called PERSON_ADDED thanks to the following code:

```
Subscription: {
  personAdded: {
    subscribe: () => pubsub.asyncIterator('PERSON_ADDED')
  },
} 

```

The iterator name is an arbitrary string, but to follow the convention, it is the subscription name written in capital letters.
Adding a new person publishes a notification about the operation to all subscribers with PubSub's method publish:

```JavaScript
pubsub.publish('PERSON_ADDED', { personAdded: person })
```
In this context, the order of execution is as follows:

1. The `subscribe` function defined in the `Subscription` resolver is executed when a client initiates a subscription. This function returns an async iterator (`pubsub.asyncIterator('PERSON_ADDED')`) that represents the stream of data for that subscription. However, it doesn't actually start sending data to clients at this point.

2. When a new person is added using the `addPerson` mutation, the line `pubsub.publish('PERSON_ADDED', { personAdded: person })` is executed. This line of code publishes a message to the `PERSON_ADDED` channel with the data of the newly added person.

3. The clients who have subscribed to the `personAdded` subscription will receive data when a message is published to the `PERSON_ADDED` channel. The WebSocket connection between the server and the subscribed clients is responsible for delivering this data.

So, the `subscribe` function is executed when a client subscribes to the subscription, and it returns an async iterator. The `pubsub.publish` method is executed when a new person is added, and it sends data to all clients who have subscribed to the `personAdded` subscription. The order of execution depends on when clients initiate subscriptions and when mutations are performed. These are separate processes, and the `subscribe` function doesn't send data immediately upon its execution; it sends data when there is something to publish, triggered by mutations or other events.