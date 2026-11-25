# Engineering decisions

Short records of the main design choices: the context, what we chose, and what it cost.

---

### 1. Reactive stack (WebFlux + reactive MongoDB) across all services

**Context.** Most requests spend their time waiting on I/O: other services, MongoDB, Redis, LLM APIs, telecom and email providers.
**Decision.** Spring WebFlux end to end, with reactive MongoDB and Redis drivers and `WebClient` for calls between services.
**Trade-off.** Fewer threads under load and natural back-pressure. The costs: a steeper learning curve for the team, and stack traces that are harder to read. We handled both with shared error-handling, logging, and Reactor context propagation in the common libraries.

### 2. Security at the gateway, identity in Redis

**Context.** Ten services with six role levels and a multi-tenant hierarchy (platform → MSP → client → department → user).
**Decision.** Spring Cloud Gateway validates JWTs, resolves the session from Redis, checks a per-route permission table, and only then forwards the request with the caller's identity and client IP. Rate limiting also uses Redis at the gateway.
**Trade-off.** Downstream services stay simple and consistent. The gateway is now critical infrastructure, so it is kept free of business logic.

### 3. One queue per workload and environment, each with a dead-letter queue

**Context.** Email, SMS, voice, LLM generation, speech-to-text, and video rendering have very different throughput, cost, and provider rate limits.
**Decision.** Separate SQS queues, producers, and consumers per workload, repeated per environment. Every queue has a DLQ with a redrive policy (5 receives) and its own least-privilege IAM policy. A campaign fans out into per-target messages on each channel it uses.
**Trade-off.** Workloads are isolated from each other, each scales on its own, and an environment can't read another environment's queues. In exchange there are many queues to manage (about 66), which is why every one is defined in Terraform rather than created by hand.

### 4. Polling consumers that let SQS do the retrying

**Context.** Provider failures are common and usually temporary. A worker that crashes mustn't lose messages.
**Decision.** Each channel has its own scheduler that long-polls in batches, with a visibility timeout sized to the slowest delivery. A message is deleted only after it's processed. On failure it stays on the queue, reappears when the timeout expires, and SQS dead-letters it after the maximum number of receives.
**Trade-off.** No retry state lives in application memory, and a crashed worker loses nothing. The cost is at-least-once delivery: a message can arrive twice, so consumers need to be idempotent. That's the next improvement, and it's shown in the [open-source reference](https://github.com/shubhajit93/event-driven-campaign-platform).

### 5. LLM generation as an async, schema-constrained step behind one adapter

**Context.** Content is generated per customer, LLM latency and failures can't be predicted, and the best model changes often.
**Decision.** Generation runs on queue workers. OpenAI, Gemini, and a third provider implement one adapter interface, and a factory chooses the provider. Prompts request structured JSON (with a response schema where the provider supports it), which is validated before use. Prompts and keys are kept in SSM Parameter Store, so they can change without a deploy.
**Trade-off.** Content isn't available instantly. In return, a slow or failing provider never blocks the API or campaign creation, and switching models is a configuration change.

### 6. API / core / service modules in every service

**Context.** Services call each other and share DTOs with the frontend.
**Decision.** Each service publishes a thin `api` module (contracts only). Consumers depend on that module, never on another service's implementation.
**Trade-off.** More Gradle modules. But a contract change is a visible, reviewable diff, and the domain logic in `core` can be tested without a running application.

### 7. Infrastructure as code, imported rather than recreated

**Context.** Some AWS resources had been created by hand before I took over the infrastructure.
**Decision.** I imported the existing resources into Terraform state and refactored them with `moved` blocks, so nothing in production was destroyed and recreated. Every new queue, repository, distribution, and secret now goes through Terraform.
**Trade-off.** The migration took careful work up front. Afterwards, every infrastructure change goes through review, and a new environment's AWS resources come from code.

### 8. Moving compute from EKS to Docker Compose hosts

**Context.** The services first ran on an EKS cluster with a VPC, NAT gateway, node groups, and ElastiCache, all in Terraform. For 10 services with predictable load, the fixed cost of the cluster, NAT, and managed Redis was high compared with the workload.
**Decision.** In early 2026 I moved the services to Docker Compose on one VPS host per environment, set up by versioned scripts and runbooks, behind Nginx with automatic TLS renewal. The managed AWS services (SQS, SES, S3, CloudFront, ECR, secrets) stayed.
**Trade-off.** A substantial cut in infrastructure cost and one less control plane to run. In exchange we gave up Kubernetes features such as autoscaling and self-healing pods. We cover that with health checks, a database replica set, and fast redeploys, and the EKS setup can come back if the load grows.

### 9. Immutable images, tests before build, rollback by version

**Context.** A small team ships often to four environments.
**Decision.** Every push runs unit, integration, and functional tests. Only then does CI bump the version, tag the commit, build an image tagged per environment, push it to ECR (with lifecycle policies to control cost), and deploy it over SSH with Docker Compose. Each branch maps to one environment, and production deploys only from `main`. A rollback is a manual run that names a previous version. It skips the build and redeploys that image.
**Trade-off.** Data changes have to stay compatible with the previous release so that a rollback is safe.

### 10. Observability in a shared library, not per service

**Context.** Debugging a request that crosses the gateway, several services, and a queue needs the same logs, metrics, and trace IDs everywhere.
**Decision.** A shared observability module gives every service the same setup: JSON logs to Loki, Prometheus metrics, health probes, and OpenTelemetry traces over OTLP (full sampling outside production, 10% in production). Grafana sits on top.
**Trade-off.** Every service upgrades its observability stack together. That's a feature for consistency, but it means the module has to be versioned carefully.
