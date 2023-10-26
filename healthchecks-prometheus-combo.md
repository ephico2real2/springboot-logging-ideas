### Various combinations


By default, the Spring Boot Actuator's health endpoints are available on the same port as the main application. So, if the application is running on port `8080`, then the health endpoint would indeed be available at `http://<host>:8080/actuator/health`.

In some configurations, how you can separate operational endpoints (like metrics and health checks) to a different port (`9090` in this case) by using the management port configurations in the `application.yml`.

However, this is optional. If you prefer to keep the health endpoints on the main application port (`8080`), you would not need to use the management port configurations. You would just rely on the default behavior of the actuator exposing health checks at `/actuator/health` on port `8080`.

To clarify:

- If you use the `management.server.port` setting in the `application.yml`, it will move all actuator endpoints, including health checks, to that port. So, with that configuration, the health endpoints would be on port `9090`.

- If you don't specify the `management.server.port`, then the actuator endpoints, including health checks, will remain on the main application port, which is `8080` in our example.

To address the `/actuator/health/readiness` and `/actuator/health/liveness` endpoints: These are specific probes exposed by Spring Boot when you enable the Kubernetes probes configuration via:

```yaml
management:
  health:
    probes:
      enabled: true
```

These endpoints are designed to align with Kubernetes liveness and readiness probes. However, the exact port they appear on (`8080` or `9090`) depends on whether or not you've configured the separate management port in the `application.yml`.


Below two primary cases:

### Case 1: App on Main Server Port (`8080`) and Both Prometheus & Health Checks on Admin Port (`9090`)

#### 1.1 Maven `pom.xml` Dependencies:
Ensure you have the necessary dependencies:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

#### 1.2 `application.yml` Configuration:
```yaml
server:
  port: 8080

management:
  server:
    port: ${ADMIN_PORT:9090}
  endpoints:
    web:
      exposure:
        include: prometheus, health, info
      base-path: /actuator
  metrics:
    export:
      prometheus:
        enabled: true
```

#### 1.3 Kubernetes Configuration:

**`deployment.yaml`**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "/actuator/prometheus"
        prometheus.io/port: "9090"
    spec:
      containers:
      - name: my-app
        image: my-app-image:latest
        ports:
        - name: http
          containerPort: 8080
        - name: admin
          containerPort: 9090
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: admin
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: admin
```

**`service.yaml`**:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
  ports:
    - name: http
      port: 8080
      targetPort: http
    - name: admin
      port: 9090
      targetPort: admin
```

**In this configuration**:
- Your main application will run on port `8080`.
- Both Prometheus metrics and health checks are exposed on the management (admin) port, which is `9090` by default.
- Prometheus will scrape metrics from the `/actuator/prometheus` endpoint on port `9090`.
- Kubernetes readiness and liveness probes will also target the health checks on port `9090`.


### Case 2: Both App and Health Checks on Main Server Port (`8080`), and Only Prometheus on Admin Port (`9090`):

#### 2.1 Maven `pom.xml` Dependencies:
Same as in Case 1.

#### 2.2 `application.yml` Configuration:
```yaml
server:
  port: 8080

management:
  server:
    port: ${ADMIN_PORT:9090}
  endpoints:
    web:
      exposure:
        include: prometheus
      base-path: /actuator
  metrics:
    export:
      prometheus:
        enabled: true
```

#### 2.3 Kubernetes Configuration:

**`deployment.yaml`**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "/actuator/prometheus"
        prometheus.io/port: "9090"
    spec:
      containers:
      - name: my-app
        image: my-app-image:latest
        ports:
        - name: http
          containerPort: 8080
        - name: admin
          containerPort: 9090
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: http
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: http
```

**`service.yaml`**:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
  ports:
    - name: http
      port: 8080
      targetPort: http
    - name: admin
      port: 9090
      targetPort: admin
```

### Case 2: Both App and Health Checks on Main Server Port (`8080`), and Only Prometheus on Admin Port (`9090`):

**In this configuration**:
- Your main application and health checks will run on port `8080`.
- Only the Prometheus metrics are exposed on the management (admin) port, which is `9090` by default.
- Prometheus will scrape metrics from the `/actuator/prometheus` endpoint on port `9090`.
- Kubernetes readiness and liveness probes will target the health checks on port `8080`.

For both configurations, remember the Maven dependencies ensure that Spring Boot's actuator and Prometheus's metrics capabilities are added to your application. The annotations in the Kubernetes deployment instruct Prometheus to scrape metrics from the designated path and port.

In both cases, make sure you adjust the image, deployment names, etc., to match your specific needs. The provided configurations are general examples based on the described scenarios.

## Prometheus and Metrics

`prometheus` and `metrics` in the context of Spring Boot Actuator and Micrometer refer to different things, and their distinction can be better understood in the broader context of monitoring and observability:

1. **metrics (in Actuator context)**:
   
   - **Definition**: Refers to the endpoint provided by Spring Boot Actuator that gives details about various application metrics. These metrics can be JVM metrics (like memory usage, garbage collection stats), system metrics, or custom metrics.
   
   - **Usage**: By accessing the `/actuator/metrics` endpoint, you can see a list of all available metrics. You can then drill down into individual metrics by appending their name, e.g., `/actuator/metrics/jvm.memory.used`.

   - **Role**: Provides a general view of metrics, but isn't tailored to any specific monitoring system.

2. **prometheus (in Actuator context with Micrometer)**:

   - **Definition**: Refers to the endpoint specifically tailored to provide metrics in a format that the Prometheus monitoring system can understand and scrape. It's an integration between Spring Boot applications and Prometheus made easier with the Micrometer library.
   
   - **Usage**: By including the `micrometer-registry-prometheus` dependency in your project and enabling the endpoint, metrics will be made available at `/actuator/prometheus` in a format suitable for Prometheus to scrape.
   
   - **Role**: Makes the integration with Prometheus straightforward. Prometheus will regularly scrape this endpoint to collect metrics and then you can visualize, alert, or analyze these metrics using Prometheus itself or other tools like Grafana.

**Broadly Speaking**:

- `metrics`: This is a general term referring to measurements or data points that represent the behavior, performance, or characteristics of a system. In the context of software, metrics can cover various things, from the number of active users, error rates, to system resource utilization.

- `Prometheus`: This is a specific monitoring system that collects and stores metrics. It has its own query language (PromQL) to query those metrics, create alerts, etc.

In the context of Spring Boot and Micrometer, the distinction between the `metrics` and `prometheus` endpoints is about the presentation of metrics. The `metrics` endpoint provides a more general and human-readable view, while the `prometheus` endpoint formats those metrics specifically for ingestion by the Prometheus system.
