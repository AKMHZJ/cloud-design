# Cloud-Design — Full Execution Plan (Bonus Excluded)

This is the build order for the whole project, start to finish. The order matters —
each phase depends on the one before it being solid. Don't skip ahead; a broken
foundation (networking, especially) will cause confusing failures three phases later
that look unrelated to their real cause.

---

## Phase 0 — Ground Yourself Before Touching AWS

1. Re-read the full project brief once more and write down, in your own words, the
   5 design pillars (scalability, availability, security, cost-effectiveness,
   simplicity). You'll be tested on these in the role-play — know them cold.
2. List what you're reusing from prior projects:
   - `crud-master` → app code for inventory-app, billing-app, api-gateway-app
   - `play-with-containers` → your existing Docker/K8s manifests and lessons learned
     (volume permissions, networking fixes)
   - `orchestrator` → reference K8s deployment structure
3. Decide **ECS vs EKS now**, in writing, with a one-paragraph justification. This
   decision shapes almost every later phase (deployment manifests, IAM roles,
   networking specifics), so don't leave it open.
4. Sketch a rough architecture diagram by hand or in draw.io/Excalidraw before
   writing any Terraform. You'll refine it, but you need a target to build toward.

**Checkpoint:** You can explain, out loud, what you're building and why, without
looking at notes.

---

## Phase 1 — AWS Account & Terraform Foundation

1. Set up (or confirm access to) an AWS account with billing alerts turned on
   **before** provisioning anything — this is non-negotiable given the cost
   management requirement.
2. Create an IAM user/role for Terraform to use (not your root account).
3. Install Terraform locally, initialize a project repo, set up remote state
   (S3 bucket + DynamoDB lock table) so state isn't just sitting on your laptop.
4. Set a budget alert (e.g. AWS Budgets) at a low threshold so you get notified
   early if something runs away.

**Checkpoint:** `terraform init` and `terraform plan` run cleanly against an empty
AWS environment, and you have a billing alert confirmed active.

---

## Phase 2 — Networking (VPC) — Do This Before Any Compute or DB Resources

This is the phase where most later security requirements get satisfied. Get it
right now and Phase 5 (security) becomes mostly verification, not rework.

1. Define the VPC in Terraform.
2. Create **public subnets** (for the load balancer / api-gateway entry point) and
   **private subnets** (for databases, RabbitMQ, and — depending on your design —
   the internal apps).
3. Set up an Internet Gateway for the public subnets and a NAT Gateway (or NAT
   instance, cheaper) so private subnets can reach the internet for updates
   without being reachable from it.
4. Define route tables tying it together correctly (public subnet → IGW, private
   subnet → NAT).
5. Define security groups as their own Terraform resources now, even if empty —
   you'll attach rules per-service as you build each one. Plan the rules on paper
   first: which service is allowed to talk to which, on which port.

**Checkpoint:** `terraform apply` produces a VPC with public/private subnets you
can see in the AWS console, and you can articulate why each subnet is public or
private.

---

## Phase 3 — Databases (inventory-db, billing-db)

1. Decide: self-managed Postgres on EC2 (matches your play-with-containers
   experience) or Amazon RDS (managed, less to break, easier to justify in the
   role-play on the "availability" pillar).
2. Place both databases in **private subnets only** — no public IP, no route to
   the internet gateway.
3. Security group rule: only allow inbound port 5432 from the security groups of
   the app services that need it (inventory-app → inventory-db,
   billing-app → billing-db) — not from each other's databases, not from the
   internet.
4. Provision via Terraform, not console clicks — everything must be reproducible.
5. Test connectivity from inside the VPC only (e.g. via a bastion host or SSM
   Session Manager) — confirm you genuinely cannot reach the DB from outside.

**Checkpoint:** Both databases are up, reachable only from inside the VPC, and you
can demonstrate (screenshot or terminal) that an external connection attempt fails.

---

## Phase 4 — Containerize the Microservices

1. Write/refine a Dockerfile for each of: inventory-app, billing-app,
   api-gateway-app. Reuse and optimize what you already built in
   play-with-containers rather than starting from zero.
2. Use multi-stage builds where applicable to shrink final image size.
3. Build and run each image **locally** first, pointed at local/dummy config, to
   confirm they still work standalone before AWS enters the picture.
4. Create an ECR (Elastic Container Registry) repo per service via Terraform.
5. Push each built image to its ECR repo.

**Checkpoint:** All three images exist in ECR and you've run each one locally at
least once to confirm the container itself isn't broken — isolates "my code is
broken" from "my AWS config is broken" later.

---

## Phase 5 — Message Queue (RabbitMQ)

1. Decide: Amazon MQ (managed RabbitMQ) or self-hosted RabbitMQ container on
   ECS/EKS. Managed is simpler to justify for availability; self-hosted mirrors
   your existing K3s setup.
2. Place it in a private subnet, same logic as the databases — only billing-app
   (and whichever service publishes to it) should have network access.

**Checkpoint:** RabbitMQ is reachable from inside the VPC only, and you know
exactly which services are allowed to talk to it.

---

## Phase 6 — Orchestration & Deployment (ECS or EKS)

Only start this once Phases 2–5 are solid — deploying compute before networking
and dependencies exist is what causes the most confusing debugging sessions.

1. Provision the cluster (ECS cluster, or EKS cluster + node groups) via
   Terraform.
