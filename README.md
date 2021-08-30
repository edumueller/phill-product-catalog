# Product Catalog

This project contains all the parts of the Product Catalog Application, each on its own sub-directory and respective github repos.

The goal of this repository is to facilitate the development and management of the application.

# Running this project

To run the project, you can clone this repository with all the submodules using the following commands:

```
$ git clone --recursive git@github.com:edumueller/phill-product-catalog.git

$ cd phill-product-catalog

$ docker-compose up
```

# Submodules:

This project is split into 4 different repositories:

- Authentication Microservice: [auth](#authentication-microservice)
- Product Catalog Microservice (API): [catalog](#product-catalog-microservice)
- Worker Microservice (job queue / processor): [worker](#worker-microservice)
- Shared NPM Package: [@phill-sdk/common](#phill-sdk)

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

## Authentication Microservice

JWT Authentication, User CRUD, Logout

| Verb | Endpoint     | URL                    | Params          |
| ---- | ------------ | ---------------------- | --------------- |
| POST | Sign Up      | /api/users/signup      | email, password |
| POST | Sign In      | /api/users/signin      | email, password |
| GET  | Current User | /api/users/currentuser |                 |
| POST | Sign Out     | /api/users/signout     |                 |

## Product Catalog Microservice

Product CRUD - RESTful API

| Verb | Endpoint | URL                 | Params                                           |
| ---- | -------- | ------------------- | ------------------------------------------------ |
| POST | new      | /api/products       | name (string), price (number), quantity (number) |
| GET  | show     | /api/products/`:id` |                                                  |
| PUT  | update   | /api/products/`:id` |                                                  |

## Phill SDK

### [`@phill-sdk/common`](./common)

This is an NPM Package that I've published in order to share classes, interfaces, types, handlers, etc. between different projects.

See the [@phill-sdk/common](https://www.npmjs.com/package/@phill-sdk/common)

## Worker Microservice

This microservice listens for published `product:created` and `product:updated` events, then adds a job to the queue to be processed.

The job tries to update the supply chain with the latest product info.
If the job fails, it will automatically retry using an exponential time function to increment the wait time between retries a maximum of 18 times in 72 hours.

When the product is successfully updated in the supply chain, it emits a sync-complete event that is caught by the product catalog microservice, updating the syncCompletedAt field with a timestamp from when the product information in the supply chain was updated.
