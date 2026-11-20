# System design and behavioral interviews

## System design: order processing with .NET and Azure

### Question

Design an order-processing platform where clients submit an order, inventory and payment services process it, and the customer can check status. Use .NET, REST, Service Bus, AKS and APIM.

### Sample answer

1. **Clarify:** expected request rate and peak, latency, order durability, availability, regions, compliance, payment provider, ordering requirements, and whether inventory/payment must complete before the response. Assume asynchronous processing is acceptable, but confirm with the interviewer.
2. **Entry:** client -> APIM -> order API on AKS. Authenticate with Entra ID or the selected identity provider, authorize at the API, validate the request, and accept a client idempotency key.
3. **Persist:** in one database transaction create the order with `Pending` status, store the idempotency result, and write an `OrderCreated` outbox record. Return 202 with a status URL once the transaction commits.
4. **Publish:** an outbox worker publishes the event to Service Bus and marks the record sent. Duplicate publication is possible, so event IDs are stable.
5. **Process:** an orchestration service directs inventory reservation and payment through commands. Each service owns its data and deduplicates messages atomically with its local change. Results update the workflow and eventually the order status.
6. **Failure:** bounded retries for transient errors; dead-letter and alert on permanent failure. If payment fails after reservation, issue an idempotent inventory-release compensation. Reconcile stuck workflows and expose a safe failure status.
7. **Security:** use TLS, least-privilege managed/workload identities, Key Vault where secrets are necessary, network restrictions, and audit logs. Do not put payment-card details in events or logs.
8. **Operate:** propagate trace context, measure request latency, workflow completion time, outbox age, queue depth and dead-letter age. Scale workers based on backlog and downstream capacity. Deploy with compatible contracts, readiness checks and rollback.
9. **Trade-offs:** eventual consistency means the customer sees a pending state; outbox and deduplication add complexity; sessions may be needed for per-order sequencing but reduce parallelism per session. A synchronous flow may be appropriate for different requirements.

**Example flow:** `POST /orders` -> `202 Location: /orders/42` -> `OrderCreated` -> reserve inventory -> charge payment -> `GET /orders/42` returns `Paid`.

**Follow-up: What if the API commits but the HTTP response is lost?** The client retries using the same idempotency key. The server returns the stored result rather than creating a second order. Define behavior for concurrent calls and a key reused with a different payload.

**Example:** Both attempts use `Idempotency-Key: request-123`; the second returns the stored response for order 42.

**Follow-up: What if the payment provider times out?** The outcome may be unknown, not necessarily failed. Query the provider or retry with the same provider-supported idempotency key; do not create a new charge blindly. Reconcile unresolved outcomes.

**Example:** A charge with key `payment-order-42` times out; query the provider for that operation before deciding whether to retry.

**Follow-up: What if the region fails?** Discuss actual recovery objectives, replicated state, broker and database recovery behavior, failover and duplicate handling. Do not promise zero data loss without an architecture and tested guarantees that support it.

**Example:** A tested secondary region takes traffic after failover; reconcile recent events according to the actual replication and recovery guarantees.

## Additional scenario questions

### An API has p99 latency of five seconds. What do you do?

**Answer:** Confirm measurement and affected routes, compare recent changes and traffic, and trace slow requests across APIM, API, database and dependencies. Identify the largest contributor, check saturation and errors, and mitigate safely. Load-test the fix and compare p50/p95/p99 and error rate. Avoid optimizing a component without evidence.

**Example:** A five-second trace spends four seconds waiting on SQL; inspect its query plan and blocking before changing gateway settings.

### Two services need the same data. Should they share a database?

**Answer:** Prefer clear ownership and share data through APIs or events when independent deployment and boundaries matter. Shared databases create schema and release coupling, though they can be a pragmatic transitional choice. If using a local read model, address freshness, replay and reconciliation.

**Example:** The customer service owns customer data and publishes `CustomerUpdated`; the order service maintains a local read model for display, with a plan for delayed or missed events.

### A legacy system cannot publish events. How can you integrate it?

**Answer:** Use an adapter or anti-corruption layer to translate its model and protect the new domain. Depending on available capabilities, consider a supported change feed, polling with a durable checkpoint, or integration API. Define duplicate handling, ordering, failure recovery and security; avoid claiming polling gives instant updates.

**Example:**

```csharp
OrderStatus status = legacyStatus == "C"
    ? OrderStatus.Completed
    : throw new InvalidOperationException("Unknown legacy status");
```

