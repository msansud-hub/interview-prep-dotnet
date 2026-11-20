# Azure integration and cloud engineering

## Azure fundamentals

### 1. Which Azure services would you use for this job description?

**Answer:** AKS runs containerized .NET services; API Management (APIM) provides an API gateway and management layer; Service Bus provides enterprise messaging; Azure Container Registry stores container images; Key Vault stores secrets and keys; Microsoft Entra ID provides identity; Azure Monitor and Application Insights support observability. Azure SQL or another database provides storage, and a CI/CD platform deploys the system. Select services based on requirements, not because every application needs all of them.

**Example:** `Client -> APIM -> .NET API on AKS -> Service Bus -> worker`; Key Vault supplies required secrets, and Azure Monitor records telemetry.

### 2. App Service versus Azure Functions versus AKS?

**Answer:** App Service is a managed hosting option for web applications and APIs. Azure Functions suits event-driven execution and scheduled tasks, with hosting and scaling depending on the plan. AKS offers Kubernetes orchestration and control for containers but adds operational complexity. Choose based on workload, runtime needs, team skills, scaling, and cost.

**Example:** Host a simple HTTP API on App Service, a scheduled cleanup function in Azure Functions, or a group of containerized services on AKS when Kubernetes control is justified.

### 3. What is managed identity?

**Answer:** It is an identity managed by Microsoft Entra ID for an Azure resource or workload to authenticate to supported services without embedding long-lived credentials. Grant only the required Azure RBAC or service permissions. Verify network access and token audience as well as identity; managed identity does not automatically grant authorization.

**Example:** An App Service managed identity receives permission to read one Key Vault secret; the application requests a token instead of storing a client secret.

#### Follow-up: Why use managed identity instead of putting a password in code?

**Answer:** Managed identity lets a workload request tokens for supported Azure services without embedding a long-lived credential. The target service still needs to grant the identity appropriate permissions, and network access must also work.

**Example:** Grant an application's identity permission to read the required Key Vault secret. The application uses Azure Identity to authenticate; it does not store a vault password in source code.

### 4. How would a .NET service access Key Vault?

**Answer:** Give the workload a managed identity, grant the minimum secret access through the configured authorization model, and use Azure Identity with the Key Vault SDK or configuration provider. Configure network access, rotation and monitoring. Never commit secrets; do not log secret values. For local development, use a developer identity with appropriate access.

**Example:** With Azure.Identity and Azure.Security.KeyVault.Secrets:

```csharp
var client = new SecretClient(
    new Uri(vaultUrl),
    new DefaultAzureCredential());

KeyVaultSecret secret = await client.GetSecretAsync("ApiKey");
```

Configure the credential and permissions for the environment; never print the secret value.

## API Management

### 5. What does APIM do?

**Answer:** APIM provides a managed API gateway and management experience. It can publish APIs, apply policies for token validation, routing, transformations, rate limiting, and collect telemetry. Developer portals and products support API discovery and consumer management. Policies and capabilities can vary by tier and gateway, so verify your deployment.

**Example:** An inbound APIM policy fragment:

```xml
<rate-limit calls="100" renewal-period="60" />
```

This limits calls per subscription over a 60-second period. Configure it at the appropriate scope and check tier support.

### 6. What is the difference between APIM and your API's own authorization?

**Answer:** APIM can validate a token and enforce coarse gateway rules. The backend must still validate or securely trust identity under a well-defined architecture and enforce business and object-level permissions. Prevent bypassing the gateway where required, and protect the gateway-to-backend connection and identity.

**Example:** APIM validates the access token; the backend still checks whether the authenticated customer may read `/orders/42`.

#### Follow-up: If APIM validates a token, why does the backend still need checks?

**Answer:** A valid token proves identity and claims under the configured validation rules; it does not automatically prove the caller owns every order. The backend must enforce business and object-level permissions and either validate identity itself or securely trust a properly designed gateway-to-backend identity mechanism.

**Example:** APIM accepts user A's valid token. The order API still checks that user A may read order 42. Restrict direct backend access when your architecture relies on the gateway.

### 7. A client gets 429 from APIM. What do you check?

**Answer:** Identify which rate-limit or quota policy applied, its scope/key and the caller's traffic pattern. Check policy order, effective inherited policies and gateway telemetry. Clients should respect `Retry-After` where provided and use bounded backoff with jitter. Review limits and traffic needs rather than immediately disabling protection.

**Example:** A client exceeds a configured 100-calls-per-minute limit and receives `429`; investigate its call rate and policy scope before changing the limit.

### 8. How do you change backends without breaking clients?

