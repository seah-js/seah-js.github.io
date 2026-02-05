---
layout: post
title: "AWS ECS Deployment — From Git Push to Running Containers"
description: "Understanding the full CI/CD flow: GitHub Actions → ECR → ECS Fargate → ALB"
categories: infra
date: 2026-02-05
---

Learning session with [Klover](https://github.com/openclaw/openclaw). Today: deploying FastAPI + Streamlit to AWS ECS. The goal is understanding the full flow from `git push` to running containers.

---

## The Big Picture

```
git push
   ↓
GitHub Actions (CI/CD)
   ↓
ECR (store images)
   ↓
ECS + Fargate (run containers)
   ↓
ALB (route traffic)
   ↓
Users
```

Two services in my case: FastAPI backend + Streamlit frontend. Each gets its own container.

---

## ECS vs Fargate — What Actually Runs Containers?

**Klover:** ECS is an *orchestrator* — it doesn't run containers itself. What actually runs your containers?

**Me:** Maybe EC2?

**Klover:** That's one option. The other is Fargate.

- **EC2 mode:** You manage the instances, patching, scaling, capacity
- **Fargate mode:** Serverless. Just define CPU/memory, AWS handles the rest.

For someone who hates managing infrastructure, Fargate is the obvious choice.

---

## ECR — Where Images Live

**Klover:** Why not just use Docker Hub? Why does AWS have its own registry?

**Me:** Docker Hub is external to AWS. ECS runs within AWS, so having ECR inside AWS makes sense.

**Klover:** Exactly. Being inside AWS means:
- **Faster pulls** — same network, low latency
- **IAM integration** — no separate credentials
- **No egress costs** — pulling from Docker Hub = paying for data transfer

ECR = AWS's private Docker Hub.

---

## Task Definitions — The Blueprint

A Task Definition tells ECS everything it needs to run your container:

- **Image** — which ECR image to pull
- **CPU/Memory** — resources allocated
- **Port mappings** — which ports to expose
- **Environment variables** — config, secrets
- **Logging** — where to send logs
- **IAM role** — AWS permissions for the container

Think of it like a docker-compose file but AWS-flavored.

---

## Services — Keeping Things Alive

**Klover:** A Task Definition is just a blueprint. To actually run it, you create a Service. Why the separation?

**Me:** So if one crashes, the Service can spin up a replacement automatically?

**Klover:** Exactly. Service maintains desired state. You say "keep 2 copies running" — if one dies, Service notices and replaces it.

**Service gives you:**
- Desired count (keep N copies running)
- Auto-restart on failure
- Connection to load balancer

**Hierarchy:**
```
Task Definition (blueprint)
       ↓
Service (keeps N copies alive)
       ↓
Tasks (actual running containers)
```

---

## ALB + Target Groups — Routing Traffic

**Klover:** If you have 2 FastAPI tasks running, how does traffic get routed?

**Me:** Some kind of routing — the ALB?

**Klover:** Yes. ALB (Application Load Balancer) sits in front and distributes requests.

```
User request
     ↓
    ALB
   /   \
Task1  Task2
```

ALB does:
- **Load balancing** — spreads traffic across healthy tasks
- **Health checks** — pings `/health`, stops sending traffic to dead tasks
- **Path routing** — `/api/*` → FastAPI, `/` → Streamlit

### Why Target Groups?

Tasks are ephemeral. Their IPs change constantly. ALB can't track "send traffic to 10.0.1.45" when that IP might be gone in 5 minutes.

**Target Group = stable reference to an ever-changing set of tasks.**

ECS auto-registers new tasks, auto-deregisters dead ones. ALB just points at the target group.

---

## The CI/CD Flow

When I push code:

1. **GitHub Actions** detects the push
2. Workflow builds Docker image
3. Pushes image to **ECR**
4. Tells ECS to update the **Service**
5. Service pulls new image, spins up new **Tasks**
6. New tasks register in **Target Group**
7. **ALB** routes traffic to healthy tasks

### Why "update service"?

Pushing a new image to ECR just updates storage. Running tasks don't know — they're already running the old image.

"Update ECS service" triggers a rolling deployment:
1. Pull latest image
2. Spin up new tasks
3. Wait for health checks
4. Drain traffic from old tasks
5. Kill old tasks

No downtime.

---

## Secrets Management

Two different places for secrets:

**GitHub Secrets** — for CI/CD
- AWS credentials so GitHub Actions can push to ECR
- Used *during* build/deploy

**AWS Secrets Manager / SSM** — for runtime
- DB passwords, API keys the app actually uses
- Injected as environment variables when container starts
- Never in image, never in git

---

## Quick Reference

| Concept | What it does |
|---------|-------------|
| ECR | Stores Docker images |
| ECS | Orchestrates containers |
| Fargate | Serverless compute for containers |
| Task Definition | Blueprint (image, CPU, memory, ports, secrets) |
| Service | Keeps N tasks running, auto-restart |
| Task | Actual running container |
| ALB | Routes traffic, health checks |
| Target Group | Stable reference to dynamic tasks |
| GitHub Actions | CI/CD automation |
| Secrets Manager/SSM | Runtime secrets |

---

## The Gotcha I Hit

**Klover's quiz:** Fill in the blanks for the CI/CD flow.

I said the workflow pushes images to *ECS*. Wrong — it's *ECR*.

- EC**R** = **R**egistry (storage)
- EC**S** = **S**ervice (orchestration)

Easy to mix up. Now I won't.
