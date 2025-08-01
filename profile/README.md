# Car Rental Microservice Project

TL;DR:

-- A study project to employ modern web development techniques. The system is built to provide users a catalog of cars to be rented for a period of time. The front end was built with the **Next js** 15 framework, many of the features available in the framework was used to make concise and simple, yet effective, UI code. Such features as next-auth, server side rendering, action-states and many others.

-- The back end was built using the extremely famous Java's framework **Spring** together with a microservice architecture system design. Spring's tools and ecosystem was extensivily used to various purposes. Spring data JPA was used to handle the repositories. Spring Boot to manage auto-configurations. Spring Security to handle authorizations with a hybrid system using a JWT approach together with oauth social authentications. I integrated the system with Mercado Pago SDK to handle the payment processing. MySQL was chosen for the database managent system for all services and Kafka to handle transactions in the system.

--All systems in the project was made into a docker image uploaded to dockerhub. Also jenkins pipelines were created for the microservices ci/cd. Now for the more detailed description of the project.

---

**Summary:**

1.  Introduction
2.  System Design
3.  Back end implementation
4.  Front end application
5.  Elasticsearch, Logstash and Kibana
6.  CI/CD and Docker

---

1 **Introduction**

There was a time when microservices were all the hype in web development, with videos and articles about them flooding the internet. It seemed like every developer was creating a microservice system, and this design decision was presented as a solution to all the world's problems. That's what was happening when I was a beginner developer, so I started learning how to create microservices. I enrolled in many courses that promised to teach me microservices, watched countless videos on the topic, and read numerous articles. All of it seemed great.

The thing is, from a learning perspective, microservices are hard—really hard. They are difficult to design well and to code, they introduce many points of failure, and it can even be hard to justify this design choice nowadays. Because of this, most of the courses and people who promised to teach microservices never presented a course or tutorial that really had a true microservice project approach. Many of them included anti-patterns in their architecture, sometimes sharing the same database or Git repository. They never showed how to deal with asynchronous communication, nor did they talk about retries and handling failures. Due to the complexity that comes with the architecture, most projects presented in those courses were simplistic simulations of a routine that in no way resembled useful software.

Designing a system with a microservices approach is quite different from the regular monolith way. That is why I came up with this project: to devise a microservice project with moderate complexity while staying as close as I could to a real-world microservice project.

The core idea is very simple: a website where users can create an account and rent a selected car from a catalog for a period of time, much like many e-commerce sites out there. Then, I intentionally "overengineered" the system with everything I thought was relevant to the project, designing the core rental functionality with asynchronous communication between several microservices. To put the project into practice, I used Spring to create the services and Next.js to build the front-end application.

2 **System Design**

<img src="car-rental-diagram2.png" alt="Car Rental System Microservices Diagram" width="600">

The main idea for the system is to use a microservice approach to provide the main car renting feature. With that in mind, I created four microservices: the inventory microservice, the user microservice, the payment microservice, and finally, the booking microservice. All of them work as independently as possible from each other. Each of the microservices has its own database schema, its own Git repository, and its own CI/CD pipeline.

The inventory microservice is responsible for keeping track of the cars the company has at its disposal. The database for this service has information on the cars' availability, as well as the car's rental price and details. The user microservice is responsible for managing user registration and login to the system through JWTs, and also keeps track of users' sensitive information. The payment microservice is responsible for processing payment transactions and connects with a third-party payment gateway; I particularly chose the Mercado Pago system. Finally, the booking microservice handles the rental details, such as reservations, total price calculation, and scheduling.

The core functionality of our system comes from connecting the microservices. In a monolith application, a car rental transaction would be straightforward, likely involving a few service calls to repository methods. In a microservice setting, this is a bit trickier. Since we can't simply wrap all cross-service operations into a single transaction, we need a different approach. I chose to implement the Saga Pattern.

To handle the core transaction, I used an Orchestration Saga Pattern. This involved a separate service, the orchestrator, which takes the rental request from the user and coordinates the microservices through a sequence of steps. The orchestrator first tells the inventory service to check the availability of the chosen car. If the car is available, the orchestrator tells the booking service to create a reservation, schedule, and price. Once that information is available, the orchestrator commands the payment service to issue a payment. This service uses information from the front end to create a payment in the Mercado Pago API and stores the payment details. The saga is then considered complete. If any of these steps fail, the orchestrator commands each service to perform a compensating action to roll back any changes made before the failure.