**Answer:** Publish a stable API contract in APIM and route to the selected backend using policies and backend configuration. Test compatibility, authentication, error mapping, observability and latency. Roll out gradually and retain a tested rollback; APIM does not automatically fix an incompatible backend contract.

**Example:** Keep the public route `/orders/42` stable while changing the APIM backend from the old API to a compatible new API, then verify both response and authorization behavior.

## Azure Service Bus

### 9. Queue versus topic/subscription?

**Answer:** A queue supports competing consumers, where one consumer processes a given message delivery. A topic can fan out a published message to multiple subscriptions; each subscription has its own delivery and can use filters. Consumers still need to handle retries and duplicates.

**Example:** A queue `orders` distributes work among order workers; a topic `order-events` can deliver `OrderPlaced` to both `billing` and `notifications` subscriptions.

#### Follow-up: When should I use a Service Bus queue versus a topic?

**Answer:** Use a queue when workers compete to process messages from one work stream. Use a topic with subscriptions when multiple independent consumers need their own copy of an event.

```text
Queue:  order command -> one available worker handles a delivery

Topic:  OrderPlaced -> billing subscription
                    -> notification subscription
```

Each subscription has its own delivery lifecycle. Redelivery is possible in either approach.

### 10. Peek-lock versus receive-and-delete?

**Answer:** In peek-lock, the receiver locks a message and explicitly completes it after successful processing. If the lock expires or the receiver abandons it, the message can be delivered again. Receive-and-delete removes the message as it is transferred and risks loss if the receiver fails; use it only when that trade-off is acceptable.

**Example:** With a peek-lock receiver:

```csharp
var message = await receiver.ReceiveMessageAsync();
if (message != null)
{
    await ProcessAsync(message);
    await receiver.CompleteMessageAsync(message);
}
```

Complete the message only after processing succeeds; production code also needs failure handling and idempotency.

### 11. What happens when processing repeatedly fails?

**Answer:** A message can be redelivered until the configured delivery limit is reached, after which it can move to the dead-letter queue. It may also be dead-lettered for other reasons, depending on configuration. Inspect reason and description, alert on count/age, fix the failure, and replay carefully with idempotent consumers.

**Example:** A message repeatedly fails because a required order ID is missing -> it reaches the configured delivery limit -> operators inspect it in the dead-letter queue.

### 12. Does Service Bus duplicate detection eliminate all duplicates?

**Answer:** No. When enabled, it suppresses duplicate sends with the same application-controlled message ID within a configured history window. With partitioning, the partition key is also relevant; session/partition settings have constraints. It does not prevent consumer redelivery after processing but before completion, or duplicates outside the window. Keep consumers idempotent.

**Example:** Two sends with `MessageId = "event-42"` within the configured detection window can be deduplicated; a previously delivered message can still be redelivered after its lock expires.

#### Follow-up: Is Service Bus duplicate detection the same as consumer idempotency?

**Answer:** No. Duplicate detection can suppress repeated **sends** using the same message ID within a configured time window. It does not prevent all redeliveries to a consumer after a lock expires or cover duplicates outside the window.

**Example:** A sender retries `event-42` within the deduplication window and the broker drops the second send. Later, the consumer can still receive the first message again after failing to complete it. Consumer-side idempotency is still needed.

### 13. How do sessions help with ordering?

**Answer:** Use the same `SessionId` for related messages, such as a particular order. A session-aware receiver holds a session lock and processes the stream for that session, enabling ordered handling within that scope. Design for lock expiration, redelivery, producer ordering, retries and idempotent processing; do not promise global ordering.

**Example:**

```csharp
var message = new ServiceBusMessage("created")
{
    SessionId = "order-42"
};
```

A later message with the same session ID belongs to the same session on a session-enabled entity.

#### Follow-up: What are sessions, and do they guarantee global order?

**Answer:** Sessions group related Service Bus messages by `SessionId` so a session-aware consumer can process a related stream in order. They do not provide one global order across every session.

```csharp
var message = new ServiceBusMessage("Order updated")
{
    SessionId = "order-42",
    MessageId = "event-42"
};
```

The queue or subscription must be configured for sessions, and the receiver must be session-aware. Handle retries, lock expiry and duplicates; use sequence/version checks if your business process needs them.

### 14. How do you scale Service Bus consumers?

**Answer:** Increase consumer instances and concurrency within downstream capacity; partition work appropriately and use sessions carefully when order is required. Monitor backlog, oldest message age, throughput, lock loss, dead letters and downstream saturation. Tune prefetch and lock renewal to processing duration; excessive prefetch can cause locks to expire before processing.

**Example:** If arrivals are 100 messages/minute and workers complete only 60/minute, backlog grows by about 40/minute. Investigate capacity and downstream limits before scaling.

### 15. Service Bus versus Event Grid versus Event Hubs?

