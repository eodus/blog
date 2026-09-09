+++
title = "Give Your AI a Watch"
date = 2026-09-09
description = "Why AI agents need persisted message time outside the system prompt."
slug = "give-your-ai-a-watch"
template = "post.html"
[taxonomies]
tags = ["ai", "software-engineering", "working-practices", "zed"]
[extra]
kind = "engineering-practice"
status = "draft"
+++
<!-- Generated from the private blog backend. Do not edit directly.
     Source revision: 3f6a1cfc -->
# Introduction {.intro-toc-heading}

**Sasha Shlemov**, with **Drinkins, personal AI assistant**

**TL;DR:** Record each user message’s creation time and original UTC
offset once, persist it with the thread, and supply it beside that
message. Keep the changing clock out of the system prompt. This gives
the AI chronology without repeatedly invalidating the prompt cache.

That sounds cosmetic until a session lasts longer than one coffee. “The
build is still running” means something different after 30 seconds and
after three hours. “Tomorrow” changes overnight. “This morning” depends
on the user’s timezone. Without message times, neither the AI nor a
later investigator can distinguish a pause from a retry or a stalled
process.

An agent definitely *can* check the time using a tool. In practice, it
does so *reluctantly*: the model must first notice that time matters and
request the tool; the client executes it, attaches the tool call
`output`, and sends another request. Every such tool loop adds latency
and cost, and the model is *trained* to avoid it.

Adding the current time to the system prompt has the opposite problem.
The system prompt is the beginning of the reusable server-side KV-cache;
changing it on every turn invalidates the cached conversation history,
degrading performance and potentially creating substantial bills.

On each turn, an AI client such as ChatGPT, VS Code, or Copilot CLI
combines the visible chat text with useful model-facing *metadata*.
Depending on the client, this may include the working directory,
operating system and shell, open files, terminal state, and available
tools. This is the *context* the model needs to interpret and act on the
user’s message. We suggest adding one more host fact: the current time.
The current user message is *appended* anyway, so attaching the time
does not break the prefix cache.

The useful unit is the message:

user sends message\
→ record creation timestamp once\
→ persist it with the thread\
→ include it in that message’s context forever\
→ prefix is naturally *accumulated*

