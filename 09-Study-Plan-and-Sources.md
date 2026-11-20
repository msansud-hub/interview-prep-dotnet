# Study plan and sources

## Seven-day intensive plan

Adjust the time to your schedule. Each day: answer questions aloud, write code without notes, and record gaps.

| Day | Study | Hands-on practice |
|---|---|---|
| 1 | C# types, OOP, SOLID, generics, async/await and DI | Two sum, first unique character, valid brackets |
| 2 | Arrays, dictionaries, stacks, queues, trees, complexity | Binary search, linked-list reversal, one BFS problem |
| 3 | ASP.NET Core pipeline, REST, authentication and authorization | Design endpoints for orders, validation, error responses and idempotency |
| 4 | Microservices, events, outbox, saga, retries | Draw order workflow; explain crash-before-ack and duplicate handling |
| 5 | APIM, Service Bus, AKS, identity and Key Vault | Trace a request and a message; debug a growing queue scenario |
| 6 | SQL, EF Core, testing and CI/CD | Write the salary query; explain transaction and API integration tests |
| 7 | System design and behavioral | 45-minute mock design, 30-minute coding exercise, five STAR answers |

If you have more time, spend additional days implementing a small API and worker, writing integration tests, and practicing medium-level coding questions. Use a local environment or an authorized Azure subscription and check costs before deploying.

Study the follow-up questions within each topic file for extra depth.

## Mock interview checklist

- [ ] Explain value/reference types, async/await and scoped DI without confusing threads or lifetimes.
- [ ] Solve two coding problems with examples, edge cases and complexity.
- [ ] Design a secure REST API with validation, idempotency and appropriate status codes.
- [ ] Explain the outbox, duplicate delivery, atomic deduplication and a saga.
- [ ] Compare Service Bus queues, topics, sessions and dead-letter handling.
- [ ] Explain APIM policies, AKS probes, scaling and workload identity.
- [ ] Write a SQL join and ranking query; explain indexes and EF Core tracking.
- [ ] Explain CI/CD, safe deployment, monitoring and rollback.
- [ ] Prepare three genuine project/incident/collaboration stories and explain your personal contribution.

## Public interview sources

- [Glassdoor: Chevron Developer interview experiences](https://www.glassdoor.com/Interview/Chevron-Developer-Interview-Questions-EI_IE13524.0,7_KO8,17.htm) — the search result snippet reported themes about project/business value, fit, weakness and leadership. It was not independently verified by opening the page; reports can be old and may not match this role.
- [Glassdoor: Chevron Software Engineer interview experiences](https://www.glassdoor.com/Interview/Chevron-Software-Engineer-Interview-Questions-EI_IE13524.0,7_KO8,25.htm) — search result available, but the page returned HTTP 403 when accessed during research. No specific .NET question is attributed to it.
- [AmbitionBox: Chevron Holdings Software Developer](https://www.ambitionbox.com/interviews/chevron-holdings-interview-questions/software-developer) — search output included mixed or unrelated material; useful only as a lead to check, not reliable evidence of questions for this vacancy.
- [Glassdoor: Azure .NET Developer interview search](https://www.glassdoor.com/Interview/azure-net-developer-interview-questions-SRCH_KO0,19.htm) — broad, cross-company search results; not evidence of Chevron's interview.
- [.NET microservices interview topics](https://codewithmukesh.com/blog/microservices-interview-questions-dotnet/) — general practice topics, not a company-specific report.

## Technical references

- [Microsoft: ASP.NET Core overview](https://learn.microsoft.com/en-us/aspnet/core/introduction-to-aspnet-core)
- [Microsoft: C# fundamentals and concepts](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/)
- [Microsoft: Microservices design patterns](https://learn.microsoft.com/en-us/azure/architecture/microservices/design/patterns)
- [Microsoft: Azure Service Bus architecture best practices](https://learn.microsoft.com/en-us/azure/well-architected/service-guides/azure-service-bus)
- [Microsoft: Service Bus duplicate detection](https://learn.microsoft.com/en-us/azure/service-bus-messaging/duplicate-detection)
- [Microsoft: API Management policy reference](https://learn.microsoft.com/en-us/azure/api-management/api-management-policies)
- [Microsoft: API Management policies](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-policies)
- [Microsoft: AKS workload identity overview](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview)
- [Microsoft: EF Core documentation](https://learn.microsoft.com/en-us/ef/core/)

These are learning resources, not a guarantee of interview questions. Service capabilities and guidance can change; confirm details in current documentation and the applicable Azure service tier.

## Additional concept references

- [Microsoft: Delegates](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/delegates/)
- [Microsoft: Events](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/events/)
- [Microsoft: Polymorphism](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/object-oriented/polymorphism)
- [Microsoft: Generics](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/types/generics)
- [Microsoft: Asynchronous programming](https://learn.microsoft.com/en-us/dotnet/csharp/asynchronous-programming/)
- [Microsoft: Dependency injection in .NET](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection)
- [Microsoft: EF Core tracking queries](https://learn.microsoft.com/en-us/ef/core/querying/tracking)
- [Microsoft: AKS workload identity](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview)