**Answer:** Service Bus is an enterprise message broker for queues and pub/sub with capabilities such as sessions and dead-lettering. Event Grid routes discrete events to subscribers. Event Hubs is a high-throughput event streaming and ingestion platform. Compare delivery, retention, ordering, throughput and consumer requirements before choosing.

**Example:** Use Service Bus for order commands, consider Event Grid for blob-created notifications, and Event Hubs for a high-volume stream of device telemetry.

### 16. Scenario: the consumer updates the database, then crashes before completing the message. What happens?

**Answer:** The lock eventually expires and the message can be redelivered. Without idempotency the update may happen twice. Record the event ID and business update atomically in the consumer's database; on redelivery recognize the ID and complete the message without repeating the effect. If external APIs are involved, use their idempotency keys or an outbox.

**Example:** `event-42: commit balance update + processed ID -> crash -> redelivery -> detect event-42 -> complete without updating balance again`.

#### Follow-up: Why can a message be delivered again after it was processed?

**Answer:** In peek-lock mode, the consumer processes a locked message and then completes it. If it commits its database work but crashes before completing the message, the lock can expire and the broker can redeliver it.

```text
Receive -> update database -> crash -> lock expires -> receive again
```

An atomic processed-event record protects the database effect. Handle the duplicate safely and complete the message.

## AKS and containers

### 17. What are a container, image, pod, deployment and service?

**Answer:** An image is a packaged application and its runtime dependencies; a container is a running instance. A pod is the smallest Kubernetes deployment unit, holding one or more tightly coupled containers. A Deployment manages replica sets and rolling updates. A Kubernetes Service gives a stable network endpoint for selected pods.

**Example:** `orders:v1` is an image; a Deployment starts three pods using it; a Service routes traffic to the pods selected by matching labels.

#### Follow-up: What is the difference between a pod and a Service in Kubernetes?

**Answer:** A pod runs one or more tightly related containers. A Kubernetes Service provides a stable network endpoint and selects matching pods. Pods can be replaced; the Service lets clients use a stable name instead of tracking individual pod addresses.

```text
orders Service
    -> ready orders pod 1
    -> ready orders pod 2
```

The Service's selector must match the pods' labels. This Kubernetes Service is not the same thing as Azure Service Bus.

### 18. How does traffic reach a .NET API running on AKS?

**Answer:** A common path is client -> APIM -> ingress/gateway -> Kubernetes Service -> ready API pod. Exact network topology varies; some services are internal-only. Configure TLS, network restrictions, identity, routing and health checks for every hop. APIM and Kubernetes ingress are complementary, not interchangeable.

**Example:** `GET /orders/42 -> APIM -> ingress -> orders Service -> ready orders pod`; inspect each hop if the client receives an unexpected gateway error.

### 19. Readiness versus liveness versus startup probes?

**Answer:** Readiness determines whether a pod should receive traffic. Liveness determines whether a container should be restarted. A startup probe protects slow-starting applications from premature liveness/readiness checks until startup succeeds. Use meaningful, inexpensive checks and avoid making liveness depend on every downstream system.

**Example:** A Kubernetes container spec can contain:

```yaml
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
```

Configure separate liveness and startup probes as needed.

#### Follow-up: Why do liveness and readiness probes need different checks?

**Answer:** Liveness asks, "Should this container be restarted?" Readiness asks, "Should this pod receive traffic?" An application can be alive but temporarily unable to serve requests.

```yaml
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
```

These are container-spec fragments; implement the endpoints separately. Making liveness fail whenever a shared database is briefly unavailable can cause unnecessary restarts. A startup probe can protect a slow-starting application.

### 20. How do you scale an AKS workload?

**Answer:** Horizontal Pod Autoscaler adjusts pod replicas based on configured metrics; KEDA can support event-driven scaling based on sources such as queue backlog. Cluster/node autoscaling can add or remove nodes when pods need capacity, subject to configuration and limits. Set CPU/memory requests and limits, ensure dependencies can handle load, and verify scaling with realistic tests.

**Example:** With suitable metrics and limits, HPA might increase an API from 2 to 4 pods under load; if nodes lack capacity, node autoscaling may add a node.

### 21. A pod is restarting or not receiving traffic. How do you investigate?

**Answer:** Inspect pod status and events, previous container logs, restart count and termination reason. Check configuration and secrets, image pulls, startup failures, resource limits/OOM, probes and dependency connectivity. For traffic, also check readiness, Service selectors/endpoints, ingress routing, DNS and network policies. Fix the evidenced cause and verify recovery.

**Example:**

```powershell
kubectl describe pod <pod-name>
kubectl logs <pod-name> --previous
```

