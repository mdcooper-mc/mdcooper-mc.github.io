+++
title = "Service Framework"
date = 2025-09-15
categories = ["Architecture", "Spring Boot"]
tags = ["Event-Driven", "Annotation-Based", "BAU Workflows"]
+++

[browse this repository](https://dev.azure.com/mdcoopermc/_git/services-framework)

## Service Framework

The Service Framework enables the development of **event-based services** to orchestrate BAU workflows. It provides a declarative, annotation-driven model for defining service behavior, routing logic, and execution conditions—all within a Spring Boot environment. By abstracting the complexities of event handling and service discovery, it allows developers to focus on core business logic while maintaining a highly decoupled architecture.

---

### Architectural Design and AOP Integration

The framework's architecture is centered around **Aspect-Oriented Programming (AOP)** using AspectJ. This design choice allows for the clean separation of cross-cutting concerns, such as event publishing, exception handling, and metadata injection.

By utilizing `@Around` advice on methods annotated with `@PhysicalEventSource`, the `ServiceEventManagerAspect` intercepts execution to automate the event lifecycle. This interception provides a high-performance **O(1)** overhead, while enabling complex **O(N)** service chaining where $N$ is the number of processors in a given workflow. The aspect uses the `ProceedingJoinPoint` to execute the business logic, then enriches the resulting `ServiceModel` with high-precision timestamps and the source name before publishing it to the broader event-driven system.

#### Implementation of the Event Aspect

The following snippet demonstrates how the `ServiceEventManagerAspect` captures the method return value and handles potential exceptions, ensuring that even failures are propagated through the event system for audit and recovery.

```java
@Aspect
@Component
@EnableAspectJAutoProxy(proxyTargetClass = true)
public abstract class ServiceEventManagerAspect {
    
    @Around("@annotation(com.cooper.framework.api.PhysicalEventSource)")
    public void evaluatePhysicalSource(final ProceedingJoinPoint pjp) throws Throwable {
        try {
            // Proceed with the original method execution
            final ServiceModel newEvent = (ServiceModel) pjp.proceed();
            
            if (nonNull(newEvent)) {
                // Enrich and publish the successful event
                this.eventPublisher.publishEvent(
                        serviceModel(newEvent)
                                .withConsumedOn(LocalDateTime.now())
                                .withSource(getSourceName(pjp))
                                .build()
                );
            }
        } catch (Throwable t) {
            // Automate exception publishing for system resilience
            this.eventPublisher.publishException(
                serviceModel().withConsumedOn(LocalDateTime.now()).build(), 
                of(new Exception("Event Source Failure", t))
            );
            throw t;
        }
    }
}
```

---

### Annotation-Driven Function Processing

The framework provides a declarative model for defining service behavior through custom annotations, which significantly reduces boilerplate and improves architectural consistency.

- **`@PhysicalEventSource`**: Acts as the primary entry point for event generation. Methods marked with this annotation automatically participate in the managed event lifecycle, ensuring that all output is captured, logged, and propagated without manual orchestration.
- **`@EventProcessor`**: Defines the processing logic for consuming services. It supports **Spring Expression Language (SpEL)**, enabling conditional routing and execution based on the event's payload or metadata. This allows for dynamic business rules that are evaluated at runtime.
- **`@EventProfile`**: Controls the activation of services based on the environment or use case. This allows a single codebase to support multiple deployment configurations.

#### Declarative Service Example

Developers can define complex event pipelines simply by annotating their methods. The framework handles the underlying messaging and orchestration.

```java
@Component
public class DemoServices {

    @Scheduled(fixedDelay = 5000)
    @PhysicalEventSource(name = "ExternalFeed")
    public ServiceModel<String> source() {
        return serviceModel()
                .withEvent("RECEIVE")
                .withPayload("Financial Market Data Update")
                .build();
    }

    @EventProcessor(condition = "event == 'RECEIVE'")
    public ServiceModel<String> processor(final ServiceModel<String> model) {
        // Logic to process the received event
        return serviceModel(model)
                .withEvent("PROCESSED")
                .build();
    }
}
```

This annotation-driven approach, combined with Spring's `@Lookup` for dynamic prototype-scoped bean retrieval, ensures that each service execution is isolated and correctly configured with its own infrastructure components.

---

### Data Modeling with Fluent API

The framework utilizes a immutable `ServiceModel` to transport data between services. This model is constructed using a **Builder Pattern** that provides a fluent API, making the creation and transformation of events intuitive and readable.

- **Immutability**: Once built, a `ServiceModel` cannot be modified, ensuring thread safety and preventing side effects as the object moves through different processing stages.
- **Generic Payloads**: The model supports generic payloads, allowing it to carry any data type while maintaining type safety during processor execution.
- **Fluent Construction**: Developers can easily chain properties such as `withSource()`, `withEvent()`, and `addMeta()`, which significantly improves code maintainability compared to traditional setters.

---

### Dynamic Orchestration and Dispatching

The `ServiceEventManagerController` acts as the framework's central orchestration engine. It provides a REST-based gateway that accepts incoming events and dynamically routes them to the appropriate business functions.

- **Type-Safe Payload Conversion**: The controller automatically converts generic event payloads into the specific Java types expected by the target method using a high-performance `ObjectMapper` configuration. This allows developers to work with strongly-typed objects while the framework handles the underlying JSON serialization.
- **SpEL-Based Routing**: Before execution, the framework evaluates the `@EventProcessor`'s `condition` using Spring Expression Language (SpEL) against the incoming payload. This enables sophisticated, data-driven routing logic that can be updated without changing the service's core implementation.
- **Runtime Observability**: A built-in `/service-index` endpoint provides a real-time view of all active event sources and processors, including their associated SpEL conditions and active profiles. This transparency is critical for debugging complex orchestration pipelines in distributed environments.

---

### Startup Validation and Statelessness

To ensure architectural integrity and prevent common pitfalls in distributed systems, the framework includes a `ServiceWiringValidation` component that performs comprehensive reflection-based scanning at startup.

- **Statelessness Enforcement**: The framework warns if service classes contain fields that could hold state. In an event-driven architecture, services should be inherently stateless to allow for independent scaling and failure recovery.
- **Contract Verification**: At startup, every method annotated with `@PhysicalEventSource` or `@EventProcessor` is validated to ensure it follows the required method signatures (e.g., returning a `ServiceModel` and having exactly one argument for processors).
- **Payload Integrity**: The validator checks that all payload classes have default constructors, which is a prerequisite for successful JSON deserialization during the event propagation lifecycle.

---

### Advanced Serialization and Error Propagation

To maintain a robust audit trail and support effective troubleshooting, the framework includes custom Jackson modules for handling complex data types and exceptions.

- **Exception Preservation**: The `ExceptionModule` provides specialized serialization for `Throwable` objects, ensuring that full stack traces and causal chains are preserved as events move across the system. This allows for detailed post-mortem analysis even when errors occur in remote service instances.
- **Temporal Consistency**: The `DateTimeModule` ensures that `LocalDate` and `LocalDateTime` objects are serialized using a standardized ISO-8601 format with microsecond precision, preventing data loss or interpretation errors during cross-service communication.

---

### Resilience and Recovery Mechanisms

Observability and reliability are core pillars of the Service Framework. To handle failures in distributed environments, the framework includes a `ResilienceStore` layer integrated directly into the orchestration lifecycle.

- **Automated Persistence and Recovery**: The `ServiceEventManagerController` automatically persists every incoming event to the `ResilienceStore` before attempting to process it. Upon application restart, the `ServiceApplicationStarter` retrieves all persisted events and re-submits them to the orchestration engine, ensuring that no data is lost during system downtime. Once processing is successfully completed, the event is removed from the store.
- **Thumbprinting**: The framework generates unique SHA-512 thumbprints for events to identify duplicates and track processing history.
- **Persistent Storage**: Through the `FileResilienceStore`, events can be persisted to disk, allowing for recovery and re-processing in the event of a system crash or downstream service unavailability.
- **Dynamic Bean Lookup**: Using Spring's `@Lookup` annotation, the framework can dynamically retrieve prototype-scoped beans (like `RestTemplate`) at runtime, ensuring that each service execution has its own isolated and correctly configured infrastructure components.

---

### Configuration and Runtime Orchestration

The Service Framework is designed to be plug-and-play, leveraging Spring's auto-configuration capabilities. To enable the default components—such as `RestTemplate` for HTTP communication and a pre-configured `ObjectMapper` for JSON processing—applications should be launched with the `defaults-enabled` Spring profile.

```bash
--spring.profiles.active=defaults-enabled
```

At runtime, individual services are activated using event profiles. This allows for a single codebase to support multiple deployment configurations, where only specific handlers are loaded based on the environment or use case:

```bash
-Devent.profiles.active=jms-service,process-service,save-service,error-service
```

---

### Debugging and Observability

To trace service execution and diagnose routing issues, the framework exposes several key entry points. The `ServiceEventManagerAspect.evaluatePhysicalSource(...)` method is the primary location for monitoring how inbound events are initially evaluated. For services exposed via web hooks, `ServiceEventManagerController.consumeEvent(...)` provides visibility into manual trigger points. Monitoring these methods allows developers to verify that SpEL (Spring Expression Language) conditions are being met and that events are flowing through the orchestration pipeline as expected.

---

## Skills and Technologies

- **Core Technologies**: Java, Spring Boot, SpEL (Spring Expression Language), AOP (AspectJ).
- **Messaging & Communication**: JMS (MQ), REST (RestTemplate), JSON (ObjectMapper).
- **Architecture**: Event-Driven Design, Annotation-Based Configuration, Declarative Service Modeling, Fluent Builders.
- **Resilience & Security**: SHA-512 Thumbprinting, Resilience Stores, Persistence, SHA-512 Hashing.
- **Workflow & Orchestration**: BAU Workflow Orchestration, Conditional Routing, Custom Profiles.
