

# On-Premises Retail Platform Documentation  

## Overview  
This on-premises retail platform serves internal and external businesses, ensuring security, availability, and operational visibility. Requests pass through a secure layer, authenticated by a user-authorization server, and are directed to microservices that utilize Redis for fast reads, Kafka for asynchronous communication, and PostgreSQL for data persistence. The observability layer provides real-time insights into system performance and failures, built on internal security controls and centralized observability.

![alt text](https://github.com/adsalvarez/ChallengeSSD/blob/feature/1.0.0/onpremisses.jpg?raw=true)

## Architecture Goals  
The architecture has this requirments:  
- Serve as a secure gateway for customers.  
- Separate responsibilities across layers.  
- Ensure high availability for critical components.  
- Enable microservices to scale independently.  
- Reduce latency through caching.  
- Utilize event-driven communication for service decoupling.  
- Ensure optimal data persistence.  
- Provide strong operational insights.

## Layer-by-Layer Architecture  

### 3.1 Perimeter and Edge Protection  
**Components**: 
    - Customer/Internal Users
    - Perimeter Firewall
    - WAF/API Protection
    - Load Balancer.  

Traffic from external users is filtered by a perimeter firewall, which prevents unauthorized requests. The WAF/API protection layer secures the platform from web-based attacks. The load balancer distributes traffic to Nginx gateways, enhancing availability and resilience.

### 3.2 Web and Access Layer  
**Components**: 
    - Nginx Gateways, 
    - Authorization Server Cluster.  

Nginx gateways handle incoming traffic, terminating TLS connections and directing requests. The Authorization Server Cluster manages identity and access control, centralizing authentication and token management to allow microservices to focus on business logic.

### 3.3 Security and Operations Layer  
**Components**: 
    - SIEM
    - PKI/Internal CA
    - Secret Vault.  
This layer enforces security governance. SIEM collects security data for visibility and incident detection. PKI manages internal certificates, ensuring secure communication. The Secret Vault stores sensitive information securely.

### 3.4 Observability Layer  
**Components**: 
    - Central Logging (Elasticsearch, Logstash, Kibana)
    - Prometheus
    - Grafana
    - Alertmanager
    - OpenTelemetry

The observability layer provides insights into system performance. Central logging consolidates logs for troubleshooting. Prometheus collects metrics for operational monitoring, while Grafana visualizes this data. Alertmanager manages alerts, and OpenTelemetry standardizes telemetry collection.

### 3.5 Microservices Layer  
**Components**: 
    - Customer Service
    - Order Service
    - Payment Adapter
    - Notification Service
    - Catalog Service
    - Inventory Service.  

This layer encapsulates business logic, allowing for maintainability and independent scaling. We can user docker or Kubernetes.

- **Customer Service**: Manages customer data and account logic.
- **Order Service**: Oversees the order lifecycle, tracking and managing interactions with inventory and payment.
- **Payment Adapter**: Isolates payment integration complexities from core services and we can outsource to a third party service.
- **Notification Service**: Handles outbound communications triggered by business events, utilizing Kafka for asynchronous processing.

This architecture prioritizes modularity, security, and efficiency in handling retail operations.


### Catalog Service. 
The **Catalog Service** maintains product information related to how products are displayed to customers and the internal systems of the organization. 
Responsibilities could include:
    - product listings. 
    - product attributes 
    - searchable catalog metadata. 
    - pricing references
    - integration hooks. 

Catalog has read-heavy patterns, so this is possibly the most promising for Redis-backed read optimization.
 
### Inventory Service. 
The **Inventory Service** can track the state and availability of inventory. responsibilities could include:
    - available stock levels. 
    - stock reservation or release. 
    - stock status checks at the time of order flow. 
    - publishing stock-change events. 

In consumer retail systems, a retailer’s inventory failures become apparent, particularly if there are orders for products that do not even exist that are not even present. Because of this, inventory logic needs to be well regulated with a strong integration of order-flow and into the event flow.

### Stock Sync Worker
The **Stock Sync Worker** synchronizes stock information across systems by:
    - Ingesting inventory-specific Kafka events.
    - Managing inventory across internal domains.
    - Updating the company based on external stock data.
    - Resolving consistency mismatches.

It addresses the asynchronous nature of inventory changes across systems.

### Fulfillment Worker
The **Fulfillment Worker** handles asynchronous order tasks post-order placement, including:
    - Processing workflows.
    - Arranging packaging or dispatch.
    - Responding to business signals.
    - Managing background orchestration.


## Database Layer

### Components
- PostgreSQL Primary
- PostgreSQL Standby
- Backup

The **database layer** offers durable transactional storage using PostgreSQL, ensuring high availability with replication and a backup for recoverability. It stores critical, non-cacheable business data such as orders and inventory, requiring persistent transactions.

## Messaging Layer

### Components
- Kafka Broker 1
- Kafka Broker 2
- Kafka Broker 3
- Kafka Schema Registry / Control Services

**Kafka** decouples services to enable parallel task processing, preventing service blocking. A three-broker architecture enhances availability and fault tolerance.
## Cache Tier

### Components
- Redis Primary
- Redis Replica
- Redis Sentinel

The **cache tier** improves performance by reducing backend reads, with active data managed by **Redis Primary** and redundancy provided by **Redis Replica**. **Redis Sentinel** monitors nodes for failover. Redis is effective for read-heavy data and short-lived access patterns, easing the load on PostgreSQL.

## High Availability Strategy

High availability is integrated across multiple architecture layers:

### Edge and Access
- Load balancer distributes requests across Nginx gateways.
- Multiple gateways prevent single points of failure.

### Authorization
- Central authorization server ensures continued access.

### Application
- Microservices isolate failures better than monolithic designs.

### Database
- PostgreSQL's primary/standby setup and backups enhance recovery.

### Messaging
- Kafka's brokers and replicated partitions improve resilience.

### Cache
- Redis employs primary, replica, and Sentinel for monitoring and failover.



## Security Strategy

Security is embedded throughout the architecture:

### Network Security
- Perimeter firewall and WAF secure traffic.

### Identity and Access
- Centralized token management and OIDC/OAuth2 patterns streamline access control.

### Secrets and Trust
- Secret vaults and PKI support secure communication.

### Monitoring and Detection
- SIEM and centralized logging aid in event correlation and audits.

### Operational Hardening
- Rate limiting and TLS termination mitigate risks.



## End-to-End Request Flow

1. Customer request passes through a perimeter firewall and WAF, then is forwarded by the load balancer.
2. Nginx routes the request, terminating TLS and implementing rate limiting.
3. If needed, the request is sent for authentication.
4. The relevant microservice processes the request, interacting with PostgreSQL, Redis, or Kafka as needed.


## Conclusion

This design balances control and modularity, keeping sensitive functions like identity and observability within organizational control. It efficiently handles asynchronous processes and ensures availability through replication and backups, while providing robust operational visibility.

The proposed on-premises architecture is resilient and well-structured, featuring:
- Protective edges.
- Routing gateways.
- Controlled access via an authorization server.
- Domain-oriented microservices.
- Kafka for decoupled and assynchronous workflows.
- Redis for accelerated reads.
- PostgreSQL for reliable data storage.
- Comprehensive observability.

This architecture effectively supports enterprise needs for availability, auditability, and operational maturity.