Replace `<pod-name>` with the actual name; the first command shows events and the second shows previous-container logs. Use the correct cluster and namespace.

### 22. How do you deploy safely to AKS?

**Answer:** Build and scan a versioned immutable image, push it to a registry, and deploy declaratively using manifests, Helm or another controlled approach. Use readiness checks, appropriate rolling-update settings, resource constraints, monitoring, and a rollback plan. Consider schema compatibility and graceful shutdown. Verify image provenance and least-privilege cluster access.

**Example:** Promote a tested image by digest from staging to production, roll out gradually, and watch readiness and error rate before completing the rollout.

### 23. How does an AKS workload authenticate to Azure?

**Answer:** Microsoft Entra Workload ID can federate a Kubernetes service account with an Entra application or managed identity, allowing the workload to obtain tokens without storing a client secret. Configure federation, service account and pod settings, then grant least-privilege access to the target resource.

**Example:** Associate the `orders` Kubernetes service account with a managed identity via federated credentials, then grant that identity only the required Key Vault access.

## Integration, reliability and deployment

### 24. What is the role of CI/CD?

**Answer:** Continuous integration builds and tests changes frequently. Continuous delivery automates creation of a deployable artifact; continuous deployment additionally deploys changes automatically when quality gates pass. A typical pipeline restores pinned dependencies, builds, tests, scans, builds an image once, publishes it, then promotes that same artifact through environments with appropriate approvals and verification.

**Example:** `commit -> dotnet build -> dotnet test -> image scan -> publish image -> staging -> approval -> production`.

### 25. How would you protect pipeline secrets?

**Answer:** Prefer workload federation or managed identities over long-lived credentials, grant least privilege, store necessary secrets in Key Vault or a secure pipeline store, and restrict access. Avoid printing secrets in logs; rotate credentials and audit usage. Separate build and deployment permissions.

**Example:** A deployment pipeline obtains a short-lived federated token for its Azure identity instead of storing a client secret in its YAML.

### 26. What is infrastructure as code?

**Answer:** Define infrastructure declaratively using tools such as Bicep or Terraform, review changes in version control, validate and deploy through pipelines, and manage environment-specific configuration safely. Plan for state protection, drift detection and destructive-change review.

**Example:**

```bicep
param location string = resourceGroup().location
```

This Bicep parameter defaults to the resource group's location. Deploy reviewed infrastructure through a controlled pipeline.

### 27. What is the difference between availability zones and regions?

**Answer:** Availability zones are physically separate locations within a supported Azure region. Regions are geographically separate. Zone redundancy can protect against a zone outage; regional disaster recovery requires an explicit cross-region design with data replication, failover, recovery time and recovery point targets.

**Example:** Deploying replicas across zones 1 and 2 in one region can reduce a single-zone failure impact; a second region needs a separate tested failover plan.

### 28. Scenario: Service Bus backlog grows although AKS pods are healthy.

**Answer:** Compare incoming rate to processing throughput, message age, active sessions, consumer errors, lock loss and database/downstream latency. Healthy pods may be blocked, throttled or underprovisioned. Scale only if the downstream system has capacity; fix poison messages, lock duration, concurrency or bottlenecks as indicated. Check recovery and avoid duplicate side effects.

**Example:** `incoming = 200/min, processed = 80/min, database latency = 5 seconds`: inspect the database bottleneck before increasing worker replicas.

#### Follow-up: Why can a queue grow even if all pods show healthy?

**Answer:** Healthy pods are not necessarily fast enough. Messages may arrive faster than workers can process them, or workers may be waiting for a slow database or external API.

**Example:** Arrival rate is 200 messages per minute; processing rate is 80. The backlog grows by roughly 120 per minute. Check message age, failures, locks, worker concurrency, and database latency. Add workers only if downstream services can handle more load.

### 29. Scenario: an Azure API is slow only during traffic spikes.

**Answer:** Use APIM and distributed traces to separate gateway, network, application, database and downstream latency. Inspect rate limits, pod scaling, CPU/memory, thread pool, connection pools, query plans, queue depth and dependency throttling. Load-test a change, verify latency/error percentiles and cost, and make sure retries do not amplify the spike.

**Example:** If traces show most of the extra time waiting for database connections, investigate pool exhaustion and query duration rather than raising the APIM timeout first.

### 30. How do you manage costs without compromising reliability?

**Answer:** Measure usage and define service-level targets first. Right-size resources, autoscale where appropriate, review idle environments, retention and premium features, and use budgets/alerts. Validate any change with realistic load and recovery tests rather than reducing replicas or redundancy blindly.

**Example:** Stop an approved nonproduction environment outside working hours and measure savings, while retaining the production redundancy needed to meet its SLO.