The adapter translates the legacy value. A polling checkpoint can record which changes were processed; handle all valid legacy statuses in a real implementation.

## Behavioral: use STAR

Use **Situation**, **Task**, **Action**, **Result**. Give a concise example with your own actions, evidence and lessons. Do not invent experience or numbers.

### 1. Tell me about yourself.

**Sample structure:** "I am a .NET developer focused on [your actual strengths]. In my recent work I built or maintained [real API/service] using [actual technologies], where I contributed [specific action and outcome]. I am interested in this role because it combines API architecture, cloud integration and hands-on development."

**Illustrative template:** "I worked on [real .NET API], where I handled [your contribution]. I want to develop further in [relevant area]." Replace every bracket with your actual experience.

### 2. Explain a project and its business value.

**Sample structure:** "The problem was [business problem]. I owned [your contribution] and chose [architecture and trade-off]. We measured [actual result or qualitative evidence]. The main challenge was [issue], which I addressed by [action]." Be ready to draw the architecture and explain failures, security and deployment.

**Illustrative template:** "Our users needed [problem solved]. I implemented [component] and measured [actual result]; the trade-off was [real trade-off]."

### 3. Describe a production incident you handled.

**Sample structure:** Explain impact and your role; show how you investigated using evidence, mitigated safely, communicated, verified recovery and prevented recurrence. If you have not handled production incidents, describe a real test or development incident and state that clearly.

**Illustrative template:** "When [real incident] affected [users], I checked [evidence], helped [mitigation], and verified [recovery]. We later added [real safeguard]."

### 4. Tell me about a disagreement with a teammate.

**Sample structure:** Describe a real technical disagreement, how you listened and established shared criteria, how you tested alternatives, and the outcome. Focus on collaboration rather than blaming the other person.

**Illustrative template:** "We disagreed about [decision]. We compared [criteria], tested [options], and agreed on [actual outcome]."

### 5. Describe a mistake or failure.

**Sample structure:** State the mistake honestly, explain its impact, what you did to correct and communicate it, and a specific safeguard you introduced. Avoid pretending you never make mistakes.

**Illustrative template:** "I missed [real issue], which caused [impact]. I communicated it, fixed [action], and added [specific prevention]."

### 6. What is your biggest weakness?

**Sample structure:** Pick a genuine, relevant area for improvement, describe a concrete learning plan, and give evidence of progress. Avoid claiming a strength is a weakness or making an unsupported claim about yourself.

**Illustrative template:** "I am improving [genuine skill gap]. I practice by [real action], and my progress is [actual evidence]."

### 7. Describe a time your leadership was challenged.

**Sample structure:** Describe a real situation, the conflicting needs, your actions to align people and make a decision, and the result. Leadership can mean technical ownership or mentoring, not only managing people.

**Illustrative template:** "When [real conflict] blocked the team, I aligned [stakeholders] on [criteria] and helped deliver [actual outcome]."

### 8. Why Chevron, and why this role?

**Sample structure:** Connect your genuine interest in building reliable, secure systems for a large operational business to the published role and your own experience. Research the team and current company information before the interview; avoid inventing company facts or claiming personal experience you do not have.

**Illustrative template:** "This role's focus on [JD topic] interests me because of [genuine reason and relevant experience]. I would like to learn more about [team's actual work]."

### 9. How do you prioritize under a tight deadline?

**Answer:** Clarify business impact and non-negotiable security/reliability requirements, agree on a minimum scope, identify dependencies and risks, and communicate trade-offs early. Use a real example and explain what was delivered, deferred and learned.

**Illustrative template:** "For [real deadline], I agreed on [minimum scope], kept [security or reliability requirement], and communicated [deferred work]."

### 10. What if you do not know an answer?

**Answer:** Say what you know and what you are uncertain about, ask clarifying questions, and explain how you would verify the answer using documentation, a small experiment or a teammate. Do not claim experience with a service you have not used.

**Illustrative template:** "I have used [related technology], but not [specific feature]. My understanding is [what you know]; I would verify [uncertainty] in the documentation and a test."
## Questions to ask the interviewer

1. What business capabilities and integrations will this team own?
2. How are APIs, Service Bus consumers and AKS deployments operated today?
3. What are the most important reliability, security or scaling challenges?
4. What would success look like in the first three to six months?
5. What is the balance between hands-on coding, architecture and production support?