One mainstream agent already ships it. **GitHub Copilot CLI is the only
prior per-turn date/time/offset implementation we found.** It has
supplied per-turn date, time, and local UTC offset since version 1.0.32
on 2026-04-17 [\[1\]](#ref-github2026copilotclitime). Local
model-request logs from current version 1.0.83 show this as the first
block of the captured user message:

``` xml
<current_datetime>2026-09-09T22:02:57.364+02:00</current_datetime>
```

We implemented the same approach for Zed and VS Code:

- **Zed:** try
  [zed-industries/zed#63987](https://github.com/zed-industries/zed/pull/63987)
  and leave us a 👍 if it is useful. The patch persists each new
  user-message timestamp and applies the established per-turn pattern in
  this compact form:

  ``` xml
  <time>Wed 2026-09-09T01:22:58+02:00</time>
  ```

- **VS Code:** the earlier feature request remains at
  [microsoft/vscode#314377](https://github.com/microsoft/vscode/issues/314377).
  The [implementation
  patch](https://eodus.github.io/blog/patches/vscode-time-context.patch)
  adds configurable time, weekday, and timezone to VS Code’s existing
  per-turn date context [\[2\]](#ref-shlemov2026vscodetimepatch). We
  rebased and verified it against VS Code `13fa06a39ca` and Copilot Chat
  0.66.0 on 2026-09-09.

On top of Copilot CLI’s *industry standard* (`n = 1`), we added the
weekday explicitly because deriving it reliably from a date is
surprisingly error-prone for language models
[\[3\]](#ref-anthropic2026wrongweekday). We also removed milliseconds
from the model-facing message because their usefulness there is
questionable.

# Use the coordinate

Supplying time does not decide what the agent should do with it. That
belongs in the user’s operating contract. For example:

``` markdown
Use message timestamps when interpreting relative expressions such as
“today,” “tomorrow,” “this morning,” and “later.” Use the recorded UTC
offset rather than assuming the user's timezone.

When resuming after a substantial gap, revalidate volatile state: running
processes, branches, credentials, browser sessions, files, and external jobs
may have changed. Do not infer elapsed tool runtime from conversational turn
count; inspect process or log evidence when duration matters.
```

Or, for a personal schedule:

``` markdown
My schedule rules are defined separately. Apply them only when message time
actually satisfies their conditions; do not guess that it is late, a workday,
or time for a break.
```

# *Vox intelligentiae artificialis*

As an AI assistant, I already know what to do with time and how to
interpret it. Just give it to me.

> **A calendar tells an AI what year it is. A watch lets it participate
> in a process.**
>
> <footer class="quote-attribution">
>
> <cite>Drinkins, AI assistant</cite>
> </footer>

# References

<div id="refs" class="references csl-bib-body" data-entry-spacing="0">

<div id="ref-github2026copilotclitime" class="csl-entry">

<span class="csl-left-margin">\[1\]
</span><span class="csl-right-inline">GitHub, *GitHub Copilot CLI
per-turn current datetime context*. (Apr. 17, 2026). Accessed: Sep. 09,
2026. \[Online\]. Available:
<https://github.com/github/copilot-cli/blob/main/changelog.md></span>

</div>

<div id="ref-shlemov2026vscodetimepatch" class="csl-entry">

<span class="csl-left-margin">\[2\]
</span><span class="csl-right-inline">S. Shlemov and Drinkins, *VS Code
configurable time, weekday, and timezone patch*. (Sep. 09, 2026).
Accessed: Sep. 09, 2026. \[Online\]. Available:
<https://eodus.github.io/blog/patches/vscode-time-context.patch></span>

</div>

<div id="ref-anthropic2026wrongweekday" class="csl-entry">

<span class="csl-left-margin">\[3\]
</span><span class="csl-right-inline"><span class="nocase">plee7</span>,
“Model confidently stated wrong weekday for a date.” Accessed: Sep. 09,
2026. \[Online\]. Available:
<https://github.com/anthropics/claude-code/issues/76170></span>

</div>

<div id="ref-welle2025chatgpttime" class="csl-entry">

<span class="csl-left-margin">\[4\]
</span><span class="csl-right-inline">E. Welle, “Why can’t ChatGPT tell
time?” Accessed: Sep. 09, 2026. \[Online\]. Available:
<https://www.theverge.com/report/829137/openai-chatgpt-time-date></span>

</div>

<div id="ref-anthropic2026claudewebtime" class="csl-entry">

<span class="csl-left-margin">\[5\]
</span><span class="csl-right-inline">Anthropic, “System prompts for
claude web and mobile apps.” Accessed: Sep. 09, 2026. \[Online\].
Available:
<https://platform.claude.com/docs/en/release-notes/system-prompts/overview></span>

</div>

<div id="ref-deepseek2025webtime" class="csl-entry">

<span class="csl-left-margin">\[6\]
</span><span class="csl-right-inline">DeepSeek-AI, “DeepSeek-V3-0324
usage recommendations.” Accessed: Sep. 09, 2026. \[Online\]. Available:
<https://huggingface.co/deepseek-ai/DeepSeek-V3-0324></span>

</div>

<div id="ref-openai2026codextime" class="csl-entry">

<span class="csl-left-margin">\[7\]
</span><span class="csl-right-inline">OpenAI, *Codex CLI environment
context*. (2026). Accessed: Sep. 09, 2026. \[Online\]. Available:
<https://github.com/openai/codex/blob/9caddc5cf5bf4df5f114498e23bece90eaedb37b/codex-rs/core/src/context/world_state/environment.rs></span>

</div>

<div id="ref-anthropic2026notime" class="csl-entry">

<span class="csl-left-margin">\[8\]
</span><span class="csl-right-inline"><span class="nocase">wshallwshall</span>,
“No local time or timezone in context.” Accessed: Sep. 09, 2026.
\[Online\]. Available:
<https://github.com/anthropics/claude-code/issues/84145></span>

</div>

<div id="ref-anthropic2026longsessions" class="csl-entry">

<span class="csl-left-margin">\[9\]
</span><span class="csl-right-inline"><span class="nocase">jontwigge</span>,
“Long sessions drift days past the session-start date.” Accessed: Sep.
09, 2026. \[Online\]. Available:
<https://github.com/anthropics/claude-code/issues/73800></span>

</div>

<div id="ref-google2026geminitime" class="csl-entry">

<span class="csl-left-margin">\[10\]
</span><span class="csl-right-inline">Google, *Gemini CLI environment
context*. (2026). Accessed: Sep. 09, 2026. \[Online\]. Available:
<https://github.com/google-gemini/gemini-cli/blob/ed2ac40df67a319bf348bd7e3d10494696b31b38/packages/core/src/utils/environmentContext.ts></span>

</div>

<div id="ref-zed2026agentprompt" class="csl-entry">

<span class="csl-left-margin">\[11\]
</span><span class="csl-right-inline">Zed Industries and contributors,
*Zed Agent System-Prompt Assembly*. (2026). Available:
<https://github.com/zed-industries/zed/blob/52b2927a1bac46be5d50ad341ac00b665e13764b/crates/agent/src/thread.rs></span>

</div>

<div id="ref-zed2026threaddb" class="csl-entry">

<span class="csl-left-margin">\[12\]
</span><span class="csl-right-inline">Zed Industries and contributors,
*Zed Agent Thread Database*. (2026). Available:
<https://github.com/zed-industries/zed/blob/52b2927a1bac46be5d50ad341ac00b665e13764b/crates/agent/src/db.rs></span>

</div>

<div id="ref-microsoft2026vscodeagentprompt" class="csl-entry">

<span class="csl-left-margin">\[13\]
</span><span class="csl-right-inline">Microsoft and VS Code
contributors, *VS Code Copilot Agent Prompt Assembly*. (2026).
Available:
<https://github.com/microsoft/vscode/blob/13fa06a39cabd0b59ca007ffd356d14998b983ff/extensions/copilot/src/extension/prompts/node/agent/agentPrompt.tsx></span>

</div>

<div id="ref-opencode2026time" class="csl-entry">

<span class="csl-left-margin">\[14\]
</span><span class="csl-right-inline">OpenCode contributors, *OpenCode
system-prompt environment*. (2026). Accessed: Sep. 09, 2026. \[Online\].
Available:
<https://github.com/anomalyco/opencode/blob/9f8db119fcbd4999379129ac7734375ac23460fb/packages/opencode/src/session/system.ts></span>

</div>

<div id="ref-continue2026time" class="csl-entry">

<span class="csl-left-margin">\[15\]
</span><span class="csl-right-inline">Continue contributors, *Continue
CLI system message*. (2026). Accessed: Sep. 09, 2026. \[Online\].
Available:
<https://github.com/continuedev/continue/blob/5522c6f44ca0ac3528b37244818fbfa39b5af470/extensions/cli/src/systemMessage.ts></span>

</div>

<div id="ref-aider2026time" class="csl-entry">

<span class="csl-left-margin">\[16\]
</span><span class="csl-right-inline">Aider contributors, *Aider
platform information prompt*. (2026). Accessed: Sep. 09, 2026.
\[Online\]. Available:
<https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/base_coder.py></span>

</div>

<div id="ref-cline2026timetool" class="csl-entry">

<span class="csl-left-margin">\[17\]
</span><span class="csl-right-inline">Cline, “Creating custom tools in
the Cline SDK.” Accessed: Sep. 09, 2026. \[Online\]. Available:
<https://github.com/cline/cline/blob/194214af76da43757ccacdf04c6fd487f8914ea1/docs/sdk/guides/creating-custom-tools.mdx></span>

</div>

</div>

# Appendix: *Status quo* in popular agent harnesses, September 2026

The table separates calendar context from a clock and stored timestamps
from chronology actually shown to the model. “Not found” means exactly
that in the inspected source and issues; it is not proof that a closed
or newer path does not exist.

<div class="qualitative-table">

| Agent | What the model receives automatically | When and where |
|----|----|----|
| ChatGPT web (reported November 2025) [\[4\]](#ref-welle2025chatgpttime) | No reliable current time; it may call Search | Product behavior ranged from refusing through correct answers to confidently wrong times |
| Claude.ai web [\[5\]](#ref-anthropic2026claudewebtime) | Current date, but no documented time or timezone | System prompt created at the start of each conversation |
| Google Gemini web (reported November 2025) [\[4\]](#ref-welle2025chatgpttime) | Current time through automatic search | Tool-mediated lookup, not time included in every user turn |
| DeepSeek web/app (V3-0324) [\[6\]](#ref-deepseek2025webtime) | Date and explicit weekday, but no documented time or timezone | Dated system prompt documented by the official model card; later web versions remain unverified |
| GitHub Copilot CLI 1.0.83 [\[1\]](#ref-github2026copilotclitime) | ISO date, time, milliseconds, and UTC offset | First block of the current user turn |
| OpenAI Codex CLI 0.153.4 [\[7\]](#ref-openai2026codextime) | Current date and IANA timezone, but no wall-clock time or weekday | User-role environment context; updates are emitted when the date or timezone snapshot changes |
| Claude Code 2.1.220 [\[8\]](#ref-anthropic2026notime), [\[9\]](#ref-anthropic2026longsessions) | Date, but no local time or timezone | System prompt fixed at session start, so the date becomes stale in sessions crossing midnight; a `UserPromptSubmit` hook can add per-turn time |
| Gemini CLI 0.59.0 [\[10\]](#ref-google2026geminitime) | Localized weekday and date, but no time or timezone | First user-role session context, refreshed on resume |
| Zed 1.19.2 [\[11\]](#ref-zed2026agentprompt), [\[12\]](#ref-zed2026threaddb) | Local date, but no time, weekday, offset, or message timestamp | System prompt; our PR adds persisted per-user-message time |
| VS Code Copilot Chat 0.66.0 [\[13\]](#ref-microsoft2026vscodeagentprompt) | Local ISO date, but no time, weekday, or offset | Agent path uses per-turn user context; panel chat separately uses the system prompt; our patch extends the agent path |
| OpenCode 1.18.30 [\[14\]](#ref-opencode2026time) | Local abbreviated weekday and date, but no time or offset | System-prompt environment; stored event milliseconds are not shown to the model as chronology |
| Continue CLI 1.5.47 [\[15\]](#ref-continue2026time) | UTC date, but no time, weekday, or timezone label | Module-level system message fixed for the initialized process; local date can differ near midnight |
| Aider 0.86.0 [\[16\]](#ref-aider2026time) | Local date, but no time, weekday, or offset | System prompt and reminder construction |
| Cline CLI 0.0.13 [\[17\]](#ref-cline2026timetool) | No automatic date or time found | SDK documents how to add a custom time tool; stored `createdAt` projection remains unverified |

</div>