Communication between the orchestrator and the microservices is handled through events and commands using the publisher/subscriber design pattern. The orchestrator publishes commands, and the microservices, acting as subscribers, listen to them to trigger their processing. Similarly, the microservices publish events based on their processing results. The orchestrator, as a subscriber, listens to these events and decides the next course of action, whether it's to trigger the next step or command previous services to roll back.

I chose to end the saga at the payment step due to the behavior of the Mercado Pago API, particularly in its test environment. While more code could be written to handle the payment webhook once a payment is completed, the webhook is not very straightforward to work with, so I decided to conclude the saga at this point.

Other than that, our system also has an API gateway to act as a single entry point. All requests are redirected by it to the appropriate service. A microservice approach truly shines in a Kubernetes setup, which can leverage its power to provide robust orchestration, auto-scaling, and self-healing for individual microservices, making the entire system highly resilient, scalable, and much easier to manage. However, since Kubernetes can be expensive, I was unable to introduce it here. Nevertheless, as the groundwork has been laid, the project will be presented with Docker images for the services and a Docker Compose file to run them in a local Docker environment.

3 **Back end Implementation:**

Now, let's explore in greater detail the implementation of the system. The core functionality and business logic are built in the backend. To accomplish this, I chose the Spring ecosystem, using Java to build all the microservices. Each microservice is a Spring Boot project, leveraging its auto-configuration capabilities. Internally, the services follow a layered architecture, with clearly separated responsibilities.

The Inventory microservice is responsible for managing car availability and details. I used Spring Data JPA to manage repositories and query methods efficiently. The REST endpoints are exposed via a controller annotated with standard Spring MVC annotations, primarily for internal communication. For centralized error handling, I used @ControllerAdvice to define a global exception handler. This service is also integrated with Kafka: I created both consumer and producer configurations to enable event-driven communication within the system.

The User microservice plays a particularly important role. One of my main goals in this project was to support both custom login credentials and social login providers. To achieve this, I implemented authentication using the OAuth2 protocol. Users can authenticate via GitHub or Google; the system then validates the access tokens and determines which endpoints are accessible based on token verification. This is enforced in each microservice through Spring Security’s AuthenticationManager and token decoders. I intentionally chose GitHub and Google because they provide different types of tokens — GitHub uses non-opaque tokens, while Google uses opaque ones — allowing me to design a more robust and flexible authentication strategy.

In addition to social login, the system also supports a custom credentials-based login, which involves managing user records internally. For this, I used Spring Security along with NimbusJwtDecoder to build the token handling infrastructure. Currently, only the orchestrator service contains the full authentication configuration; most of the other microservices have not yet been updated to enforce auth validation.

The Booking service is relatively simple. It is responsible for scheduling rentals and calculating the total rental price. Much like the other services, it uses Spring Data JPA for managing database repositories and includes Kafka producer and consumer configurations to communicate with other components in the system.

The Payment microservice is another special component of the system. One of my major goals was to integrate a real-world third-party payment gateway, so I chose the Mercado Pago SDK, primarily to support both credit card and PIX payment methods. For the most part, the integration was straightforward — it simply required creating the appropriate Payment instance, and the SDK would internally handle the API request to the Mercado Pago platform. The SDK configuration itself is minimal and easy to work with. However, a notable limitation in the test environment is that PIX payment statuses cannot be updated, and they are always marked as "pending", which is a significant drawback for testing full payment flows in this project. Aside from the Mercado Pago integration, this microservice follows a similar structure as the others, with layers for repositories, services, and Kafka messaging infrastructure.

All microservice configurations are managed through a centralized Config Server, using Spring Cloud Config. I maintain a private configuration repository that stores all application settings in a version-controlled manner. This setup enables each service to dynamically retrieve its required configurations at runtime, promoting consistency and maintainability across the system.

Additionally, the system includes an Eureka Server for service discovery and registration. Each microservice registers itself with Eureka upon startup, enabling other services to locate and communicate with it dynamically, without needing to hardcode IP addresses or port numbers. Finally, I used Spring Cloud Gateway to build an API Gateway, which acts as the unified entry point for the backend system. It routes incoming requests to the appropriate microservices.

