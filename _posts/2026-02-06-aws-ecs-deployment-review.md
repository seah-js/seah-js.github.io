---
layout: post
title: "AWS ECS Deployment — Review"
description: "Reviewing ECS concepts: what stuck, what didn't, and correcting misconceptions"
categories: infra
date: 2026-02-06
---

Review session with [Klover](https://github.com/openclaw/openclaw). Testing recall on AWS ECS deployment concepts from yesterday's session.

---

## What I Recalled

**Klover:** Walk me through the flow from `git push` to a user hitting your API.

**Me:** After git push, CI/CD builds the image and pushes to ECR, then deployed via ECS. Fargate is where you configure system requirements. ALB routes containers to speak to one another. There's some kind of task system.

Verdict: got the big picture, but fuzzy on half the details.

---

## What Stuck ✅

- **ECR → ECS pipeline** — the overall flow is solid
- **Task Definition = blueprint, Task = running container** — same relationship as Docker image vs running container
- **Fargate = serverless** — AWS manages the infrastructure

---

## What Didn't Stick ❌

### Services

Couldn't remember what a Service does at all.

**The answer:** A Service maintains your **desired task count**. If you say "keep 3 running" and one crashes, the Service launches a replacement. It also handles rolling deployments — gradually replacing old tasks with new ones during updates.

**Hierarchy:**
```
Task Definition (blueprint)
       ↓
Service (manager — maintains desired count)
       ↓
Tasks (running containers)
```

### ALB + Target Groups

I said ALB "routes containers to speak to one another." Wrong.

**The answer:** ALB sits between **users and your containers**. Users hit the ALB, ALB routes to healthy tasks. You need this because tasks are ephemeral — IPs change constantly. ALB gives you one stable URL.

Target Group = the registry of currently healthy tasks that ALB routes to.

### Crash Loop Behavior

**Klover:** What happens if you push a broken image?

**Me:** The Service retries, and if it doesn't work, it reverts to the previous task definition.

**Wrong.** The Service is persistent but **not smart**. It sees the task crash, launches another from the same broken definition, that crashes too, and it loops — a **crash loop**. It does NOT auto-rollback.

You have to intervene manually: fix the image or update the Service to use the old Task Definition.

**Mitigation:** Rolling deployments with health checks. New tasks must pass health checks before old ones are killed. Doesn't "revert" — just stops the rollout and keeps old tasks running.

---

## Serverless ≠ No Servers

I asked: isn't EC2 also using AWS infra? Why is Fargate "serverless"?

The difference is **who manages the servers**:

- **EC2:** AWS gives you a VM. You manage it — capacity, patches, disk space, scaling rules.
- **Fargate:** You never see a machine. Just declare CPU/memory needs. AWS handles placement.

Analogy:
- EC2 = renting an apartment (you fix the plumbing)
- Fargate = hotel room (they handle everything)

"Serverless" means you don't manage servers, not that servers don't exist.

**Tradeoff:** Fargate costs more per compute unit, but saves ops effort. For most apps, worth it.

---

## Self-Assessment

**Rating: 2.7/5**

The overall pipeline is there, but I need more reps on:
- Service role and behavior
- ALB / Target Group relationship
- Fargate vs EC2 at a deeper level

Staying at **exposure** level. Reviewing again tomorrow.