2. Define task definitions / Kubernetes manifests for inventory-app and
   billing-app, pointing them at their respective databases and (for billing-app)
   RabbitMQ via internal service discovery / DNS — not hardcoded IPs.
3. Deploy inventory-app and billing-app first, **without** the gateway yet.
   Confirm each can reach its database and (for billing-app) the queue, using
   logs/internal test calls.
4. Deploy api-gateway-app last, since it depends on the other two being
   reachable. Route it through an Application Load Balancer in the public
   subnet.
5. Confirm end-to-end request flow: external request → ALB → api-gateway →
   inventory-app/billing-app → database.

**Checkpoint:** You can hit the gateway's public endpoint and get a real response
that round-trips through the backend services and databases.

---

## Phase 7 — Authentication (Cognito)

1. Set up an AWS Cognito user pool (or equivalent) for the publicly accessible
   parts of the app.
2. Wire it in front of the API Gateway / gateway app so unauthenticated requests
   are rejected before they reach your services.
3. Test both paths: a valid authenticated request succeeds, an unauthenticated
   one is cleanly rejected (not a 500, a proper 401/403).

**Checkpoint:** Public endpoints require valid auth; internal service-to-service
calls still work without needing to pass through Cognito again.

---

## Phase 8 — Monitoring & Logging

1. Enable CloudWatch for basic metrics/logs on all compute resources first —
   it's the lowest-effort baseline and AWS-native.
2. Layer in Prometheus + Grafana for deeper application-level metrics and
   dashboards.
3. Set up the ELK stack (or CloudWatch Logs Insights, if you want to keep it
   simpler) for centralized log search across services.
4. Create at least one dashboard and one alert (e.g. high error rate, DB CPU
   spike) so monitoring is demonstrably functional, not just installed.

**Checkpoint:** You can pull up a dashboard showing live metrics, and you can
trigger/show an alert firing.

---

## Phase 9 — Auto-Scaling & Load Testing

1. Define auto-scaling policies (target tracking on CPU/memory, or request count
   for the gateway) via Terraform.
2. Run a basic load test (e.g. `k6`, `hey`, or `ab`) against the gateway endpoint.
3. Observe and record: does it scale out under load? Does it scale back in after?
4. Tune thresholds based on what you observe — don't leave default values
   unverified.

**Checkpoint:** You have before/after evidence (screenshots, metrics) of scaling
actually happening under load.

---

## Phase 10 — Security Hardening Pass

By now the architecture is functionally complete — this phase is about
tightening what's already there, not building new things.

1. AWS Certificate Manager: issue a cert, enforce HTTPS on the ALB/gateway.
2. Re-audit every security group: confirm nothing is wider than it needs to be
   (no `0.0.0.0/0` on anything except the ALB's public-facing port).
3. Run AWS Inspector (or equivalent) against your compute resources and images;
   address anything critical/high.
4. Confirm again — explicitly — that databases, RabbitMQ, and any private
   resources have zero route from the public internet.
5. Confirm encryption at rest is enabled on databases and any storage (S3, EBS).

**Checkpoint:** Every item in the project's "Security" requirement list has a
specific answer you can point to, not a general "yes we did security" claim.

---

## Phase 11 — Cost Review

1. Go through the AWS Billing dashboard and identify what's actually costing
   money.
2. Kill anything left over from earlier testing (old EC2 instances, unused EIPs,
   orphaned EBS volumes, old load balancers) — these are the classic silent
   cost leaks.
3. Confirm your budget alert from Phase 1 is still active and correctly
   thresholded now that real resources exist.

**Checkpoint:** No orphaned resources; billing dashboard matches what you expect
to be running.

---

## Phase 12 — Documentation (README.md)

Do this last, once the architecture is stable — writing docs against a moving
target wastes effort.

1. Write the final architecture diagram (clean version of your Phase 0 sketch,
   now reflecting what you actually built).
2. Document each component, the design decisions behind it, and how it satisfies
   each of the 5 pillars.
3. Include prerequisites, setup steps, configuration, and usage instructions —
   someone else should be able to follow it and reproduce your deployment.
4. Cross-check the README against the submission checklist: source code,
   IaC files, container configs, orchestration manifests all referenced/included.

**Checkpoint:** A stranger could read the README and rebuild your environment
without asking you anything.

---

## Phase 13 — Role-Play Prep

1. Go through each of the 5 pillars and prepare a 30-second spoken explanation
   of how your architecture satisfies it.
2. Prepare to justify your ECS-vs-EKS decision, your database choice, and your
   auto-scaling thresholds — these are the most likely "why did you do it this
   way" questions.
3. Think of one alternative approach you *didn't* take for each major decision,
   and why you didn't — this is explicitly called out as something they'll probe.

**Checkpoint:** You can defend the architecture to a skeptical stakeholder without
notes.

---

## Order-of-Operations Summary (quick reference)

```
0. Understand & decide architecture (ECS vs EKS)
1. AWS account + Terraform state setup
2. VPC / networking (public + private subnets, security groups)
3. Databases (private subnet only)
4. Containerize apps → push to ECR
5. RabbitMQ (private subnet only)
6. Deploy apps to ECS/EKS → deploy gateway last → verify end-to-end
7. Cognito auth in front of public endpoints
8. Monitoring & logging (CloudWatch → Prometheus/Grafana → ELK)
9. Auto-scaling + load test
10. Security hardening pass (HTTPS, SG audit, Inspector)
11. Cost review / cleanup
12. Documentation
13. Role-play prep
```