For the core functionality of the project, I needed an Orchestrator service to implement the Saga Pattern. This service was created specifically to handle that responsibility. Its controller is responsible for starting the Saga and initiating the rental process. The service includes Kafka producer and consumer configurations, as well as Spring Security configuration. The Saga will only be initiated if the user is authenticated, whether through the project's custom credentials or via social login. The Orchestrator listens to events published by the microservices and responds accordingly.

---

4 Front end Application

To implement the UI of the project, I chose to use Next.js. Having studied React for a couple of years now, I feel pretty comfortable coding with it, so I decided to use the most recent version — Next.js 15. I saw it as a good step forward in my front-end studies. Overall, it was a great experience. Next.js offers a lot of features that made working on the project enjoyable. The server-side rendering works amazingly well, and its integration with TypeScript is also quite smooth. I used Zod for handling the registration and login forms. I created pages to explore the car catalog, place rental orders, and handle login and registration, all of them styled using TailwindCSS.

To handle user login, I used NextAuth.js, and it was actually pretty simple. It's a great tool that provides a secure user session implementation with minimal configuration. With just a couple of files, I was able to create and manage login using GitHub and Google providers, as well as my own custom credentials all in a way that ensures the JWT is never exposed directly to the client.

To structure the application, I used the App Router introduced in the latest versions of Next.js, taking advantage of nested layouts and dynamic routing to organize pages like car details, catalog. I implemented loading states using the loading.tsx file in route segments, along with Tailwind-based spinners and skeleton placeholders that provide visual feedback during data fetching and server interactions. I also leveraged the useActionState hook to manage form state and validation feedback on the client side.

---

5 Elasticsearch, Logstash and Kibana

To implement observability across the microservices architecture, I integrated the ELK Stack (Elasticsearch, Logstash, and Kibana) along with Zipkin for distributed tracing. Each microservice is configured to store the logs in a folder that are read by Logstash. The Logstash then forwards the logs to Elasticsearch which indexes this data, enabling querying and filtering based on timestamp, log level, or custom fields. Then I use Kibana to help me visualize and query the logs.

---

6 CI/CD and Docker

Since this is a microservice based project each microservice has it's own pipeline, so I set up a CI/CD pipeline using Jenkins to automate the build, containerization, and deployment of each microservice. The pipeline fetches shared dependencies, builds each service with Maven, and packages the application into a Docker image. These images are then tagged and pushed to Docker Hub for version control and consistent environment parity. For deployment, the pipeline authenticates with Google Cloud using a service account and securely connects to a Compute Engine VM, where it pulls the latest Docker image and restarts the corresponding container using Docker Compose.

At the start of the project my goal for the deployment was to set a kubernetes so I made a bunch deployment and services yaml, however kubernetes is expensive, I tried first to use AWS to deploy a simple kubernetes for the microservices but that was outside free tier plan. I then decided to stepback and just set a dockercompose file and upload my images on dockerhub, the deploy became just a VM on GCP that would use this dockercompose file and create containers for the services.

7 Wrap up and conclusions

Microservices are a complex architectural choice, implementing one gave me a clearer understanding of why many tutorials or courses either oversimplify the pattern or end up breaking fundamental architectural principles. The reality is that microservices introduce a lot of design and operational complexity from distributed data management to fault tolerance and asynchronous communication which makes them hard to implement correctly, especially in a learning environment.

Despite these challenges, I set out to create a project that reflects a realistic microservice architecture with moderate complexity and did my best to follow best practices: separating services completely (codebase, database, deployment), using event-driven communication, and implementing an orchestration-based Saga pattern to handle distributed transactions. Along the way, I integrated a real-world payment gateway, Mercado Pago, configured observability with the ELK stack and Zipkin, and built a full front-end application using modern tools and patterns available in Next.js 15.

There are still areas for improvement. Some microservices are still missing authentication features, and the rent workflow could be extended to fully handle post-payment webhook events. Nevertheless, I think I've met the goals I had when I started. It allowed me to solidify my knowledge of Spring Boot, Kafka, Docker, and CI/CD, while also pushing my frontend skills forward with React, Next.js, and TailwindCSS.
