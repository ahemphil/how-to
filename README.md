---
title: Home
nav_order: 1
permalink: /
description: Tested, step-by-step guides for technical tasks, written to be followed by people and by AI assistants.
---

# How-To Guides

Step-by-step guides for technical tasks I have worked through, most of them with help from AI assistants. Each guide records:

- the exact versions it was tested on,
- the exact commands to run,
- how to check that each step worked,
- what did **not** work, so you don't repeat the same dead ends.

Every guide has a **status**. Read it before you start.

| Status | Meaning |
|---|---|
| `working` | Followed start to finish and confirmed working on the listed versions. |
| `partial` | Some steps confirmed, others not. The guide says which. |
| `outdated` | Worked once, but newer versions have broken or changed it. |

## Using an AI assistant with these guides

Give your assistant the **raw markdown** link to a guide. It is plain text with no page clutter, so any model can read it. The link is shown at the top of every guide page, and follows this pattern:

```
https://raw.githubusercontent.com/ahemphil/how-to/main/<guide-folder>/README.md
```

AI agents working in this repository should read [AGENTS.md](AGENTS.md) first.

## Guide index

<!-- Add one row per guide. Keep this list in alphabetical order by guide title. -->

| Guide | Status | Last verified |
|---|---|---|
| [Run OpenCode on local models with llama.cpp on an AMD GPU (Ubuntu)](opencode-llamacpp-local-models/) | `working` | 2026-11-07 |

## License

Guide text is licensed CC BY 4.0. Scripts and code are licensed MIT. See [LICENSE.md](LICENSE.md).
