+++
title = "Service Framework"
date = 2025-09-15
categories = ["Architecture", "Spring Boot"]
tags = ["Event-Driven", "Annotation-Based", "BAU Workflows"]
+++

[📥 browse this repository](https://dev.azure.com/mdcoopermc/_git/services-framework)

## Service Framework

The Service Framework enables the development of **event-based services** to orchestrate BAU workflows. It provides a
declarative, annotation-driven model for defining service behavior, routing logic, and execution conditions — all within
a Spring Boot environment.

---

### Configuration Overview

To activate default components like `RestTemplate` and `ObjectMapper`, launch your application with:

```bash
--spring.profiles.active=defaults-enabled
```

All services must consume or return an instance of `ServiceModel`. A fluent builder is provided to construct this
object.

To declare active services at runtime, use:

```bash
-Devent.profiles.active=jms-service,process-service,save-service,error-service
```

---

### Core Annotations

Each service must include:

- `@EventProfile` — Required for all services. Without this, the service will be ignored.
- `@PhysicalEventSource` — Marks an inbound event source.
- `@EventProcessor` — Defines a transformation step.
- `@PhysicalEventDestination` — Marks an outbound destination.

Services are only executed if:

- The **SpEL condition** matches the event object.
- The **payload type** matches the expected input.

---

### Debugging Entry Points

To trace service execution and event flow, use:

- `ServiceEventManagerAspect.evaluatePhysicalSource(...)`
- `ServiceEventManagerController.consumeEvent(...)`
- `ServiceEventManagerController.execute(...)`

---

### Example Service Chain

```java
public class DemoServices {

    @EventProfile("jms-service")
    @PhysicalEventSource(name = "MySource")
    @JmsListener(destination = "${mq.inbound}")
    public ServiceModel<String> source(@Payload final String message,
                                       @Headers final Map<String, Object> headers) {
        return serviceModel()
                .withEvent("JMS_RECEIVED")
                .withPayload(message)
                .build();
    }

    @EventProfile("process-service")
    @EventProcessor(condition = "event == 'JMS_RECEIVED'")
    public ServiceModel<String> processor(final ServiceModel<String> serviceModel) {
        return serviceModel(serviceModel)
                .withEvent("I_PROCESSED_THE_MESSAGE").build();
    }

    @EventProfile("save-service")
    @PhysicalEventDestination(condition = "event == 'I_PROCESSED_THE_MESSAGE'")
    public void save(final ServiceModel<String> serviceModel) throws IOException {
        Files.write(Paths.get("file.txt"), serviceModel.getPayload().getBytes());
    }

    @EventProfile("error-service")
    @PhysicalEventDestination
    public void error(final ServiceModel<HashMap> serviceModel) {
        log.error("Balls... " + serviceModel);
    }
}
```

---

### Status

- ✅ Modular annotation system in place
- ✅ Runtime orchestration verified
- ✅ Error handling and conditional routing supported
- 🚀 Ready for integration into larger orchestration pipelines

---

### Contact

Mark Cooper  
Senior Technology & Risk Leader  
[LinkedIn](https://www.linkedin.com/in/m-d-cooper)  
[GitHub](https://github.com/mdcooper-mc)