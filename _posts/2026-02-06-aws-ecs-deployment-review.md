---
layout: post
title: "AWS ECS Deployment — Review"
description: "Reviewing ECS concepts: what stuck, what didn't, and correcting misconceptions"
categories: infra
date: 2026-02-06
---

Review session with [Klover](https://github.com/openclaw/openclaw). Testing recall on AWS ECS deployment concepts from yesterday's session.

---

## The Recall Test

**Klover:** Walk me through the flow from `git push` to a user hitting your API. What are the key AWS services involved?

**Me:** After git push, if you set up CI/CD, the image will be pushed to AWS ECR, then deployed via ECS. After that I don't remember well. I know there's Fargate where you configure system requirements, ALB which routes containers to speak to one another, and some kind of task system.

**Klover:** Good start — you've got the big picture. But let's untangle the ECS side because you're mixing a few things.

---

## Task Definition vs Task

**Klover:** You mentioned "task system or something." What do you think a Task Definition actually is? Think blueprint vs running thing.

**Me:** Task Definition is the blueprint for the actual running of the containers? What settings for this service and everything like that.

**Klover:** Yes. Task Definition = blueprint. It defines which Docker image, how much CPU/memory, environment variables, port mappings, secrets. The actual running instance of that blueprint is called a **Task**.

**Me:** So is it accurate to say the Task is the running of the container, with configurations defined in the Task Definition?

**Klover:** Exactly. Same relationship as a Docker image vs a running Docker container.

---

## The Service — What I Forgot

**Klover:** What does an ECS Service do?

**Me:** I don't know.

**Klover:** What happens if your Task crashes? It's just dead. Nobody restarts it. That's what a **Service** does — it says "I want 3 Tasks running at all times." If one dies, the Service launches a replacement. It also handles rolling deployments.

**The hierarchy:** Task Definition (blueprint) → Service (manager) → Tasks (running containers)

---

## Crash Loop — My Wrong Assumption

**Klover:** If you push a broken image that crashes on startup, what happens?

**Me:** It will retry, and if it doesn't work, it will revert to the previous successful task definition.

**Klover:** Close but not quite — it's actually dumber than that. The Service sees the crash, launches a new one from the **same broken definition**. That crashes too. It tries again. And again. This is a **crash loop**.

It does NOT auto-rollback. The Service only knows "maintain desired count." You have to intervene manually.

**Mitigation:** Rolling deployments with health checks. New tasks must pass health checks before old ones are killed. Doesn't "revert" — just stops the rollout.

**Key takeaway:** The Service is persistent but not smart.

---

## ALB — What I Got Wrong

**Me:** ALB routes containers to speak to one another.

**Klover:** Not quite. ALB sits between **users and your containers**. Users hit the ALB, ALB routes traffic to healthy Tasks. Two reasons you need this middleman:

1. Tasks are ephemeral — they get replaced, scale up/down, IPs change
2. ALB gives you a **stable endpoint** (one URL) that always works

There's a **Target Group** between ALB and Tasks — a registry of currently healthy tasks that ALB routes to.

---

## Serverless ≠ No Servers

**Me:** Regarding Fargate, I don't understand what "serverless" means. Isn't EC2 also using AWS infra?

**Klover:** "Serverless" is confusing because obviously there ARE servers. The difference is who manages them:

- **EC2:** AWS gives you a VM. You manage capacity, patches, disk space, scaling.
- **Fargate:** You never see a machine. Just say "I need 0.5 vCPU and 1GB RAM." AWS figures out the rest.

Think of it like:
- EC2 = renting an apartment (you fix the plumbing)
- Fargate = hotel room (they handle everything)

Both use AWS infra. "Serverless" means you don't manage servers — not that servers don't exist.

---

## Self-Assessment: 2.7/5

**Solid on:** ECR → ECS pipeline, Task Definition = blueprint, Fargate = serverless.

**Fuzzy on:** ALB/Target Groups, Services, crash loop behavior, serverless meaning.

Staying at exposure. Review again tomorrow.
