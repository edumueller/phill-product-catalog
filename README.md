# phill-product-catalog

# Running this project

To run the project, you can clone this repository with all the submodules using the command ``

# Infrastructure as Code - Kubernetes

This project can run in a development machine using `docker-compose up`, but I have also created the kubernetes configuration files that can be used to automatically create all the infrastructure necessary.

You can also test these configurations by using [skaffold](https://skaffold.dev/) in your local environment (tested on MacOS Big Sur).

# Confiability

Since the supply chain API is unreliable and might return random errors, the products are initially received by the catalog microservice and added to mongoDB, then an event is emitted to another microservice responsible for making sure the updates are successfully received by the supply chain api.

As soon as a product is received and added, an event containing that product's data is published to all microservices that are listening.

The [workers](./worker) listen to product:created and product:updated events. When one of these events is received, a job is added to our queue. _To manage the queue we are using the [Bull](https://optimalbits.github.io/bull/) library._

To make sure every update is successfully received by the Supply Chain API, the [workers](./worker) send a POST request, if the request fails, the job is marked as unsuccessful and an exponential time function is used to retry the job 18 times in 72 hours (see line 18 from [this file](./worker/src/events/listeners/product-updated-listener.ts)).

This approach ensures that product updates are synchronized as soon as possible, but also prevents job processing delays and big backlogs of pending requests from being processed at the same time.

# Scalability

I have created a microservice-oriented, event-driven architecture, using [NATS Streaming Server](https://docs.nats.io/nats-streaming-concepts/intro) for communication between microservices using events and a job queue with dedicated workers that can also be scaled. I have implemented document versioning to make sure every process can be executed in parallel (solves async communication problems).

# Performance

Using NATS Streaming Server to communicate between microservices is way more performant than using http. NATS can deliver between 7 and 88 million messages per second. Also, this microservice-oriented architecture can be easily scaled horizontally.
