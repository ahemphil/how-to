---
title: For AI assistants
nav_exclude: true
description: Instructions for AI assistants and agents that read, follow, or help write the guides in this repository.
---

# Instructions for AI assistants

This file is for AI assistants and agents. Humans are welcome to read it too.

## What this repository is

A collection of step-by-step technical guides. Each guide is one markdown file at `<guide-folder>/README.md`. Supporting scripts for a guide live in `<guide-folder>/scripts/`.

Repository: https://github.com/ahemphil/how-to
Rendered site: https://ahemphil.github.io/how-to/
Raw markdown of any file: `https://raw.githubusercontent.com/ahemphil/how-to/main/<path>`

## Front matter fields

Every guide starts with a YAML block. The fields mean:

- `title`: what the guide accomplishes.
- `description`: one sentence summary.
- `status`: one of `working`, `partial`, `outdated`.
  - `working`: confirmed working end to end on the versions in `tested_on`.
  - `partial`: some steps are confirmed and some are not. Steps that were not confirmed are marked **[UNTESTED]** in the text.
  - `outdated`: worked in the past; known to be broken or changed on newer versions.
- `status_note`: one sentence explaining the status.
- `last_verified`: the date (YYYY-MM-DD) the guide was last checked against real hardware/software.
- `tested_on`: the exact environment the guide was tested in.
- `ai_assisted`: `true` if an AI assistant helped work out the procedure. Steps were still run by a human unless marked **[UNTESTED]**.
- `tags`: keywords.

## If you are helping a user follow a guide

1. Compare the user's environment to `tested_on`. If versions differ, tell the user which ones differ before starting. Do not assume a step still works on a different version.
2. Respect the status. For `partial` guides, tell the user which steps are **[UNTESTED]** before they reach them.
3. Each step has a **Check:** line. Have the user run the check and confirm the expected result before moving to the next step.
4. Read the **What didn't work** section before suggesting alternatives. Do not suggest an approach listed there unless you explain what is different this time.
5. If the user gets an error, search the **Troubleshooting** section for the exact error text first.
6. Ask before running any step marked destructive, or any step that deletes, overwrites, or reformats something.
7. If the guide is wrong or out of date, the user can report it at: https://github.com/ahemphil/how-to/issues

## If you are helping write or update a guide

Copy `_template.md` into a new folder named with lowercase words and hyphens (for example `msfs2020-neofly-linux/README.md`). Then follow these rules.

1. **One action per step.** Number the steps. Start each step heading with a verb.
2. **Exact commands in code blocks.** No paraphrased commands. Mark placeholders as `<LIKE_THIS>` and say right below the block what to substitute.
3. **Every step gets a Check line**: a command or observation, plus the exact expected result.
4. **Sections must make sense on their own.** Readers, and AI search tools, often see one section without the rest of the page. Do not write "as above", "the same command", or "that file". Repeat the path or command instead.
5. **Exact versions, not "latest".** Put the environment in `tested_on`. Name versions in steps when they matter.
6. **Mark anything not run on real hardware as [UNTESTED].** Never present a plausible step as a confirmed one.
7. **Record dead ends** in "What didn't work", with the reason each one failed.
8. **Troubleshooting headings are the literal error text**, in backticks, so people can find them by pasting the error into a search engine.
9. **No information only in images.** Screenshots are optional extras. Any instruction must also exist as text.
10. **Remove personal details** from pasted logs and paths: usernames in home directories, hostnames, IP addresses (LAN and VPN), serial numbers, email addresses, and tokens. Use `<USER>`, `<HOSTNAME>`, and similar placeholders.
11. When the guide changes, update `last_verified` (only if re-tested), `status`, and the **Changelog** section, and update the row in the root `README.md` index.
