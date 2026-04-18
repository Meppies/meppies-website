---
layout: post
title: "From second brain to team brain: a developer standards MCP"
date: 2026-03-29
categories: [ai, devops]
tags: [mcp, ai, developer-standards, claude, second-brain]
description: How a personal second brain pattern turned into a shared knowledge system for an entire development team, connected to every developer's AI through MCP.
---

A few weeks ago I sat in a DuCUG session by Brian Madden on AI in the modern workplace. He talked about the idea of a **second brain**: giving your AI a persistent store of who you are and how you work, so that every new session starts with context instead of a blank page. Not a prompt you paste every time. An actual, versioned, read-at-startup knowledge base.

I let that sit for a week. Then something clicked.

## What if you did this for a whole team?

Every day, developers ask their AI to write code. And every day, the AI has no idea how the team works. It doesn't know the stack. It doesn't know the project structure. It doesn't know that secrets belong in Vault, that every endpoint needs authentication, that the team uses OAuth 2.0 with OIDC and not an in-house token scheme.

So the AI gives generic answers. Developers get generic code. And somewhere, a senior engineer is writing the same review comment for the third time this week.

The usual defence against this is a wiki, a style guide, or a 40-page onboarding PDF. These do not solve the problem. They aren't read. They go stale. They sit in Confluence until someone does a "documentation sprint" and pretends it wasn't a waste of a day.

Connecting the same standards to an AI that developers *already use every day* is a different conversation.

## A quick detour: what is MCP

MCP, the [Model Context Protocol](https://modelcontextprotocol.io), is a small, open standard from Anthropic. It lets an AI tool connect to external context at runtime: a folder on disk, a database, an API, a git repo. The AI can read from those sources the moment a session starts, without needing them in a prompt.

For this pattern you only need one MCP server, the one Anthropic publishes themselves:

```json
{
  "mcpServers": {
    "dev-standards": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/path/to/dev-standards-starter"
      ]
    }
  }
}
```

Drop that in your Claude Desktop config, point it at a local folder, and the contents of that folder are available to the AI from session start. No cloud, no hosting, no back end. Just a folder and a JSON entry.

## The pattern: personal second brain, team edition

My own second brain is a vault of markdown files: who I am, how I like to work, what I'm currently thinking about, which projects are in flight. Claude reads it at the start of every session, so I never have to explain myself twice.

The team version of the same pattern is a vault of markdown files too, but the "who" is the team and the "how" is the standards:

```
dev-standards-starter/
├── STANDARDS.md                 # Master AI instruction file
├── starter-interview.md         # Guided setup conversation
├── standards/
│   ├── languages/
│   │   ├── primary-language.md
│   │   └── general.md
│   ├── infrastructure/
│   │   ├── iac.md
│   │   ├── config-management.md
│   │   └── pipelines.md
│   ├── security/
│   │   └── authentication.md
│   └── quality/
│       ├── code-review.md
│       ├── testing.md
│       └── documentation.md
├── templates/
│   └── README-template.md
└── methodology/
    ├── overview.md
    ├── decision-log.md
    └── onboarding.md
```

`STANDARDS.md` is the instruction file the AI reads first: "here is how this team works, here is what to read next, here is what matters". Everything else is linked from there. The structure mirrors how engineering teams already think about their work: languages, infrastructure, security, quality, methodology.

The same pattern works whether you fill it in with Python and Terraform or with Rust and Pulumi. The shape of the vault is what matters.

## The interview that writes itself

The interesting part is `starter-interview.md`. It's a guided conversation the AI runs when a developer first connects to the starter and says "help me set up our dev standards".

The AI walks through:

- Which languages does the team use, in which order of priority
- Which infrastructure tooling, which config management, which CI system
- Which authentication and secrets pattern
- What the PR process looks like
- What a good README looks like for a new project in this team

Every answer gets written back into the relevant standards file. By the end of the conversation the vault is filled in, committed to git, and ready to share with the team.

One session, one pull request, standards live for the whole team.

## Why this beats a wiki

A wiki is passive. Nobody opens it unless something's already broken.

Standards connected through MCP are active:

- They're read at the start of every session, by every developer, every time. The AI can't skip them the way a human can skip the onboarding PDF.
- They live in git. One pull request updates the standard for everyone. No announcements in #engineering, no "please re-read the style guide" emails, no chasing laggards.
- They're testable. Because the standards are structured markdown, you can lint them, you can check links, you can write a pre-commit hook that blocks a PR without a justification entry in `decision-log.md`.
- They onboard humans too. A new developer reads `STANDARDS.md` as a regular doc on day one. The AI reads the same file on every session after that. Same source, two audiences.

## How this differs from cursor rules or `copilot-instructions.md`

Fair question, and worth being direct about it.

Cursor rules and `copilot-instructions.md` live inside a single repo, for a single tool, for a single project. They're fine. They're not a team pattern. You'd be copying the same rules file into every repo and keeping them in sync by hand.

MCP-connected standards live in one place, outside the project repo, and apply across every project the developer touches. Switching projects doesn't reset the standards. Switching tools doesn't, either — if the next editor speaks MCP, the same standards show up. The team decides once, the developer configures once.

And because the standards are their own repo with their own PR process, the standards themselves get the same review rigour as code. Which is how it should be.

## The starter is open source

I published a fork-and-customise version on GitHub:

{:.link-accent}
**[github.com/Meppies/develop-standards-starter](https://github.com/Meppies/develop-standards-starter)**

Clone it, wire it into Claude Desktop, run the guided setup, and your team has a starting vault in one session. If you hate one of my defaults, change it. It's your vault.

## What I'd do differently next time

Two things worth flagging if you build your own.

**Start with a ruthlessly small `STANDARDS.md`.** The urge is to document everything on day one. Don't. A page of real, enforced standards is worth a hundred pages of aspirational standards nobody reads. Grow the vault when a real review comment repeats itself.

**Treat the vault as a product.** If the vault is a dumping ground, AI output quality degrades the same way human attention degrades. Shorter files, clear hierarchy, one idea per file. The same editorial discipline you'd apply to a README, applied across fifteen files.

---

Brian's DuCUG talk was about AI changing how individuals work. The same idea scales to teams: give every developer's AI the same context, and you get consistency without the overhead. An AI that writes code the way your team writes code, not the way the internet writes code.

Happy to argue about it in the comments.
