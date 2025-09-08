# Target Microservices Architecture - Block Diagram

This diagram illustrates the cloud-native, microservice-based target architecture for the modernized IoT platform, designed to run in a self-hosted, intranet environment.

```mermaid
graph TD
    subgraph "External World"
        iot_devices[IoT Devices]
        users[Users / UI]
    end

    subgraph "On-Premise Infrastructure"
        api_gateway[API Gateway]

        subgraph "Kubernetes Cluster"
            direction LR
            subgraph "Service Mesh (e.g., Istio)"
                ms_device[Device Management Service]
                ms_ingestion[Data Ingestion Service]
                ms_processing[Telemetry Processing Service]
                ms_alerting[Alerting Service]
                ms_users[User Management Service]
                ms_reporting[Reporting Service]
            end

            subgraph "Data Tier (Polyglot Persistence)"
                db_device[PostgreSQL - Devices]
                db_timeseries[InfluxDB/TimescaleDB - Telemetry]
                db_users[PostgreSQL - Users]
                cache[Redis - Cache/Sessions]
            end

            subgraph "Observability Stack"
                prometheus[Prometheus - Metrics]
                grafana[Grafana - Dashboards]
                efk[EFK Stack - Logging]
                jaeger[Jaeger - Tracing]
            end
        end

        subgraph "Event-Driven Backbone"
            kafka[Apache Kafka]
        end

        subgraph "CI/CD & Tooling"
            gitlab[GitLab / Jenkins]
            harbor[Harbor - Container Registry]
        end
    end

    %% Connections
    iot_devices -->|MQTT/HTTP| api_gateway
    users -->|HTTPS| api_gateway

    api_gateway --> ms_device
    api_gateway --> ms_ingestion
    api_gateway --> ms_users
    api_gateway --> ms_reporting

    ms_device <--> db_device
    ms_users <--> db_users
    ms_reporting <--> db_timeseries
    ms_processing <--> db_timeseries
    ms_processing <--> cache

    ms_ingestion --> kafka
    kafka --> ms_processing
    ms_processing --> kafka
    kafka --> ms_alerting

    %% CI/CD Flow
    gitlab --> harbor
    gitlab --> Kubernetes

    %% Observability Connections
    ms_device --> prometheus
    ms_ingestion --> prometheus
    ms_processing --> prometheus
    ms_alerting --> prometheus
    ms_users --> prometheus
    ms_reporting --> prometheus

    ms_device --> efk
    ms_ingestion --> efk
    ms_processing --> efk
    ms_alerting --> efk
    ms_users --> efk
    ms_reporting --> efk

    ms_device --> jaeger
    ms_ingestion --> jaeger
    ms_processing --> jaeger
    ms_alerting --> jaeger
    ms_users --> jaeger
    ms_reporting --> jaeger

    prometheus --> grafana
```

## Diagram Components Explained

*   **External World**: Represents the clients of the system.
    *   **IoT Devices**: The source of telemetry data, communicating via protocols like MQTT or HTTP.
    *   **Users / UI**: Human users interacting with the system through a web interface.

*   **On-Premise Infrastructure**: All components are self-hosted within the company's intranet.

*   **API Gateway**: The single entry point for all incoming traffic. It routes requests to the appropriate microservices and handles cross-cutting concerns like authentication, rate limiting, and SSL termination.

*   **Kubernetes Cluster**: The core of the platform, responsible for orchestrating and managing all the containerized applications.
    *   **Service Mesh**: Manages inter-service communication, providing features like load balancing, service discovery, retries, circuit breaking, and security (mTLS) automatically.
    *   **Microservices**:
        *   `Device Management Service`: Handles device registration, configuration, and status.
        *   `Data Ingestion Service`: A lightweight service responsible for receiving raw data from devices and publishing it to the Kafka backbone.
        *   `Telemetry Processing Service`: Consumes data from Kafka, performs transformations, applies business logic, and stores the results in the time-series database.
        *   `Alerting Service`: Monitors the processed data for specific conditions and triggers alerts.
        *   `User Management Service`: Handles user authentication and authorization.
        *   `Reporting Service`: Provides APIs for the UI to query data for reports and dashboards.
    *   **Data Tier**: Illustrates the polyglot persistence approach, where each service uses a database best suited to its needs.
        *   `PostgreSQL`: For relational data like device metadata and user information.
        *   `InfluxDB/TimescaleDB`: A time-series database optimized for storing and querying IoT telemetry data.
        *   `Redis`: An in-memory data store for caching and session management.
    *   **Observability Stack**: A crucial set of tools for monitoring and debugging the distributed system.
        *   `Prometheus`: Collects metrics from all services.
        *   `Grafana`: Visualizes metrics in dashboards.
        *   `EFK Stack`: Centralizes logs from all services.
        *   `Jaeger`: Provides distributed tracing to follow a request's lifecycle across multiple services.

*   **Event-Driven Backbone (Apache Kafka)**: Decouples the data ingestion from data processing. This creates a scalable and resilient buffer, allowing the system to handle bursts of data and for services to be independently developed and deployed.

*   **CI/CD & Tooling**: The automation backbone.
    *   `GitLab/Jenkins`: Automates the build, test, and deployment pipelines for each microservice.
    *   `Harbor`: A private container registry to securely store the Docker images of the microservices.
