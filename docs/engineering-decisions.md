# Engineering decisions

Short records of the main design choices: the context, what we chose, and what it cost.

---

### 1. Reactive stack (WebFlux + reactive MongoDB) across all services

**Context.** Most requests spend their time waiting on I/O: other services, MongoDB, Redis, LLM APIs, telecom and email providers.
**Decision.** Spring WebFlux end to end, with reactive MongoDB and Redis drivers and `WebClient` for calls between services.
**Trade-off.** Fewer threads under load and natural back-pressure. The costs: a steeper learning curve for the team, and stack traces that are harder to read. We handled both with shared error-handling and logging conventions in the common libraries.

### 2. Security at the gateway, identity in Redis

**Context.** Ten services with six role levels and a multi-tenant hierarchy (platform → MSP → client → department → user).
**Decision.** Spring Cloud Gateway validates JWTs, resolves the session from Redis, checks a per-route permission table, and only then forwards the request with the caller's identity and client IP. Rate limiting also uses Redis at the gateway.
**Trade-off.** Downstream services stay simple and consistent. The gateway is now critical infrastructure, so it is kept free of business logic.

### 3. One queue per channel, not one queue per campaign

**Context.** Email, SMS, and voice have very different throughput, cost, and provider rate limits.
**Decision.** Separate SQS queues, producers, and consumers per channel. A campaign fans out into per-target messages on each channel it uses.
**Trade-off.** The channels are isolated from each other and each scales on its own. In exchange there are more queues to manage, which is why every queue is defined in Terraform (about 66 in total) rather than created by hand.

### 4. LLM generation as an async, schema-constrained step

**Context.** Content is generated per customer, and LLM latency and failures can't be predicted.
**Decision.** Generation runs on a queue worker. Prompts request structured JSON, which is validated before use, and calls use timeouts, retries with backoff, and rate-limit handling. GPT-4 and Gemini sit behind one internal interface.
**Trade-off.** Content isn't available instantly. In return, a slow or failing provider never blocks the API or campaign creation.

### 5. API / core / service modules in every service

**Context.** Services call each other and share DTOs with the frontend.
**Decision.** Each service publishes a thin `api` module (contracts only). Consumers depend on that module, never on another service's implementation.
**Trade-off.** More Gradle modules. But a contract change is a visible, reviewable diff, and the domain logic in `core` can be tested without a running application.

### 6. Infrastructure entirely as code, imported rather than recreated

**Context.** Some AWS resources had been created by hand before I took over the infrastructure.
**Decision.** I imported the existing resources into Terraform state and refactored them with `moved` blocks, so nothing in production was destroyed and recreated. Every new queue, repository, distribution, and secret now goes through Terraform.
**Trade-off.** The migration took careful work up front. Afterwards, environments are reproducible and every infrastructure change goes through review.

### 7. Immutable images and rollback by tag

**Context.** A small team ships often to four environments.
**Decision.** CI builds a versioned image per release, pushes it to ECR (with lifecycle policies to control cost), and deploys it to EKS. Rolling back means redeploying a previous tag.
**Trade-off.** Data changes have to stay compatible with the previous release so that a rollback is safe.
