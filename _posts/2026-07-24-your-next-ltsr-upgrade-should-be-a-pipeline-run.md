---
layout: post
title: "Your next LTSR upgrade should be a pipeline run"
date: 2026-07-24
categories: [citrix]
tags: [citrix, iac, ansible, automation, devops, ltsr]
---

Every Citrix engineer knows the ritual. A new LTSR comes out, and somewhere in the following months there's a change window, a runbook with forty steps, a couple of long evenings and at least one moment where somebody asks "did we do this bit on the second Delivery Controller as well?"

I've done those evenings. I don't want to do them any more. So over the past years I've been moving towards a different model: the entire Citrix environment as code, rebuilt and upgraded through pipelines, with no manual changes in between.

## The two questions that started it

Two questions pushed me down this path.

First: if my environment disappeared tomorrow, could I rebuild it exactly as it was, purely from what's in version control? Not most of it. All of it.

Second: can I take the environment from one LTSR to the next without a single manual action? No installer wizards, no clicking through StoreFront consoles, no "quick fix" on a policy afterwards.

For a long time my honest answer to both was no. The environment was partly automated, which sounds fine until you realise that partly automated means partly manual, and the manual part is exactly where drift, mistakes and undocumented changes live.

## Infrastructure as Code, briefly

Infrastructure as Code means the environment is described in files, those files live in git, and tooling makes the real world match the description. The description is declarative: you say what the end state should be, not which buttons to press to get there. Run the code against an empty datacentre and it builds the environment. Run it against a healthy one and it changes nothing. That second part, idempotency, is the whole trick. Without it you have scripts. With it you have desired state.

A side effect I didn't fully appreciate up front: the repository becomes the documentation. Configuration questions turn into "read the code". Audit questions turn into "read the git log". Both are always current, because they are the thing itself rather than a description of it.

## How I build it up

My stack is Ansible for configuration, git for everything, and pipelines as the only path to production. The order I recommend, and mostly followed:

1. Version control first. All existing scripts and config exports go into a repository before anything else changes.
2. One component at a time. I started where the pain was highest: the VDA and golden image build. It's rebuilt constantly, so the automation pays for itself within weeks.
3. Idempotency as a hard requirement. Every playbook must be safe to run twice. If it isn't, it goes back to the drawing board.
4. Pipelines only. No playbook runs from a laptop. The pipeline has the credentials, the pipeline has the audit trail, the pipeline does the work.
5. Secrets in a vault. Nothing sensitive in the repository, ever.

After the image, the rest follows: Delivery Controllers and site config, StoreFront, Citrix policies, machine catalogues, delivery groups, certificates. Each component you move into code is a component that can never silently drift again.

And the LTSR upgrade? Once the installation is code, the version number is just a variable. Bump it, let the pipeline roll through the environment in the order you've defined, watch the checks go green. The forty-step runbook becomes a merge request.

## The exception: the licence server

One piece refuses to be fully automated: the Citrix License Server. The installation itself is code like everything else, but allocating licence files and linking the server to LAS (License Activation Service) still means logging in to the Citrix portal and doing it by hand. I've accepted it as the one documented manual step in the whole rebuild. If Citrix ever gives us an API for it, that step disappears too.

## Code coming

I'm cleaning up example code for a public repository so the Ansible side of this is something you can read rather than imagine. I'll link it here once it's up.

I've also written a companion piece on the Citrix community site that approaches the same subject from the "where do I start" angle. Between the two, the message is the same: put one component in code this month. The rebuild test gets easier every component after that.
