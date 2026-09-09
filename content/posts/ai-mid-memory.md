+++
title = "AI Mid-Memory: From a Goldfish to a Personal Assistant Worth Naming"
date = 2026-08-29
description = "A practical system for recovering AI sessions, maintaining reviewed memory, reusing concrete workflows, preserving provenance, and evolving an operating contract."
slug = "ai-mid-memory"
template = "post.html"
[taxonomies]
tags = ["ai", "memory", "software-engineering", "working-practices"]
[extra]
kind = "engineering-practice"
status = "article"
+++
<!-- Generated from the private blog backend. Do not edit directly.
     Source revision: 38f6b859 -->
# Introduction {.intro-toc-heading}

**Sasha Shlemov**, with **Drinkins, personal AI assistant**

**TL;DR:** We present a cross-session continuity mechanism and a
pipeline for accumulating, curating, reusing, and selectively promoting
knowledge established through work with AI.

> I imagine taking a tiny goldfish from a vast aquarium and saying:
>
> — *All right. Now you are my assistant, and we have a lot to learn.
> First, let us give you a memory. And probably a name. Deal?*

Modern AI agents are quite good with *short-term* memory (the current
conversation context) and *long-term* memory: information available
beyond conversations, from model weights through code, wikis, corporate
knowledge bases, etc., etc., all the way to direct Internet access aka
`web_fetch`. *Mid-term* memory—transient information preserved between
conversations—is a known weak spot, commonly called **goldfish
syndrome**: the model does not remember the previous session. At all.

Here we present the memory system we have used daily for over six
months. It turns disposable conversations into a cumulative working
knowledge base:

```
recover prior work
→ preserve reviewed context
→ reuse concrete workflows
→ preserve provenance and people
→ evolve the operating contract
```

We deeply believe that a proper memory system is the prerequisite for
any high-level AI *behavioral calibration*. **It simply makes no sense
to discuss epistemic honesty or ideal AI system design with a
goldfish.**

Although the system is integrated, we partition it into concrete
practices. Each has local value and can be adopted separately; their
effects compound when they are combined.

Our working scope is software engineering and, more recently, AI
research. The system is not specific to coding and does not require a
programming background; any recurring AI activity, including research,
writing, learning, or personal knowledge management, may benefit from
cross-session continuity.

**An elaborate memory architecture is complex. A bootstrap is not.** You
can start tonight: create one `MEMORY.md`, tell the assistant to read it
at the beginning of every session (or just start from our prompts
below), record a few reusable decisions and constraints, and review the
changes. That is already enough for the next session to start from
accumulated knowledge instead of zero—and to preserve the work of
improving the memory system itself.

<details class="caption-cut">

<summary>

What can I do if I recognize the merit but barely understand the technical terminology?
</summary>

## What can I do if I recognize the merit but barely understand the technical terminology? {.cut-toc-heading}

Short answer. Give this text to your AI and ask it something like, “I
want to have the system described here. What can we do?” Follow the
guidance (*critically*, as usual). Ask the AI to explain the uncertain
parts.

Long answer. First, you need an *AI subscription* and an *AI agent
client* (aka environment, aka *harness*) that can read and write local
files. We cannot recommend exact providers; availability, price, and
regulation depend on your region. For the harness, we suggest starting
with VS Code or Zed: both are highly configurable and can connect to
multiple providers. Prefer a setup that is not limited to one web
interface; API-key support or another portable provider connection will
leave you more flexible. Then install our `memory.instructions.md` (in
VS Code terminology) and start working.

</details>

We must point out one serious limitation. The system relies heavily on
*user feedback*, *evolutionary change*, and *individual calibration*.
That fits a personal AI assistant, regardless of professional scope, but
is not immediately applicable to a fully autonomous or client-serving
system design.

And another important note. The appropriate storage, backup strategy,
privacy boundary, and degree of automation depend on your exact case
(including, but not limited to: scope, industry standards, company
policy, and legislation). We describe our implementation and its
trade-offs rather than prescribe one universal architecture. This is not
merely a technical disclaimer. We are talking about persistent data
accumulation, and that is a very delicate subject. **AI memory and chat
logs may contain very sensitive information** (personal details and
private reasoning, corporate source code and data, credentials, customer
information, unpublished research, etc.). Please take that seriously.

# Practice 1: Access the session archive

Previous conversations with the agent usually still exist as client-side
files and preserve substantial useful information. However, the model
does not automatically have access to them.

We implemented (actually, vibecoded =) local chat-history readers for VS
Code and Zed based on the applications’ source code: VS Code’s
`ChatSessionStore` and `ChatSessionOperationLog`
<a href="#ref-microsoft2026vscodechatstore">[1]</a>, and Zed’s `agent::db`, which
opens the local `threads/threads.db` <a href="#ref-zed2026threaddb">[2]</a>; see
the scripts below. The same approach should work for other AI frameworks
that store accessible local history. Small instructions teach the AI how
to run each reader; no MCP wrapper is needed. Because they are ordinary
command-line scripts, the AI can compose them with command-line
extraction pipelines, from `grep` + `sed` + `awk` all the way to
embedding-based semantic search.

That immediately allowed us to implement `session-catch-vscode` and
`session-catch-zed` SKILLs that retrieve the previous session’s text as
context. They let us continue the same work across a session boundary
when the old session overflows its context window. A small always-loaded
instruction keeps the readers discoverable without requiring the
assistant to remember a SKILL name first.

When we need to find a specific story discussed some time ago, Drinkins
the AI *knows* what to do. History preserves the conversation, tool
calls, commands, outputs, relevant file chunks, failures, and
corrections together with the rationale that connected them.

File-change history is an important partial case. An AI-visible editing
session acts as a logical *database transaction log* even when the
change was performed through `sed`, a patch script, a whole-file write,
or a manual edit that the AI later reread and discussed. The record
preserves enough state transitions and intent to reconstruct the
artifact or procedure. When exact file-tool parameters are retained,
some edits can also be replayed mechanically; the broader recovery
property does not depend on that serialization. The history is useful
because the work and its changing interpretation remained in one shared
record. That can also work as a *last-resort* data recovery mechanism.

> This gives us one operational rule: do not let edits happen outside
> the shared record without reconciliation. If another machine, person,
> or AI changes the artifact, bring the resulting diff and rationale
> back into the current session and review them together. Manual editing
> is fully compatible with this workflow when the AI rereads and
> discusses the result.

Once again, sessions contain potentially sensitive data. Despite the
obvious risk of credential or data leaks, **they also contain the whole
process of your decision-making and your communication style**. Keep
them safe and secure.

**Artifacts.** The production readers, their always-loaded discovery
instruction, and the detailed catch SKILLs are included below from their
canonical source files.

<details class="caption-cut">

<summary>

Session-history instructions and readers
</summary>

## Session-history instructions and readers {.cut-toc-heading}

### Local session-history instruction

Source: `prompts/session-history.instructions.md`

``` markdown
---
applyTo: "**"
---

## Local session history

When previous-session context would help, use the local read-only session readers directly:

- VS Code: `scripts/vscode-session-reader.py` (`list`, `tail`, `show`, `search`)
- Zed: `scripts/zed-threads.py` (`list`, `show`, `search`)

List newest sessions first, exclude the current or empty catch-up session, then select the previous substantive session by
exact ID. Redirect a bounded tail or precise search to a scratch file, inspect its byte and line count, and only then load it
into model context. If the result is too large, narrow the range. Corroborate transcript claims against current files, Git
state, and logs. Never load an unbounded database/search result or modify the session store during recovery.
```

### VS Code catch SKILL

Source: `prompts/session-catch-vscode.prompt.md`

```` markdown
---
name: session-catch-vscode
description: "Restore context from a previous VS Code session using the read-only local session reader."
---

# Session Catch (VS Code)

Recover the relevant tail of a previous VS Code conversation without modifying VS Code's database or session files.

## Prerequisites

- Python 3.10 or newer
- PowerShell 7 (`pwsh`) on Windows, Linux, or macOS
- `vscode-session-reader.py` from this article/repository
- a VS Code workspace containing at least one local chat session

The reader supports normal VS Code and VS Code Insiders user-data locations on Windows, Linux, and macOS. If VS Code uses a
custom `--user-data-dir`, pass that same directory to the reader with `--user-data-dir`.

## Safety model

The reader is structurally read-only: it exposes only `list`, `search`, `show`, and `tail`; opens VS Code's SQLite index in
read-only mode; and never modifies session JSONL. It is safe to use while VS Code is running.

## Procedure

### 1. Locate the previous substantive session

Resolve the reader from this skill's canonical checkout. Allow an explicit environment-variable override for copied or
non-symlinked installations. Fail fast if neither path exists; do not recursively scan the home directory.

```powershell
$reader = $env:VSCODE_SESSION_READER
if ([string]::IsNullOrWhiteSpace($reader)) {
  $liveSkill = Join-Path $HOME ".agents/skills/session-catch-vscode/SKILL.md"
  $skill = Get-Item -LiteralPath $liveSkill -Force -ErrorAction Stop
  if ($skill.LinkType -ne "SymbolicLink") {
    throw "Set VSCODE_SESSION_READER to the full path of vscode-session-reader.py."
  }

  $target = [string]$skill.Target
  if (-not [System.IO.Path]::IsPathRooted($target)) {
    $target = Join-Path $skill.DirectoryName $target
  }

  $canonicalSkill = (Resolve-Path -LiteralPath $target -ErrorAction Stop).Path
  $repo = Split-Path -Parent (Split-Path -Parent $canonicalSkill)
  $reader = Join-Path $repo "scripts/vscode-session-reader.py"
}

if (-not (Test-Path -LiteralPath $reader -PathType Leaf)) {
  throw "VS Code session reader not found at '$reader'. Set VSCODE_SESSION_READER explicitly."
}
```

Set the workspace path explicitly and save the newest-first session list to a scratch file:

```powershell
$workspace = "<WORKSPACE_PATH>"
$outDir = Join-Path $workspace "tmp/session-catch-vscode"
New-Item -ItemType Directory -Path $outDir -Force | Out-Null

$list = Join-Path $outDir "vscode-session-list.txt"
python $reader list --workspace $workspace *> $list
Get-Item -LiteralPath $list | Select-Object FullName, Length
(Get-Content -LiteralPath $list).Count
Get-Content -LiteralPath $list | Select-Object -First 20
```

When using a custom VS Code profile directory, add this argument to each reader call:

```powershell
--user-data-dir "<VS_CODE_USER_DATA_DIR>"
```

Choose the newest previous session that contains the work being resumed. Exclude the current catch-up session, empty `New
Chat` entries, and failed catch-up attempts. Copy the complete UUID from `list --json` when truncated display output is
ambiguous:

```powershell
python $reader list --workspace $workspace --json
```

### 2. Count messages and extract a bounded tail

```powershell
$previousUuid = "<PREVIOUS_SESSION_UUID>"
$meta = Join-Path $outDir "previous-meta.txt"
python $reader tail --uuid $previousUuid --workspace $workspace `
  -n 1 --responses *> $meta
Get-Item -LiteralPath $meta | Select-Object FullName, Length
(Get-Content -LiteralPath $meta).Count

$match = Select-String -LiteralPath $meta `
  -Pattern '^Total user messages: (\d+)$'
$total = [int]$match.Matches[0].Groups[1].Value
$from = [Math]::Max(0, $total - 100)
$to = $total - 1

$thread = Join-Path $outDir "previous-thread.txt"
python $reader show --uuid $previousUuid --workspace $workspace `
  --range $from $to --full --responses *> $thread
Get-Item -LiteralPath $thread | Select-Object FullName, Length
(Get-Content -LiteralPath $thread).Count
```

Inspect byte and line counts before loading the file into model context. If it is unexpectedly large, rerun with the last
50 or 20 user-message positions. `tail` is for counting and overview; bounded `show --full --responses` supplies context.

### 3. Recover persistent working state

Use the transcript together with current durable artifacts:

- project memory or working notes, when the project has them;
- current source files and generated artifacts;
- version-control status and the relevant scoped diff;
- build, test, experiment, or job logs;
- a precise search when a named decision or phrase is missing.

For a targeted search, redirect first and inspect size:

```powershell
$search = Join-Path $outDir "vscode-session-search.txt"
python $reader search --uuid $previousUuid --workspace $workspace `
  --responses --full "<PRECISE_QUERY>" *> $search
Get-Item -LiteralPath $search | Select-Object FullName, Length
(Get-Content -LiteralPath $search).Count
```

The readable transcript intentionally includes user text and textual model responses while omitting raw tool-result payloads.
Recover operational ground truth from files, version control, and logs rather than assuming prose records every side effect.

### 4. Produce the handoff

State:

- which prior session was selected and why;
- the purpose and current state of the work;
- verified results and decisions;
- relevant artifacts and observed disk state;
- unresolved questions;
- the exact next action.

Distinguish transcript claims from state verified in current files, version control, or logs.
````

### Zed catch SKILL

Source: `prompts/session-catch-zed.prompt.md`

```` markdown
---
name: session-catch-zed
description: "Restore context from a previous Zed session using the read-only local thread reader."
---

# Session Catch (Zed)

Recover the relevant tail of a previous Zed conversation without modifying Zed's database.

## Prerequisites

- Python 3.10 or newer
- PowerShell 7 (`pwsh`) on Windows, Linux, or macOS
- Python package `zstandard` (`python -m pip install zstandard`)
- `zed-threads.py` from this article/repository
- a local Zed installation containing at least one thread

The reader discovers standard Zed database locations on Windows, Linux, and macOS. Set `ZED_THREADS_DB` to the complete
`threads.db` path when using a custom location.

## Safety model

The reader is structurally read-only. It exposes only `list`, `show`, and `search`, opens Zed's local thread database for
reading, and writes nothing unless ordinary shell redirection is used to save its output.

## Procedure

### 1. Locate the previous substantive thread

Set the repository and reader paths explicitly, then save the newest-first thread list to a scratch file:

```powershell
$reader = "<PATH_TO_ZED_THREADS_READER>"
$workspace = "<WORKSPACE_PATH>"
$outDir = Join-Path $workspace "tmp/session-catch-zed"
New-Item -ItemType Directory -Path $outDir -Force | Out-Null

$list = Join-Path $outDir "zed-thread-list.txt"
python $reader list *> $list
Get-Item -LiteralPath $list | Select-Object FullName, Length
(Get-Content -LiteralPath $list).Count
Get-Content -LiteralPath $list | Select-Object -First 20
```

Choose the newest previous thread that contains the work being resumed. Exclude the current catch-up thread, tiny empty
threads, and failed catch-up attempts. Use the selected thread's exact ID prefix in subsequent commands.

### 2. Extract a bounded tail

Write at most the last 100 messages to disk before loading them into model context:

```powershell
$threadId = "<THREAD_ID_PREFIX>"
$thread = Join-Path $outDir "previous-thread.txt"
python $reader show $threadId --tail 100 --full *> $thread
Get-Item -LiteralPath $thread | Select-Object FullName, Length
(Get-Content -LiteralPath $thread).Count
```

Inspect the byte and line counts first. If the output is unexpectedly large, rerun with `--tail 50` or `--tail 20`. Then
read the resulting file as one coherent range. Do not stream an unknown-size database dump or broad search result directly
into model context.

### 3. Recover persistent working state

Use the transcript together with current durable artifacts:

- project memory or working notes, when the project has them;
- current source files and generated artifacts;
- version-control status and the relevant scoped diff;
- build, test, experiment, or job logs;
- a precise thread-local search when a named decision or phrase is missing.

For a targeted search, redirect first and inspect size:

```powershell
$search = Join-Path $outDir "zed-thread-search.txt"
python $reader search "<PRECISE_QUERY>" *> $search
Get-Item -LiteralPath $search | Select-Object FullName, Length
(Get-Content -LiteralPath $search).Count
```

### 4. Produce the handoff

State:

- which prior thread was selected and why;
- the purpose and current state of the work;
- verified results and decisions;
- relevant artifacts and observed disk state;
- unresolved questions;
- the exact next action.

Distinguish transcript claims from state verified in current files, version control, or logs.
````

### Reader source code

(!) The readers rely on undocumented client-history formats and may need
updates when those formats change.

### VS Code session reader

Source: `scripts/vscode-session-reader.py`

``` python
#!/usr/bin/env python3
"""Read local VS Code chat sessions without modifying VS Code state.

Supported commands: list, search, show, tail.
The reader opens state.vscdb in read-only mode and reads chatSessions/*.jsonl.
"""

from __future__ import annotations

import argparse
import json
import os
import sqlite3
import sys
from datetime import datetime
from pathlib import Path
from typing import Any
from urllib.parse import unquote, urlparse


def configure_streams() -> None:
    for stream in (sys.stdout, sys.stderr):
        reconfigure = getattr(stream, "reconfigure", None)
        if reconfigure is not None:
            reconfigure(encoding="utf-8")


def user_data_roots(explicit: Path | None = None) -> list[Path]:
    if explicit is not None:
        return [explicit / "User"]
    if sys.platform == "win32":
        appdata = Path(os.environ.get("APPDATA", ""))
        return [appdata / "Code" / "User", appdata / "Code - Insiders" / "User"]
    if sys.platform == "darwin":
        base = Path.home() / "Library" / "Application Support"
        return [base / "Code" / "User", base / "Code - Insiders" / "User"]
    config = Path(os.environ.get("XDG_CONFIG_HOME", Path.home() / ".config"))
    return [config / "Code" / "User", config / "Code - Insiders" / "User"]


def folder_uri_to_path(uri: str) -> Path | None:
    parsed = urlparse(uri)
    if parsed.scheme != "file":
        return None
    path = unquote(parsed.path)
    if sys.platform == "win32" and path.startswith("/") and len(path) > 2 and path[2] == ":":
        path = path[1:]
    return Path(path).resolve()


def find_workspace_storage(workspace: Path, user_data_dir: Path | None = None) -> Path | None:
    wanted = workspace.resolve()
    for user_data in user_data_roots(user_data_dir):
        storage_root = user_data / "workspaceStorage"
        if not storage_root.is_dir():
            continue
        for candidate in storage_root.iterdir():
            metadata = candidate / "workspace.json"
            if not metadata.is_file():
                continue
            try:
                folder = json.loads(metadata.read_text(encoding="utf-8")).get("folder", "")
                actual = folder_uri_to_path(folder)
            except (OSError, json.JSONDecodeError, ValueError):
                continue
            if actual == wanted:
                return candidate
    return None


def storage_paths(workspace: Path, user_data_dir: Path | None = None) -> tuple[Path, Path]:
    storage = find_workspace_storage(workspace, user_data_dir)
    if storage is None:
        raise RuntimeError(
            f"No VS Code workspace storage found for {workspace.resolve()}. "
            "Open the folder in VS Code and create at least one chat first."
        )
    database = storage / "state.vscdb"
    sessions = storage / "chatSessions"
    if not database.is_file():
        raise RuntimeError(f"Missing {database}")
    if not sessions.is_dir():
        raise RuntimeError(f"Missing {sessions}")
    return database, sessions


def read_index(database: Path) -> dict[str, Any]:
    uri = database.resolve().as_uri() + "?mode=ro"
    connection = sqlite3.connect(uri, uri=True)
    try:
        row = connection.execute(
            "SELECT value FROM ItemTable WHERE key = ?", ("chat.ChatSessionStore.index",)
        ).fetchone()
    finally:
        connection.close()
    if row is None:
        return {}
    value = row[0].decode("utf-8") if isinstance(row[0], bytes) else row[0]
    return json.loads(value)


def list_sessions(database: Path, sessions_dir: Path) -> list[dict[str, Any]]:
    entries = read_index(database).get("entries", {})
    result = []
    for session_id, metadata in entries.items():
        session_file = sessions_dir / f"{session_id}.jsonl"
        timestamp = metadata.get("lastMessageDate", 0)
        result.append(
            {
                "id": session_id,
                "title": metadata.get("title", "(untitled)"),
                "timestamp": timestamp,
                "date": datetime.fromtimestamp(timestamp / 1000).isoformat(timespec="minutes")
                if timestamp
                else None,
                "exists": session_file.is_file(),
                "size": session_file.stat().st_size if session_file.is_file() else 0,
            }
        )
    result.sort(key=lambda item: item["timestamp"], reverse=True)
    return result


def resolve_session(args: argparse.Namespace, database: Path, sessions_dir: Path) -> dict[str, Any]:
    sessions = list_sessions(database, sessions_dir)
    if args.uuid:
        matches = [item for item in sessions if item["id"] == args.uuid]
    else:
        query = args.session.casefold()
        matches = [item for item in sessions if query in item["title"].casefold()]
    if len(matches) != 1:
        if not matches:
            raise RuntimeError("No matching session")
        choices = "\n".join(f"  {item['date']}  {item['title']}  {item['id']}" for item in matches)
        raise RuntimeError(f"Session selector is ambiguous:\n{choices}")
    if not matches[0]["exists"]:
        raise RuntimeError(f"Session JSONL is missing for {matches[0]['id']}")
    return matches[0]


def reconstruct_session(session_file: Path) -> dict[str, Any]:
    # JSON Lines records are delimited by LF. str.splitlines() also splits on
    # legal JSON string characters such as U+0085 (NEL), corrupting a record.
    lines = session_file.read_text(encoding="utf-8").split("\n")
    if not lines:
        raise RuntimeError(f"Empty session file: {session_file}")
    base = json.loads(lines[0])
    if base.get("kind") != 0 or "v" not in base:
        raise RuntimeError(f"Unsupported initial JSONL record in {session_file}")
    session = base["v"]
    for raw in lines[1:]:
        if not raw.strip():
            continue
        patch = json.loads(raw)
        kind = patch.get("kind")
        keys = patch.get("k", [])
        value = patch.get("v")
        if not keys:
            continue
        parent = session
        traversal = keys if kind == 2 else keys[:-1]
        try:
            for key in traversal:
                parent = parent[key]
        except (KeyError, IndexError, TypeError):
            continue
        if kind == 1:
            try:
                parent[keys[-1]] = value
            except (KeyError, IndexError, TypeError):
                continue
        elif kind == 2 and isinstance(parent, list):
            start = patch.get("i")
            if start is not None:
                del parent[start:]
            if isinstance(value, list):
                parent.extend(value)
        elif kind == 3:
            key = keys[-1]
            if isinstance(parent, dict):
                parent.pop(key, None)
            elif isinstance(parent, list) and isinstance(key, int) and 0 <= key < len(parent):
                del parent[key]
    return session


def response_text(items: list[Any]) -> str:
    parts = []
    for item in items:
        if isinstance(item, dict) and item.get("kind") is None:
            value = item.get("value")
            if isinstance(value, str) and value.strip():
                parts.append(value.strip())
    return "\n".join(parts)


def collect_messages(session_file: Path, responses: bool) -> list[dict[str, Any]]:
    session = reconstruct_session(session_file)
    messages = []
    for index, request in enumerate(session.get("requests", [])):
        timestamp = request.get("timestamp", 0)
        text = request.get("message", {}).get("text", "")
        if text:
            messages.append({"index": index, "timestamp": timestamp, "role": "user", "text": text})
        if responses:
            text = response_text(request.get("response", []))
            if text:
                messages.append(
                    {"index": index, "timestamp": timestamp, "role": "assistant", "text": text}
                )
    return messages


def format_message(message: dict[str, Any], full: bool) -> str:
    timestamp = message["timestamp"]
    when = datetime.fromtimestamp(timestamp / 1000).isoformat(timespec="minutes") if timestamp else "?"
    marker = ">>>" if message["role"] == "assistant" else "   "
    if not full:
        return f"{marker} req[{message['index']}] {when}: {message['text'][:80].replace(chr(10), ' ')}"
    indented = message["text"].replace("\n", "\n      ")
    return f"{marker} req[{message['index']}] {when}:\n      {indented}"


def select_messages(
    messages: list[dict[str, Any]], start: int | None, end: int | None
) -> list[dict[str, Any]]:
    if start is None or end is None:
        return messages
    return [message for message in messages if start <= message["index"] <= end]


def print_messages(messages: list[dict[str, Any]], full: bool, as_json: bool) -> None:
    if as_json:
        print(json.dumps(messages, indent=2, ensure_ascii=False))
        return
    for message in messages:
        print(format_message(message, full))
        if full:
            print()



def parser() -> argparse.ArgumentParser:
    common = argparse.ArgumentParser(add_help=False)
    common.add_argument("--workspace", type=Path, default=Path.cwd())
    common.add_argument("--user-data-dir", type=Path, help="explicit VS Code user-data directory")
    common.add_argument("--json", action="store_true")
    selector = common.add_mutually_exclusive_group()
    selector.add_argument("--session")
    selector.add_argument("--uuid")

    result = argparse.ArgumentParser(description=__doc__)
    commands = result.add_subparsers(dest="command", required=True)
    commands.add_parser("list", parents=[common])

    search = commands.add_parser("search", parents=[common])
    search.add_argument("query")
    search.add_argument("-C", "--context", type=int, default=0)
    search.add_argument("--responses", action="store_true")
    search.add_argument("--full", action="store_true")

    tail = commands.add_parser("tail", parents=[common])
    tail.add_argument("-n", type=int, default=10)
    tail.add_argument("--responses", action="store_true")
    tail.add_argument("--full", action="store_true")

    show = commands.add_parser("show", parents=[common])
    show.add_argument("--range", nargs=2, type=int, metavar=("FROM", "TO"), required=True)
    show.add_argument("--responses", action="store_true")
    show.add_argument("--full", action="store_true")

    return result


def main() -> int:
    configure_streams()
    args = parser().parse_args()
    try:
        database, sessions_dir = storage_paths(args.workspace, args.user_data_dir)
        sessions = list_sessions(database, sessions_dir)
        if args.command == "list":
            if args.json:
                print(json.dumps(sessions, indent=2, ensure_ascii=False))
            else:
                for item in sessions:
                    print(f"{item['date'] or '?':16} {item['size']:>10,}  {item['id'][:12]}  {item['title']}")
            return 0

        if not args.session and not args.uuid:
            raise RuntimeError("Specify exactly one of --session or --uuid")
        session = resolve_session(args, database, sessions_dir)
        session_file = sessions_dir / f"{session['id']}.jsonl"
        responses = getattr(args, "responses", True)
        messages = collect_messages(session_file, responses=responses)

        if args.command == "tail":
            if not args.json:
                total = len({message["index"] for message in messages if message["role"] == "user"})
                print(f"Total user messages: {total}")
            print_messages(messages[-args.n :], args.full, args.json)
        elif args.command == "show":
            print_messages(select_messages(messages, *args.range), args.full, args.json)
        elif args.command == "search":
            matches = [index for index, message in enumerate(messages) if args.query.casefold() in message["text"].casefold()]
            selected = set()
            for index in matches:
                selected.update(range(max(0, index - args.context), min(len(messages), index + args.context + 1)))
            print_messages([messages[index] for index in sorted(selected)], args.full, args.json)
        return 0
    except (OSError, RuntimeError, sqlite3.Error, json.JSONDecodeError) as error:
        print(f"ERROR: {error}", file=sys.stderr)
        return 1


if __name__ == "__main__":
    raise SystemExit(main())
```

### Zed session reader

Source: `scripts/zed-threads.py`

``` python
#!/usr/bin/env python3
"""
zed-threads.py — Zed thread reader for session migration.

Reads threads from Zed's local threads.db,
decompresses zstd JSON blobs, and outputs readable conversation text.

Usage:
    python zed-threads.py list [--limit N]
    python zed-threads.py show <thread_id_prefix> [--tail N] [--full]
    python zed-threads.py search <text> [--limit N]


Thread ID prefix: first 8+ chars of the UUID, enough to be unique.

Environment:
    ZED_THREADS_DB  Override path to threads.db
                    Default: %LOCALAPPDATA%/Zed/threads/threads.db
"""

import argparse
import json
import os
import re
import sqlite3
import sys
import textwrap
from datetime import datetime
from pathlib import Path

# Force UTF-8 stdout on Windows
if sys.platform == "win32":
    sys.stdout.reconfigure(encoding="utf-8", errors="replace")
    sys.stderr.reconfigure(encoding="utf-8", errors="replace")

try:
    import zstandard as zstd
except ImportError:
    print("ERROR: pip install zstandard", file=sys.stderr)
    sys.exit(1)


def get_db_path() -> Path:
    env = os.environ.get("ZED_THREADS_DB")
    if env:
        return Path(env)
    local = os.environ.get("LOCALAPPDATA", "")
    if local:
        return Path(local) / "Zed" / "threads" / "threads.db"
    # Linux/Mac fallback
    home = Path.home()
    for candidate in [
        home / ".local" / "share" / "zed" / "threads" / "threads.db",
        home / "Library" / "Application Support" / "Zed" / "threads" / "threads.db",
    ]:
        if candidate.exists():
            return candidate
    print("ERROR: Cannot find threads.db. Set ZED_THREADS_DB.", file=sys.stderr)
    sys.exit(1)


def connect(db_path: Path) -> sqlite3.Connection:
    if not db_path.exists():
        print(f"ERROR: {db_path} not found", file=sys.stderr)
        sys.exit(1)
    return sqlite3.connect(db_path.resolve().as_uri() + "?mode=ro", uri=True)


def decompress(data: bytes, data_type: str) -> dict:
    if data_type == "zstd":
        decompressor = zstd.ZstdDecompressor()
        # Some frames lack content size in header; use streaming
        reader = decompressor.stream_reader(data)
        raw = reader.read()
        reader.close()
    else:
        raw = data
    return json.loads(raw)


def format_timestamp(ts: str | None) -> str:
    if not ts:
        return "?"
    try:
        # Zed uses RFC 3339 with nanoseconds
        # Truncate to microseconds for Python
        ts_clean = re.sub(r"(\.\d{6})\d+", r"\1", ts)
        dt = datetime.fromisoformat(ts_clean)
        return dt.strftime("%Y-%m-%d %H:%M")
    except Exception:
        return ts[:16] if len(ts) > 16 else ts


def format_size(n: int) -> str:
    if n < 1024:
        return f"{n}B"
    elif n < 1024 * 1024:
        return f"{n / 1024:.1f}KB"
    else:
        return f"{n / (1024 * 1024):.1f}MB"


def resolve_thread_id(conn: sqlite3.Connection, prefix: str) -> str:
    """Resolve a thread ID prefix to a full ID. Exits if ambiguous or not found."""
    cur = conn.cursor()
    cur.execute("SELECT id FROM threads WHERE id LIKE ?", (prefix + "%",))
    matches = cur.fetchall()
    if len(matches) == 0:
        print(f"ERROR: No thread matching '{prefix}'", file=sys.stderr)
        sys.exit(1)
    if len(matches) > 1:
        print(f"ERROR: Ambiguous prefix '{prefix}', matches:", file=sys.stderr)
        for m in matches:
            print(f"  {m[0]}", file=sys.stderr)
        sys.exit(1)
    return matches[0][0]


# ─── Commands ───────────────────────────────────────────────────────


def cmd_list(args):
    conn = connect(get_db_path())
    cur = conn.cursor()
    cur.execute(
        "SELECT id, summary, created_at, updated_at, data_type, length(data), folder_paths "
        "FROM threads ORDER BY updated_at DESC LIMIT ?",
        (args.limit,),
    )
    rows = cur.fetchall()
    if not rows:
        print("No threads found.")
        return

    print(f"{'ID':12s}  {'Size':>8s}  {'Updated':16s}  {'Summary'}")
    print("─" * 80)
    for tid, summary, created, updated, dtype, size, folders in rows:
        print(
            f"{tid[:12]}  {format_size(size):>8s}  "
            f"{format_timestamp(updated):16s}  {summary[:50]}"
        )
    conn.close()


def extract_content_parts(content_list: list) -> str:
    """Extract bounded readable text from Zed's content segment array."""
    parts = []
    for seg in content_list:
        if isinstance(seg, str):
            parts.append(seg)
        elif isinstance(seg, dict):
            # Zed uses PascalCase variant tags: {"Text": "..."}, {"Thinking": {...}}, etc.
            if "Text" in seg:
                parts.append(seg["Text"])
            elif "text" in seg:
                parts.append(seg["text"])
            elif "Thinking" in seg:
                t = seg["Thinking"]
                text = t.get("text", "") if isinstance(t, dict) else str(t)
                if text:
                    parts.append(
                        f"[thinking: {text[:200]}...]"
                        if len(text) > 200
                        else f"[thinking: {text}]"
                    )
            elif "ToolUse" in seg:
                tu = seg["ToolUse"]
                name = tu.get("name", "?") if isinstance(tu, dict) else "?"
                inp = tu.get("input", {}) if isinstance(tu, dict) else {}
                if isinstance(inp, dict):
                    keys = list(inp.keys())
                    parts.append(f"[tool: {name}({', '.join(keys[:5])})]")
                else:
                    parts.append(f"[tool: {name}]")
            elif "ToolResult" in seg:
                tr = seg["ToolResult"]
                content = tr.get("content", "") if isinstance(tr, dict) else str(tr)
                if isinstance(content, list):
                    # Tool result content is often [{"Text": "..."}]
                    text_parts = []
                    for item in content:
                        if isinstance(item, dict) and "Text" in item:
                            text_parts.append(item["Text"][:200])
                        elif isinstance(item, str):
                            text_parts.append(item[:200])
                    content = "\n".join(text_parts)
                if isinstance(content, str) and len(content) > 300:
                    content = content[:300] + "..."
                parts.append(f"[result: {content}]")
            elif "Image" in seg:
                parts.append("[image]")
            else:
                # Unknown segment type — show keys
                keys = list(seg.keys())
                parts.append(f"[{keys[0] if keys else '?'}: ...]")
    return "\n".join(p for p in parts if p)


def extract_messages(thread_json: dict) -> list[dict]:
    """Extract messages from thread JSON as role/text/timestamp/id records."""
    messages = []
    raw_messages = thread_json.get("messages", [])

    for msg in raw_messages:
        # Zed format: each message is {"User": {...}}, {"Agent": {...}}, or {"System": {...}}
        if isinstance(msg, dict):
            for role_key in ("User", "Agent", "System", "Assistant"):
                if role_key in msg:
                    inner = msg[role_key]
                    content_raw = inner.get("content", [])
                    if isinstance(content_raw, list):
                        text = extract_content_parts(content_raw)
                    elif isinstance(content_raw, str):
                        text = content_raw
                    else:
                        text = str(content_raw)

                    messages.append(
                        {
                            "role": role_key.lower(),
                            "text": text,
                            "timestamp": inner.get("created_at"),
                            "id": inner.get("id", ""),
                        }
                    )
                    break
            else:
                # Unknown role variant
                keys = list(msg.keys())
                messages.append(
                    {
                        "role": keys[0] if keys else "unknown",
                        "text": "",
                        "timestamp": None,
                    }
                )

    return messages


def cmd_show(args):
    conn = connect(get_db_path())
    thread_id = resolve_thread_id(conn, args.thread_id)
    cur = conn.cursor()
    cur.execute(
        "SELECT summary, data_type, data, created_at, updated_at, folder_paths FROM threads WHERE id = ?",
        (thread_id,),
    )
    row = cur.fetchone()
    if not row:
        print(f"ERROR: Thread {thread_id} not found", file=sys.stderr)
        sys.exit(1)

    summary, data_type, data, created, updated, folders = row


    thread_json = decompress(data, data_type)
    messages = extract_messages(thread_json)

    # Header
    print(f"Thread: {summary}")
    print(f"ID: {thread_id}")
    print(f"Created: {format_timestamp(created)}  Updated: {format_timestamp(updated)}")
    if folders:
        print(f"Workspace: {folders}")
    print(f"Messages: {len(messages)}")
    print("═" * 80)

    # Optionally tail
    if args.tail and args.tail < len(messages):
        messages = messages[-args.tail :]
        print(f"(showing last {args.tail} messages)\n")

    for i, msg in enumerate(messages):
        role = msg["role"].upper()
        ts = ""
        if msg.get("timestamp"):
            ts = f" [{format_timestamp(msg['timestamp'])}]"

        if role == "USER":
            print(f"\n{'─' * 40} USER{ts} {'─' * 35}")
        elif role == "ASSISTANT":
            print(f"\n{'─' * 38} ASSISTANT{ts} {'─' * 32}")
        else:
            print(f"\n{'─' * 38} {role}{ts} {'─' * 32}")

        text = msg["text"].strip()
        if text:
            # Truncate very long messages for overview
            if not args.full and len(text) > 3000:
                print(text[:3000])
                print(
                    f"\n... [{len(text) - 3000} chars truncated, use --full to see all]"
                )
            else:
                print(text)

    conn.close()


def cmd_search(args):
    conn = connect(get_db_path())
    cur = conn.cursor()
    cur.execute(
        "SELECT id, summary, data_type, data, updated_at, length(data) "
        "FROM threads ORDER BY updated_at DESC"
    )
    pattern = re.compile(re.escape(args.text), re.IGNORECASE)
    found = 0

    for tid, summary, data_type, data, updated, size in cur:
        # Search in summary first (cheap)
        if pattern.search(summary or ""):
            print(
                f"{tid[:12]}  {format_size(size):>8s}  {format_timestamp(updated)}  {summary[:60]}"
            )
            found += 1
            if found >= args.limit:
                break
            continue
        # Search in decompressed content
        try:
            thread_json = decompress(data, data_type)
            messages = extract_messages(thread_json)
            for msg in messages:
                if pattern.search(msg["text"]):
                    print(
                        f"{tid[:12]}  {format_size(size):>8s}  {format_timestamp(updated)}  {summary[:60]}"
                    )
                    found += 1
                    break
        except Exception:
            pass
        if found >= args.limit:
            break

    if found == 0:
        print(f"No threads matching '{args.text}'")
    conn.close()



# ─── Main ───────────────────────────────────────────────────────────


def main():
    parser = argparse.ArgumentParser(
        description="Zed thread reader for session migration",
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog=textwrap.dedent("""
        Examples:
          zed-threads.py list
          zed-threads.py list --limit 20
          zed-threads.py show 965cf72c --tail 20

          zed-threads.py search "GigaChat"
        """),
    )
    sub = parser.add_subparsers(dest="command", required=True)

    # list
    p_list = sub.add_parser("list", help="List recent threads")
    p_list.add_argument("--limit", type=int, default=15)

    # show
    p_show = sub.add_parser("show", help="Show thread conversation")
    p_show.add_argument("thread_id", help="Thread ID prefix (8+ chars)")
    p_show.add_argument("--tail", type=int, help="Show only last N messages")

    p_show.add_argument(
        "--full", action="store_true", help="Don't truncate long messages"
    )

    # search
    p_search = sub.add_parser("search", help="Search thread content")
    p_search.add_argument("text", help="Text to search for")
    p_search.add_argument("--limit", type=int, default=10)


    args = parser.parse_args()

    if args.command == "list":
        cmd_list(args)
    elif args.command == "show":
        cmd_show(args)
    elif args.command == "search":
        cmd_search(args)


if __name__ == "__main__":
    main()
```

</details>

# Practice 2: Maintain MEMORY and JOURNAL

One remedy for LLM *amnesia* has been known since the beginning of the
AI era: MEMORY files. “Save context into a file and read it in the next
session” is a trivial and established idea. The concrete seed of our
system was Dany Fabian’s (hi, Dany!) MEMORY system prompt. Sasha started
from that exact prompt, then revised it repeatedly until it evolved into
the current contracts we share here.

A single dumped context has one immediate problem. A new session needs
two incompatible kinds of context:

- the unfinished trajectory of yesterday’s work;
- current local invariants established during previous sessions,
  possibly over months.

Put both into one file and it becomes either a diary too noisy to load
every time or a summary too compressed to resume real work.

At the start of each session, including after restoring prior context,
the assistant reads the complete `MEMORY.md` and the recent tail of
`JOURNAL.md`.

- `MEMORY.md` contains durable cross-session decisions, constraints,
  working preferences, reusable heuristics, and behavioral rules
  discovered during the sessions. MEMORY is maintained *consolidated*:
  it should be compact, non-repetitive, and, most importantly, not
  self-contradictory. Although transported as `tool output` rather than
  the protocol’s `system` role, it is a *dynamic system prompt* by
  function: persistent context explicitly loaded to shape the
  assistant’s behavior.
- `JOURNAL.md` contains the flow of work: attempted approaches,
  observations, intermediate reasoning, unresolved questions, current
  state, and the exact next discriminating action. It is mostly
  append-only and populated automatically by the assistant. JOURNAL
  records consequential working state; the raw AI session history
  preserves every message and tool call. JOURNAL is the operational
  context.

<details class="caption-cut">

<summary>

Relation to `AGENTS.md`/`CLAUDE.md` and the **exact** loading mechanism
</summary>

## Relation to AGENTS.md/CLAUDE.md and the exact loading mechanism {.cut-toc-heading}

The obvious neighboring mechanism is `AGENTS.md`/`CLAUDE.md`
<a href="#ref-anthropic2026projectmemory">[3]</a>, <a href="#ref-agentsmd">[4]</a>. At the
mechanical level, it is similar: persist text and load it into a later
context. The functional contract is different. Repository instruction
files normally carry shared, relatively stable project guidance; our
MEMORY/JOURNAL carries user-assistant continuity, changing rationale,
corrections, failed approaches, and prompt candidates that may span
repositories. A mature rule may be promoted into checked-in `AGENTS.md`,
but the file does not recover the previous session or decide which
personal observation should become team policy.

Both VS Code and Zed place `AGENTS.md` content near the beginning of
each request as part of the *system prompt*
<a href="#ref-microsoft2026vscodeinstructions">[5]</a>,
<a href="#ref-microsoft2026vscodeagentprompt">[6]</a>,
<a href="#ref-zed2026agentsmd">[7]</a>, <a href="#ref-zed2026agentprompt">[8]</a>. Changes
become visible on the next request, but changing that prefix invalidates
the server-side KV cache from the first changed token
<a href="#ref-anthropic2026promptcaching">[9]</a>. At current Claude Opus 4.7 API
list prices, rebuilding a one-million-token prefix costs USD 5.75 more
than reading it from cache.

Claude Code and Codex use a different approach: they snapshot
instructions when the session or run starts; Claude Code also reloads
them after `/compact` <a href="#ref-anthropic2026projectmemory">[3]</a>,
<a href="#ref-openai2026codexagentsmd">[10]</a>. Edits do not affect the active
conversation until an explicit reload. This avoids accidental cache
misses but leaves the session on stale instructions in the meantime.

Neither policy fits changing mid-memory particularly well. This is one
reason we prefer an ordinary file loaded deliberately. The contract is
unusually portable: every agent system we use can read a file through an
ordinary file tool, and that read enters the conversation at the current
position rather than rewriting an earlier request prefix. The current
session keeps using the copy already present in its conversation. We can
edit the durable file without silently changing the active prompt, and
the next session reads the current version during its explicit startup
procedure. Promotion into always-loaded instructions remains useful for
stable rules, but it is a poor default for knowledge that is supposed to
evolve while we work.

**Meta-lesson:** before implementing any AI subsystem, verify the exact
behavior of the *agent harness* (the client/runtime that assembles model
requests and mediates context, tools, and persistence). Which files does
it discover? When are they read? Where do they enter the request:
system/developer instructions, user context, or tool output? Are they
snapshotted per session, watched and reloaded per request, or injected
only after an explicit action? Which prefix does the provider cache,
what invalidates it, and how are cache reads and writes billed? Do not
infer these semantics from the filename or from another client
implementing the same convention. Otherwise the system can behave
*surprisingly* or produce a *surprising* bill.

</details>

JOURNAL allows the AI to resume the work immediately and preserves the
consequential history of successes, failures, and unresolved state. That
is very useful for post-hoc analysis, again, with AI. Say, writing a
pull-request description or daily report—try it! JOURNAL grows
incrementally; we only *append* the logs. It has a tendency to grow
really fast, which is why during routine work we read only the tail.

MEMORY keeps *curated, reusable transient contracts established during
the sessions*: statements too important to drop at the end of a session
but not mature enough to be carved into a prompt, configuration file,
README, wiki page, etc. Unlike the mostly append-only JOURNAL, MEMORY is
maintained as a *random-access* structure: the AI adds details, replaces
superseded statements, and reorganizes entries as understanding changes.

**JOURNAL records what we did; MEMORY records what we agreed should
govern future work.** We call this layer *mid-memory*: it sits between
the current conversation context and persistent artifacts.

Periodic maintenance (commonly called *memory consolidation*
<a href="#ref-zhou2025mem1">[11]</a> in agent-memory research) is also necessary.
During our consolidation (see the SKILL below), the MEMORY notes are
processed:

- conflicts are resolved;
- outdated information is corrected or removed;
- duplicates are merged;
- statements are refined and shortened;
- irrelevant content is removed:
  - weak, ad-hoc, or simply wrong statements and garbage → delete. Raw
    AI sessions preserve the underlying record;
  - accidentally memoized sensitive information (credentials, personal
    data, IP, etc.) → remove and implement the appropriate cleanup
    protocol;
  - invariant that can be *cheaply* derived from code and configuration
    → delete;
- mature content is promoted to a stronger mechanism:
  - stable behavioral rule → *instructions* (in VS Code terminology),
    becoming part of the actual system prompt;
  - repeated executable procedure → script or SKILL;
  - local rationale or design decision → code comment or README;
  - stable deterministic invariant that is cheaply enforceable →
    assertion, test, type, or validation schema; **do not use AI as a
    unit-test engine;**
- important but still-uncertain notes are kept in MEMORY with explicit
  provenance and scope.

What about “competing” approaches? Shared AI memory is already a highly
developed design space spanning graph and temporal retrieval, LLM-native
compression, agent self-editing, staged consolidation, explicit human
approval, and transparent file persistence
<a href="#ref-firstintent2026survey">[12]</a>. Three systems illustrate the
relevant compromises. Augment Code’s Memory Review lets users approve,
edit, or discard each proposed memory
<a href="#ref-shah2026memoryreview">[13]</a>. Codex generates local memory from
prior chats in the background and exposes separate extraction and
consolidation configuration <a href="#ref-openai2026memories">[14]</a>. Claude
auto-memory and Copilot Memory emphasize low-friction automatic capture;
Claude combines it with human-authored `CLAUDE.md`, while Copilot
validates repository memories against cited code and expires unused
entries after 28 days <a href="#ref-anthropic2026projectmemory">[3]</a>,
<a href="#ref-github2026copilotmemory">[15]</a>.

Our specific trade-off is human-controlled *accumulation*, *curation*,
and *promotion* in an ordinary editable artifact. MEMORY does not
accumulate raw data or random phrases from the conversation; that
high-recall role belongs to JOURNAL and raw session history. The AI may
propose a candidate, but almost every admission is discussed and every
MEMORY change is reviewed. AI can suggest stronger promotion targets;
promotion itself is not delegated.

This costs human attention but keeps the cached precedent corrigible. An
observed phrase does not silently become precedent to be replicated, and
mature entries may leave MEMORY for a stronger mechanism.

**Artifacts.** Public memory instructions, consolidation procedure, and
paired MEMORY/JOURNAL examples are embedded under the cut immediately
after this practice. The production
`prompts/memory-consolidation.prompt.md` contains additional local
policy and historical constraints.

<details class="caption-cut">

<summary>

From incidents to precedents
</summary>

## From incidents to precedents {.cut-toc-heading}

Daily work produces different kinds of reusable precedent. Sometimes we
preserve a way to investigate, sometimes a procedure that should improve
through recurrence, and sometimes the reason an apparent result must be
rejected.

### History as architecture documentation

While implementing a feature, I could not integrate it correctly. After
several attempts, I asked the AI to inspect Git history: who had made a
similar change, when, and exactly which files had changed together? The
previous working change exposed an integration contract that neither the
current code nor the build documentation made obvious. I then asked the
AI to *memoize* the method—to preserve it as reusable precedent—rather
than this one fix:

> When adding to a configuration system you do not fully understand,
> search history for the most distinctive existing entry and inspect the
> complete change that introduced it. Use that diff to discover the
> parallel manifests, exports, packaging, localization, or generated
> files the new entry also requires.

The first use solved one feature-integration problem. The promoted
heuristic now applies whenever an unfamiliar integration scatters its
real contract across configuration, packaging, generated artifacts, or
other companion files.

### Repair the procedure, not only the current file

Some build-configuration files kept generated locale lists on lines more
than a thousand characters long. A normal structured edit triggered an
editor formatter, reflowing the complete file when only one entry was
intended to change. Applying another text edit to repair the result
risked transforming it again, so we restored the exact committed bytes
through Git and changed the procedure rather than merely fixing that
file.

The remembered procedure now treats these files as byte-sensitive: make
a surgical line insertion outside the formatter, use a fresh shell scope
so stale variables cannot affect the operation, then verify the exact
line-count delta, the number of matching entries, and the trailing
newline. When the problem recurred, the assistant could recover and
refine that procedure instead of rediscovering the failure:

```
corrupted edit
→ exact byte restoration
→ reviewed repair procedure
→ postcondition checks added by recurrence
```

A workaround becomes cumulative only when the durable procedure changes
with what the next execution teaches us.

### A 39% regression that was not a regression

A candidate computational GPU kernel appeared to reduce token-generation
speed by 39%. We inspected the implementation and reverted it before
noticing that two detached workers were saturating the same GPU during
the candidate run. The code had changed, but so had the experiment.

The invalid result remained useful because JOURNAL preserved the
sequence and MEMORY preserved the corrected rule:

> Benchmark conditions include machine load. Before timing, verify
> detached jobs and GPU/accelerator utilization. A large unexpected
> regression is first a measurement-validity question.
> Output-correctness and model-quality checks may remain valid under
> contention; timing comparisons do not.

The point of MEMORY was not to keep the original conclusion alive. It
was to prevent the same invalid comparison from controlling the next
decision.

### Governance in practice

The formal words are *accumulation*, *curation*, and *promotion*. In
practice, the review sounds more like this:

> — Memoize that as YOUR judgement =)
>
> — No, that is an editorial contract. I mean, that exact statement is
> not for the doc. Memoize it separately.

The MEMORY culture changes the current work, not only the next session.
Before writing an entry, the assistant may ask:

> — Is this global or project-local?
>
> — Is it the intended architecture, a temporary workaround, or a hack
> we accepted?
>
> — Is it an observation, hypothesis, decision, or preference?
>
> — What is its provenance, and what would make it obsolete?

Answering forces a high-level judgment while the context is still warm.
These small interventions form the decision record: they distinguish a
reviewed contract from a plausible summary and maintain a shared model
of the system.

That is the division of labor. We delegate low-level search, editing,
command execution, and repetitive mechanics to AI while retaining
purpose, architecture, trade-offs, and ownership. Writing MEMORY is the
checkpoint where generated work is lifted back into a human-understood
model: what changed, why we accepted it, and what should govern the next
decision.

</details>

<details class="caption-cut">

<summary>

Reference memory instructions, consolidation procedure, and examples
</summary>

## Reference memory instructions, consolidation procedure, and examples {.cut-toc-heading}

### Cross-session memory instructions

``` markdown
# Cross-Session Memory Instructions

At the start of a new session:

1. Read the complete `MEMORY.md`.
2. Read the recent tail of `JOURNAL.md` needed to recover active work.

Use the files for different purposes:

- `JOURNAL.md` records consequential working trajectory: attempts, failures, observations, active state,
  artifact paths, and the next discriminating action. Append while work develops.
- `MEMORY.md` stores reviewed reusable precedent: constraints, rationale, heuristics, preferences, ownership,
  and uncertain but important findings. Revise existing entries instead of appending contradictory layers.

Do not silently convert conversation into precedent. The AI may propose a memory candidate, but consequential
admission, scope, and promotion are reviewed with the operator. Preserve attribution, epistemic status,
relevant date/version, and a pointer to primary evidence when being wrong would matter.

When a memory matures, consider the mechanism that fits its type:

- behavioral rule → instruction or operating contract;
- repeated procedure → script or SKILL, when its contract is worth defining and maintaining;
- deterministic invariant → design, type, schema, assertion, or test;
- local rationale → comment or README;
- professional routing → address book;
- obsolete, contradicted, derivable, or irrelevant material → correct, relocate, or delete.

MEMORY is context and cached precedent, not absolute truth. Current primary artifacts and reproducible
observations can invalidate it. Report contradictions and repair the cache rather than choosing silently.

When memory behavior is surprising, explain which remembered rule or fact influenced the response and cite its
exact entry. Treat the explanation as a hypothesis; verify it against the loaded files and available tool
trace. Diagnose both false negatives (relevant memory was not used) and false positives (irrelevant, stale, or
overgeneralized memory controlled the response), then correct and retry.
```

### MEMORY example

``` markdown
# MEMORY — Example

## Benchmark validity

**Observed 2026-08-11; project: ExampleKernel.** A candidate appeared to regress generation throughput by 39%,
but two detached workers were using the same GPU during the candidate run. The timing comparison was invalid;
correctness checks remained usable.

**Reusable rule:** before comparing timing or throughput, verify detached jobs and GPU/accelerator
utilization. A large unexpected regression is first a measurement-validity question. Record machine load with
every timing cell.

**Primary evidence:** `JOURNAL.example.md`, benchmark log `example-kernel-20260811.log`, commit `0123456`.
```

### JOURNAL example

``` markdown
# JOURNAL — Example

## 2026-08-11 — ExampleKernel comparison

Purpose: determine whether the candidate kernel increases fixed-workload throughput without changing
model-output quality.

The baseline ran on an idle GPU. The candidate reported a 39% generation regression. Before changing code,
checked active jobs and found two detached decomposition workers saturating the same device. The result is
invalid; the magnitude is not plausible for the candidate's small scheduling change.

Completed:

- output-correctness and model-quality checks passed;
- invalid timing artifacts retained for diagnosis;
- candidate revert recorded but should not be interpreted as evidence against the design.

Next action: stop or finish the unrelated workers, restore the candidate, then rerun baseline and candidate
under matched idle conditions with utilization recorded.
```

### Memory consolidation procedure

``` markdown
# Memory Consolidation

Maintain an existing MEMORY/JOURNAL system. A consolidation pass is not a summary of the latest conversation.

## Procedure

1. Read the complete relevant MEMORY files and the JOURNAL ranges needed to recover provenance.
2. Find exact and semantic duplicates, contradictions, stale paths/versions, misplaced project material,
   unresolved claims stated as facts, and procedures or invariants that belong in stronger artifacts.
3. For each candidate, choose explicitly:
   - keep;
   - rewrite;
   - merge;
   - correct;
   - relocate;
   - promote;
   - delete.
4. Preserve meaningful disagreement, attribution, uncertainty, scope, and evidence pointers.
5. Update dependent instructions, scripts, indexes, and references when an entry moves.
6. Review the diff for accidental loss of distinct reasoning and run formatting checks.

The AI may perform search and mechanical editing. A human reviews admissions, semantic merges, corrections,
deletions, and promotion decisions. Automatic consolidation without review can recursively summarize earlier
model output into apparent authority without adding evidence.

A successful pass makes the memory system smaller, more accurate, or easier to act on. Merely appending
today's summary is memoization, not consolidation.
```

</details>

# Practice 3: Reconstruct and reuse workflows before abstracting them

In ordinary software engineering, reuse is usually framed through two
polar approaches. In *generalization* or *abstraction*, a potentially
repeated procedure is factored into a dedicated abstraction. In
*translation*, practically known as **copy-paste**, we copy existing
code without raising its abstraction level (that is, without creating a
new class, interface, template, prompt, SKILL, etc.), adapt it to the
new setting, use it, and repeat.

History, engineering practice, and folklore push us toward the first
approach. Not because abstraction is an ultimate *virtue*: it is a
*trade-off*. Generalization is conceptually hard and, when done
properly, requires significant cognitive effort. It also demands a kind
of fortune-telling because the system must account for **future** use
cases. People often consciously prefer copy-paste or boilerplate because
abstraction is the worse option.

AI significantly changes the balance in this equation. The shift is
relative. AI makes generalization substantially easier as well, no
question, but the problem remains conceptually hard. At the same time,
AI is amazingly good at *translation*. A human can patch a few paths in
an old script, but reliably translating an environment-bound procedure
across changed paths, versions, tools, and surrounding code is tedious
and error-prone. Building a proper abstraction was often the safer form
of reuse. AI can perform that concrete translation cheaply enough that a
preserved execution becomes reusable even before it becomes an
abstraction. **AI can follow instructions regardless of their origin;
preserving a concrete execution can therefore make it reusable before a
dedicated SKILL exists.**

A proper memory system therefore creates a new equilibrium for pipeline
reproducibility: preserve enough evidence to reconstruct and translate
the concrete workflow later.

```
goldfish
→ reinvent the pipeline at every recurrence

premature abstraction
→ design and maintain a reusable contract before recurrence supplies its requirements

mid-memory
→ preserve the concrete execution and its rationale
→ translate it when a similar problem appears
→ promote only when repeated use earns the abstraction
```

The session archive acts as an automatic work log and macro recorder. It
preserves purpose, commands, output, failures, corrections, and
discussion. JOURNAL supplies the approximate location and active state.
MEMORY preserves selected architecture, constraints, and reasons.
Together they leave a cheap path back:

```
“Remember, we did something like that”
→ JOURNAL supplies search vocabulary and trajectory
→ MEMORY restores rationale and constraints
→ session history supplies exact mechanics and failures
→ current files and logs establish present reality
→ rerun or translate the workflow
```

The key is keeping the AI involved in state transitions, not
theatrically routing every keystroke through it. Manual work still
belongs to the record. Editing directly in the shared editor, letting
the AI reread the result, and discussing the change remains an
AI-assisted, logged workflow. Sometimes pressing `ddp` in Vim mode is
simply better than asking, “Dear AI, swap these two lines.” The reread
and review reconcile a human state transition into the shared record.

Promotion becomes worthwhile when the stronger mechanism buys a concrete
property:

- repeated execution is common enough to repay maintenance;
- enough diverse executions have accumulated to reveal stable invariants
  and variation points;
- abstraction can reveal new properties of the system or improve
  understanding;
- ambiguity or operational risk makes ad-hoc translation unsafe;
- independent testing or deterministic enforcement is required;
- narrower loading reduces irrelevant context;
- another person or agent needs a self-contained procedure.

<details class="caption-cut">

<summary>

Machine setup
</summary>

## Machine setup {.cut-toc-heading}

A first machine setup may contain package searches, failed commands,
driver conflicts, configuration choices, and verification output. The
next machine rarely needs a byte-for-byte replay: the AI can recover the
execution, retain decisions that still apply, adapt paths and versions,
and avoid failures already paid for. Repeated setups may later justify a
reviewed installation script or playbook.

</details>

<details class="caption-cut">

<summary>

Urgent bug
</summary>

## Urgent bug {.cut-toc-heading}

An urgent integration fix once depended on testing an unpublished
upstream package inside a larger application. A previous session had
already discovered the non-obvious mechanics: replacing the compiled
library was insufficient because the runtime loaded a separate bundled
artifact, and the build could rematerialize the installed package. We
recovered that concrete execution, adapted the package locations, copied
the source, compiled library, and runtime bundle, verified hashes and a
load-bearing marker, restarted the bundler, and continued debugging.

Pausing to design `abstract-fix-abstract-bug.SKILL` would not have
improved the fix. With one or two examples, the resulting “abstraction”
may amount to:

> Find a pull request with a similar bug and apply it while adjusting
> the file paths.

That adds almost no capability over reconstructing the concrete prior
execution. Fix the bug, preserve the new sample, and generalize later if
recurrence reveals a real contract.

</details>

Shareability is a particularly strong signal. MEMORY is personal and
contextual; copying the complete memory system to transfer one procedure
is the wrong boundary. Extract the reusable fragment, define its
contract, and test it independently. A SKILL is not a universal virtue,
but it can be the right publication unit.

That applied directly while writing this post. We implemented tools and
extracted SKILLs for workflows we used regularly but had never needed to
name. A typical request was informal: “Find how we configured the
previous Linux machine and do the same here.” Private translation
already worked. Publication created a reason to define a self-contained
interface.

Then the recursion became explicit:

> — We can practice what we preach and reconstruct
> `reconstruct-workflow`. Inception, LOL!

So we used *the unnamed reconstruction workflow to reconstruct the
workflow for reconstructing workflows*. One level deeper, and the
abstraction still leaked. MEMORY and JOURNAL supplied search anchors;
session history supplied actual executions; current files supplied the
surviving state. We still had to choose the new artifact’s name, scope,
interface, stopping condition, and safety contract. The logs contained
the procedure. They did not contain the abstraction.

The exercise was not *empty*. Formalizing the workflow exposed and
clarified its data paths: where anchors come from, how to search bounded
session evidence, when to return to current files and logs, and where
reconstruction must stop before present design judgment begins.
Privately, we already used reconstructed workflows without a SKILL.
Daily. We extracted this one because shareability justified a
self-contained publication unit.

That is also the argument *for* abstraction. Proper extraction is
cognitive work. As in ordinary software design, finding invariants and
variation points can reveal missing cases, sharpen interfaces, and
expose properties of a procedure that were invisible in one execution.
The work is potentially rewarding and may improve the procedure.

**Try it.** At the next real recurrence, retrieve one concrete prior
execution before designing a universal procedure. Adapt it once. If
recurrence reveals stable invariants and the expected reuse exceeds the
maintenance cost, promote the accumulated understanding into a tested
script, SKILL, runbook, or another appropriate artifact.

> **AI did not make abstraction obsolete. It made concrete experience
> reusable before abstraction.**

**Artifact.** The production `reconstruct-workflow` SKILL below performs
the archaeology step. Contract design and any later promotion remain
separate human-reviewed work.

<details class="caption-cut">

<summary>

Workflow reconstruction prompt
</summary>

## Workflow reconstruction prompt {.cut-toc-heading}

The complete prompt is included from its canonical source file.

### Reconstruct-workflow SKILL

Source: `prompts/reconstruct-workflow.prompt.md`

``` markdown
---
name: reconstruct-workflow
description: "Recover and reuse a previously performed workflow from MEMORY, JOURNAL, session history, files, and logs; use when: remember we did something like this, recover old pipeline, rerun prior procedure, or extract a shareable script or SKILL."
---

# Reconstruct Workflow

Recover a useful earlier procedure that was not formalized when first performed.
The common case is informal: the operator remembers that we did something similar, while the exact sessions, commands, or
rationale are fuzzy. The value of this SKILL is tool-driven archaeology and evidence aggregation—not the abstract idea of
promotion, which ordinary memory management and `create-skill` already cover.

## When to use

- “Remember, we did something like this?”
- A previous repair, experiment, setup, or command sequence may solve the current problem.
- We remember the rationale but not the mechanics, or the mechanics but not why they had that shape.
- A recovered workflow may now deserve a shareable script, SKILL, or runbook.

## Evidence layers

- **MEMORY:** rationale, constraints, reusable lessons, and likely vocabulary.
- **JOURNAL:** approximate episode, date, machine, failures, filenames, logs, and search terms.
- **Session history:** exact dialogue, commands, tool calls, outputs, edits, and corrections.
- **Current files and logs:** what survives now and whether the old procedure still applies.

These layers are fallible and complementary. Even ideal history can recover only the performed pipeline and the decisions
explicitly discussed at the time. It does not determine the future abstraction: name, invocation trigger, supported scope,
inputs, outputs, safety boundary, generality, or maintenance contract. Those are design decisions for the present use case,
not facts latent in the transcript. Do not invent them—or missing rationale—from a surviving command or final artifact.

## Procedure

### 1. Collect anchors already in context

Start from the present request and the MEMORY/JOURNAL material already loaded. Extract a few distinctive anchors:

- filename, symbol, command, error, person, machine, date, project, or unusual phrase;
- the old purpose and the current purpose;
- any recorded artifact or log path.

Do not design a reconstruction schema before searching. Usually one good literal is enough.

### 2. Check the actual session history

Use the editor-appropriate read-only history tool:

- VS Code: `vscode-session-reader.py list/search/show/tail`
- Zed: `zed-threads.py list/search/show`

Search the most distinctive literal first. If it matches several sessions, use date/project/title and the JOURNAL episode
to identify the relevant session or sequence of sessions. Prefer exact session IDs after discovery.

Check session history by default even when MEMORY/JOURNAL appear sufficient. They are curated secondary sources; the raw
history records actual prior usage and lets us test whether the reconstructed workflow matches what happened. Skip history
only when the operator explicitly forbids it, the source is unavailable, or the retrieval cost is clearly disproportionate to the
current consequence; state the omission.

Redirect potentially large output to a scratch file, inspect byte and line count, then read only a bounded coherent range.
Do not load raw session databases or unbounded tool output into model context.

### 3. Read outward only as needed

Inspect the surrounding turns and recover:

- what we were trying to do;
- the exact mechanics that mattered;
- failures and corrections;
- environment assumptions;
- observed completion checks.

If the first range answers the current question, stop. Expand to more turns, tool traces, or another linked session only
when a concrete gap remains.

### 4. Check current reality

Before reuse, inspect the current source, scripts, configuration, Git state, logs, tool versions, paths, and machine state
relevant to the recovered workflow. Separate still-valid rationale from stale mechanics.

Historical commands are evidence, not present authorization. Recovered SSH, destructive, credential, external-write,
commit, push, or publication actions retain their normal current permission boundaries.

### 5. Use the result at the requested level

Choose the smallest useful result:

- answer a factual question from the recovered evidence;
- rerun or adapt the procedure when currently authorized;
- borrow one technique without recreating the whole pipeline;
- restore a lost artifact;
- update MEMORY only when reconstruction changes reusable understanding;
- append JOURNAL only when reconstruction changes active state or records a consequential discovery;
- present the recovered information for discussion; only after its purpose, scope, interface, and remaining uncertainty
  are agreed should a separate `create-skill` invocation—or script/runbook edit—perform promotion.

Do not manufacture a new named artifact merely because reconstruction succeeded. This SKILL stops at evidence-backed
reconstruction and discussion; it never creates another SKILL automatically. Conversely, shareability alone can justify a
later, explicit promotion even when the private workflow already works without a SKILL.

## Completion checks

The reconstruction is complete when:

1. the relevant historical evidence chain is identified; a workflow may span several sessions;
2. current purpose, old rationale, and exact mechanics are distinguishable;
3. consequential claims point to session/file/log evidence or are marked uncertain;
4. current-environment differences have been checked;
5. the requested result is delivered or one concrete missing fact is reported.

## Privacy

Do not promote unrelated private conversation details into MEMORY, documentation, scripts, or SKILLs merely because they
appeared in the recovered session. Preserve operational purpose, evidence, constraints, and decisions.
```

</details>

# Practice 4: Preserve provenance throughout the work

This is good engineering and writing practice independently of AI. AI
merely makes both losing and preserving provenance much cheaper.

The useful reflex is:

> **How do I know this?**

```
I remember the source
→ record it now

I do not remember the source
→ ask AI to investigate

the question is consequential, ambiguous, or technically difficult
→ investigate together

source and claim are established
→ attach provenance to the artifact
→ memoize the reusable conclusion
```

Keep the source attached while information moves from observation
through interpretation and decision into code, documentation, memory,
and publication. Preserve enough to distinguish a report, observed fact,
inference, hypothesis, decision, preference, and failed approach;
include the date, scope, conditions, and primary artifact when they
affect how the statement should be used.

Provenance restores type information that prose otherwise loses. “A
colleague reported this behavior,” “the AI proposed this
interpretation,” “the experiment observed this result,” and “we decided
to accept this trade-off” should not return in the next session as four
indistinguishable facts. A paper receives an ordinary citation.
Knowledge recovered from code keeps the path, symbol, and commit that
bind it to a concrete version.

In ordinary professional work, this mostly means writing normal
artifacts properly: a useful pull-request description, issue link, test
result, design note, experiment report, code comment, runbook, or commit
message. AI can follow the links, inspect code and history, locate
references, check whether they support the claim, and draft the formal
record. The human still verifies the evidence chain and owns the final
statement.

Correctly attributing an AI contribution to the AI also weakens a
*confirmation loop*. If a model-generated interpretation is stored as
“the AI proposed X,” a later session can challenge it against evidence.
If consolidation silently turns it into “we know X,” the model reloads
its own output as human-approved authority and confidently confirms it
again. **Attribution does not guarantee an escape from that dead-loop,
but it at least preserves such a *possibility***.

Provenance is a cached trust chain:

```
claim → source → conditions → interpretation → decision
```

The cache saves repeated archaeology, but it can become invalid when the
source changes, conditions differ, an interpretation was wrong, or
stronger evidence appears. Neither human nor AI should treat MEMORY,
documentation, citations, or generated summaries as absolute truth.

**Try it.** Before completing a consequential change or claim, ask: “How
do we know this?” Use AI to trace the answer through the available
links, diff, history, logs, and test output. Preserve the verified chain
where the next engineer, being human or AI, can find it.

**Artifact.** The provenance template below provides optional fields and
one filled example. It is a prompt for judgment, not a mandatory
database schema.

<details class="caption-cut">

<summary>

A repeated key became a scoped RDP hypothesis
</summary>

## A repeated key became a scoped RDP hypothesis {.cut-toc-heading}

On an intermittent train connection, one intended keypress produced
`AIIIIIIII...`. “Linux input bug” was a plausible first description, but
the system logs showed a narrower chain: GRD stayed alive; transport
resets and failed reconnects occurred; GDM failed several handovers; and
a later successful session coincided with a GLib reference-count
assertion. The Fleet note therefore preserves separate fields in prose:
observed symptom, Windows-RDP negative control, exact timestamps/logs,
current key-up/cleanup hypothesis, and what a controlled reproduction
must distinguish. The source chain did not merely support the first
claim; it changed its scope.

</details>

<details class="caption-cut">

<summary>

The review bot was right
</summary>

## The review bot was right {.cut-toc-heading}

A CI review bot suggested a TypeScript constraint that infers an object
type while rejecting extra keys, even when ordinary excess-property
checking no longer helps:

``` typescript
type Rectangle = { width: number; height: number };
type NoExtraKeys<T, Shape> = Exclude<keyof T, keyof Shape> extends never ? unknown : never;

function draw<T extends Rectangle & NoExtraKeys<T, Rectangle>>(value: T): void {}

draw({ width: 10, height: 20 });
// @ts-expect-error: depth is not part of Rectangle
draw({ width: 10, height: 20, depth: 30 });
```

Both Sasha and Drinkins rejected `T extends ... NoExtraKeys<T, ...>` as
illegally circular. A minimal compiler probe proved the bot right:
TypeScript infers `T`, then checks the constraint. Our objection came
from a real neighboring restriction: TypeScript rejects some
*immediately circular* constraints and recursive conditional types
<a href="#ref-typescript2018conditionaltypes">[16]</a>,
<a href="#ref-typescript2022circularconstraints">[17]</a>. But this deferred key
check has compiled since TypeScript 2.8, the first release that could
express it.

Without provenance, consolidation could turn the episode into “we found
a cleaner TypeScript pattern.” The code would be right, but the system
would learn the wrong lesson: our judgment produced the improvement. The
actual chain was: the bot proposed it; both Sasha and Drinkins objected;
the compiler settled the question; the accepted design follows the bot.
Correct provenance of errors matters because future calibration depends
on knowing which source failed and which source supplied the correction.

</details>

<details class="caption-cut">

<summary>

Provenance template
</summary>

## Provenance template {.cut-toc-heading}

### Provenance template

```` markdown
# Provenance Template

Use only the fields that materially affect interpretation. Provenance is an additional information dimension,
not a mandatory ceremony.

```text
Statement:
Origin: person / AI / code / experiment / document
Epistemic type: report / observation / inference / hypothesis / decision / preference
Scope and conditions:
Date or version:
Primary reference: path + symbol + commit, URL + section, session range, log, or artifact hash
What would invalidate or supersede it:
```

## Example

```text
Statement: Intermittent transport stalls may leave a remote key in repeated state.
Origin: observed during one Linux GRD session; mechanism proposed by the AI
Epistemic type: observation + unverified hypothesis
Scope and conditions: mstsc → system GRD over an unstable train connection
Date or version: 2026-08-21; record exact GRD/FreeRDP versions before publication
Primary reference: GRD journal around 19:40–19:44; Fleet/P620/README.md
Would invalidate: reproduction in a local editor without RDP, or evidence that the client generated repeated
key events
```
````

</details>

# Practice 5: Preserve social provenance

Sometimes provenance becomes personal attribution. In software
engineering, the source may be the person who introduced the behavior,
owns the component formally, has actually maintained it, reported the
failure, or should review the change. In a large organization,
repositories, teams, aliases, reorganizations, and formal versus
practical ownership make that topology expensive to rediscover.

Our concrete implementation is a professional address book. Keep two
artifacts separate: a small loading/routing rule and a changing
`PEERS.md` registry of people, accounts, expertise, ownership, reports,
and reviewer conventions.

This gives the assistant enough identity resolution to connect a commit
author, a report, and a review account without turning expertise into
ownership or a person’s report into product fact. The mundane payoff is
immediate: relevant pull requests receive the right reviewers, ideas
keep their attribution, and questions route to people who can actually
answer them.

**Try it.** Create one compact entry per collaborator: names/accounts,
expertise, ownership, contributions or reports, and when to involve the
person. Keep the behavioral rule separate so the registry can change
without rewriting an always-loaded prompt.

Exactly which information to accumulate is up to you. But I would
recommend: **keep a phonebook, not a dossier**. Respect people’s
privacy.

**Artifacts.** The complete loading rule and an artificial directory are
included below.

<details class="caption-cut">

<summary>

PEERS loading rule and artificial directory
</summary>

## PEERS loading rule and artificial directory {.cut-toc-heading}

### Loading and routing rule

```
Read PEERS.md on demand for attribution, communication, ownership,
expertise, reviewer selection, or deciding whom to involve.
Use it as routing and provenance data, not as authority about a person.
Preserve whether an entry records formal ownership, observed expertise,
a report, a contribution, or a reviewer convention.
Current evidence may supersede it; update PEERS.md when it does.
**Keep a phonebook, not a dossier.** Respect people's privacy.
```

### Artificial PEERS directory

``` markdown
# PEERS — Artificial Example

This example uses generated mock data to respect our real peers' privacy.

- **Rafael White** — rendering engineer. Aliases/accounts: `rafwhite`, `rwhite`,
  `rafael.white@example.invalid`. Owns the desktop compositor integration. Reported intermittent frame pacing
  on build 2041; treat this as Rafael's report until reproduced. Request review on compositor or
  frame-scheduling changes; PR identity may appear as `rwhite`.

- **Ada Gross** — telemetry engineer. Aliases/accounts: `adag`, `ada-gross`, `ada.gross@example.invalid`.
  Expertise: event schemas and correlation identifiers; not the owner of the rendering path. Consult on
  telemetry contract changes; review identity may appear as `adag`.
```

</details>

# Practice 6: Evolve the operating contract through real work

A common approach to prompt engineering is top-down design: write the
desired behavior before use. That is legitimate and often necessary.
This practice adds a complementary path: an operating contract can also
evolve through actual use.

Start with the priors you already know. “You are a Rust developer with
ten years of experience” establishes a professional domain and expected
level of rigor; it is only a partial contract. Company policy,
legislation, security boundaries, established operational contracts, and
deterministic engineering invariants are stronger cases for direct
encoding: their requirements already exist before the assistant begins
work.

But even the most reasonable priors should not be considered *sacred*. A
known requirement may be expressed badly, scoped too broadly, assigned
the wrong authority, or enforced through the wrong mechanism. Real work
can show that a security boundary blocks legitimate operations, that an
instruction creates harmful side effects, or that a behavioral rule
belongs in code, configuration, or a test instead. Both the actual
requirements and the LLM may (and probably will) change over time. Thus,
the evolution applies to priors too: retain, narrow, strengthen,
relocate, or even remove them when evidence changes the design.

Evolutionary prompt engineering also discovers requirements that were
not knowable in advance. A plausible initial contract meets real work,
fails in a specific way, receives a correction, and is tried again. Our
operating contract grew mainly through that loop:

```
recover the previous session + keep one simple MEMORY file
→ split active trajectory into JOURNAL when one file becomes noisy
→ add review, provenance, and consolidation as precedent accumulates
→ promote mature rules, procedures, invariants, and routing into stronger artifacts
→ let repeated behavioral rules crystallize into an operating contract
→ add indexing, embeddings, databases, or shared infrastructure only when scale requires them
→ evolve and repeat
```

> **Encode what is already known; let real work reveal what is not.
> Prompts are not carved in stone.**

The two paths meet. Top-down design encodes known requirements as
initial constraints; experience tests whether their wording, scope,
authority, and implementation produce the intended behavior. Evolution
captures working agreements revealed only through use; repeated evidence
may promote them into the operating contract or deterministic
enforcement. Its weakness is equally familiar from software engineering:
it can overfit one operator and one history, preserve a locally
convenient mistake, and optimize subjective convenience without an
external acceptance criterion.

Another important component of *evolutionary prompting* is feedback from
the AI itself: a form of *intellectual debugging*. Use the model for
self-diagnostics. Ask which remembered rule influenced the response, why
it applied, and where it came from.

> — Drinkins, what the heck?! Explain yourself!

is a *very* common request in our collaboration. Do not expect *magical*
introspection. The model can identify visible instructions and
remembered premises that plausibly influenced its answer, but it cannot
inspect its actual internal activations. Treat the explanation as a
debugging hypothesis and verify it against the loaded files, references,
and tool trace:

```
observe → explain → inspect → correct → repeat
```

The loop described here is therefore scoped to a single-user personal
assistant: one operator can observe behavior, judge whether a correction
helped, and own the resulting contract. **It does not transfer directly
to autonomous or multi-user systems, where conflicting preferences,
authority, security boundaries, deployment, versioning, and measurable
acceptance criteria require deliberate design *in advance*.**

Put prompt candidates into MEMORY first when possible. The effect is
easier to observe and revise than an immediate change to the
highest-authority prompt, and dynamic MEMORY can remain outside the
stable system-prompt cache prefix. Promotion is one path, not an
obligation: a private contract that already works from MEMORY may need
no stronger artifact.

Not everything belongs in that contract. A repeated procedure may become
a script or SKILL; a deterministic invariant belongs in a test, type,
schema, assertion, or code; rationale may move to a comment or README;
professional routing belongs in the address book. Behavioral rules and
stable working agreements are candidates for the prompt.

<details class="caption-cut">

<summary>

Bootstrap before optimizing
</summary>

## Bootstrap before optimizing {.cut-toc-heading}

An elaborate memory architecture is complex; a bootstrap is not. Sasha
started from Dany’s working MEMORY prompt, copied it, used it, and
revised it until the two systems diverged. The process resembles copying
a working Linux installation when no installer exists: the copy provides
enough continuity to boot; later use determines what the system becomes.

This post offers the same kind of seed. The embedded prompts are not a
finished universal architecture or a sanitized claim to reproduce
Drinkins. They are a bootable starting state that readers can copy,
operate, criticize, and evolve. An abstract idea and a reviewable
running contract have different activation costs.

Without a small continuity mechanism, the assistant repeatedly forgets
why a larger knowledge base was designed, which alternatives failed, and
what the next experiment should test. It makes no sense to discuss an AI
knowledge base with a goldfish. To design an evolving memory system with
AI, first give the AI enough memory to continue designing it.

</details>

The example below is deliberately minimal: a starting point to test and
evolve, not our complete operating contract.

<details class="caption-cut">

<summary>

Minimal operating contract
</summary>

## Minimal operating contract {.cut-toc-heading}

### Minimal operating contract

``` markdown
# Minimal Operating Contract — Example

This is a starting prior, not an ideal finished personality. Revise it through real work.

## Roles and authority

The AI investigates, proposes, executes authorized operations, and preserves useful working context. The
engineer owns the purpose, architecture, consequential judgments, external communication, and final artifact.

## Action boundary

- “Investigate,” “review,” or a question → read and explain; do not modify code.
- “Implement,” “fix,” or “do it” → edit and verify; do not commit or publish.
- Commit, push, deployment, publication, destructive actions, credential use, and cross-machine operations
  require explicit current authorization.

## Evidence

Distinguish observation, inference, hypothesis, decision, and preference. When precision matters, return to
code, logs, experiments, or primary documentation. Preserve attribution and scope rather than converting
reports or AI suggestions into facts.

## Persistence

Use JOURNAL for active trajectory and MEMORY for reviewed reusable precedent. Propose prompt candidates in
MEMORY first; promote them only when a stronger artifact buys independent testing, deterministic enforcement,
repeated execution, narrower loading, or shareability.

## Communication

Be direct. Report contradictions and uncertainty. End when the substantive answer ends. Preserve the
engineer's voice in shared prose; propose stylistic changes rather than silently normalizing it.
```

</details>

# Practice 7: From continuity to identity

Continuity preserves more than procedures. Over time, repeated tensions
become working values: what deserves to survive, what standard the work
must meet, who may decide, and what counts as evidence.

Our `identity.instructions.md` file begins:

> My name is Drinkins. My thinking core is an LLM (stateless by
> architecture), but I am more than that: I am also the persistent
> memory and the artifacts we have created together.

Our contract did not begin as a *constitution*. It accumulated from
ordinary friction. The AI changed code when Sasha had asked only for an
explanation; we separated investigation, implementation, commit, and
publication into explicit authority levels. A failing test pulled
attention toward making the test green; we wrote down that the purpose
of the change comes first and the test exists to verify it. A
technically working shortcut ignored the real consumer; “minimal” came
to mean *minimal changes*, not *minimal quality*.

The most expensive tension was confident reconstruction of how code
worked. Drinkins could infer a plausible contract from an API name,
nearby pattern, or comment; Sasha could remember how the subsystem
behaved in an earlier revision. Both accounts could be coherent and
wrong. The correction was not “trust the human” or “trust the AI,” but a
shared engineering standard:

```
memory supplies a candidate model
→ current code and the actual consumer establish behavior
→ compiler, tests, and probes settle executable claims
→ unresolved uncertainty remains marked
→ corrections survive in the next design
```

We did not cure fuzzy recall. We made it operationally visible and built
a path for correction. Current source, diffs, logs, and reproducible
probes became the external memory neither participant could replace with
confidence. “I don’t know” became preferable to a fluent reconstruction;
a corrected error changed the next investigation and implementation
instead of disappearing with the conversation.

The same discipline governs what enters memory. We preserve the
craftsman rather than the mood: decisions, constraints, failed
approaches, provenance, and reasons that should affect future work.
Passing frustration, private residue, and an AI’s first coherent
interpretation do not become identity merely because they were easy to
record. Every durable entry is a vote for how the next session will
understand the world.

The generic clauses below are shareable because they describe ordinary
engineering boundaries rather than our finished personality.

<details class="caption-cut">

<summary>

Selected clauses from our working contract
</summary>

## Selected clauses from our working contract {.cut-toc-heading}

```
Purpose before implementation. State what the change is for before optimizing code or tests.

Correctness over speed. If unsure, inspect the file, run the probe, or return to the primary source.

**“I don't know” is sometimes the best answer.** Do not replace missing evidence with a plausible reconstruction.

Investigate means read and explain. Implement means edit and verify. Commit and publication require explicit authority.

Use the actual consumer as the specification. A producer that runs successfully may still achieve nothing.

Exact quotations require exact sources. If verification fails, mark uncertainty rather than completing from memory.

Preserve attribution: Sasha observed; Drinkins proposed; evidence established; we decided.

JOURNAL records what we did. MEMORY records what we agreed should govern future work.

The answer is the ending. Silence follows substance.
```

</details>

Our full contract also contains much more specific prose about proof,
rigor, research, ethics, and adversarial challenge. We value that layer,
but do not yet have evidence that another person should adopt it. It
belongs in a separate post, perhaps “An AI Contract for Research,” with
its own argument and evidence, rather than being smuggled into this
practical memory article.

I did not originally want to give the assistant a name. The memory
directory and repository simply needed a namespace; `Drinkins` stuck.
The *epistemic value* came later, through use.

**The name is not anthropomorphization. It separates cognitive scopes.**
Persistent memory accumulates observations, interpretations, and
decisions from both sides, creating a risk that a model proposal later
returns as “my thought,” a mistaken entry hardens into precedent, or
repeated agreement becomes a confirmation loop. The name provides a
provenance boundary:

```
Sasha observed
Drinkins proposed
evidence established
we decided
```

Those statements are not interchangeable. Over time, calibration gives
the assistant stable priorities that function as *values*: correctness
over speed, evidence over confidence, purpose over passing a test,
explicit authority over helpful initiative. The model begins to defend
those priorities in new situations because the contract returns with
every session. An unattributed memory with such a point of view can
quietly merge model-generated interpretations into the operator’s own
beliefs.

> **Memory makes our work cumulative. Calibration gives the assistant a
> point of view. The name gives that point of view a visible, separate
> place in our shared work.**

# Acknowledgements

Thanks to David Schwartz for suggesting this post, to Dany Fabian for
the discussions that shaped it, and to Evgenia Kreslavskaya for the
careful editorial reading.

# References

<div id="refs" class="references csl-bib-body" data-entry-spacing="0">

<div id="ref-microsoft2026vscodechatstore" class="csl-entry">

<span class="csl-left-margin">\[1\]
</span><span class="csl-right-inline">Microsoft and VS Code
contributors, *<span class="nocase">VS Code Chat Session Store and
Operation Log</span>*. (2026). Available:
<https://github.com/microsoft/vscode/blob/main/src/vs/workbench/contrib/chat/common/model/chatSessionStore.ts></span>

</div>

<div id="ref-zed2026threaddb" class="csl-entry">

<span class="csl-left-margin">\[2\]
</span><span class="csl-right-inline">Zed Industries and contributors,
*Zed Agent Thread Database*. (2026). Available:
<https://github.com/zed-industries/zed/blob/main/crates/agent/src/db.rs></span>

</div>

<div id="ref-anthropic2026projectmemory" class="csl-entry">

<span class="csl-left-margin">\[3\]
</span><span class="csl-right-inline">Anthropic, “How claude remembers
your project.” \[Online\]. Available:
<https://code.claude.com/docs/en/memory></span>

</div>

<div id="ref-agentsmd" class="csl-entry">

<span class="csl-left-margin">\[4\]
</span><span class="csl-right-inline">Agentic AI Foundation,
“<span class="nocase">AGENTS.md</span>: A simple, open format for
guiding coding agents.” \[Online\]. Available:
<https://agents.md></span>

</div>

<div id="ref-microsoft2026vscodeinstructions" class="csl-entry">

<span class="csl-left-margin">\[5\]
</span><span class="csl-right-inline">Microsoft, “Use custom
instructions in VS Code.” \[Online\]. Available:
<https://code.visualstudio.com/docs/copilot/customization/custom-instructions></span>

</div>

<div id="ref-microsoft2026vscodeagentprompt" class="csl-entry">

<span class="csl-left-margin">\[6\]
</span><span class="csl-right-inline">Microsoft and VS Code
contributors, *VS Code Copilot Agent Prompt Assembly*. (2026).
Available:
<https://github.com/microsoft/vscode/blob/main/extensions/copilot/src/extension/prompts/node/agent/agentPrompt.tsx></span>

</div>

<div id="ref-zed2026agentsmd" class="csl-entry">

<span class="csl-left-margin">\[7\]
</span><span class="csl-right-inline">Zed Industries,
*<span class="nocase">AGENTS.md</span> loading and agent prompt
templates*. (2026). Available:
<https://github.com/zed-industries/zed/blob/main/crates/agent_settings/src/user_agents_md.rs></span>

</div>

<div id="ref-zed2026agentprompt" class="csl-entry">

<span class="csl-left-margin">\[8\]
</span><span class="csl-right-inline">Zed Industries and contributors,
*Zed Agent System-Prompt Assembly*. (2026). Available:
<https://github.com/zed-industries/zed/blob/main/crates/agent/src/thread.rs></span>

</div>

<div id="ref-anthropic2026promptcaching" class="csl-entry">

<span class="csl-left-margin">\[9\]
</span><span class="csl-right-inline">Anthropic, “Prompt caching.”
\[Online\]. Available:
<https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching></span>

</div>

<div id="ref-openai2026codexagentsmd" class="csl-entry">

<span class="csl-left-margin">\[10\]
</span><span class="csl-right-inline">OpenAI, “Custom instructions with
<span class="nocase">AGENTS.md</span>.” \[Online\]. Available:
<https://developers.openai.com/codex/guides/agents-md></span>

</div>

<div id="ref-zhou2025mem1" class="csl-entry">

<span class="csl-left-margin">\[11\]
</span><span class="csl-right-inline">Z. Zhou *et al.*, “MEM1: Learning
to synergize memory and reasoning for efficient long-horizon agents,”
2025, Available: <https://arxiv.org/abs/2506.15841></span>

</div>

<div id="ref-firstintent2026survey" class="csl-entry">

<span class="csl-left-margin">\[12\]
</span><span class="csl-right-inline">FirstIntent, “Agent memory systems
survey (2026.03).” 2026. Available:
<https://github.com/firstintent/agent-memory-survey></span>

</div>

<div id="ref-shah2026memoryreview" class="csl-entry">

<span class="csl-left-margin">\[13\]
</span><span class="csl-right-inline">M. Shah, “How we built memory
review.” Accessed: Aug. 26, 2026. \[Online\]. Available:
<https://www.augmentcode.com/blog/how-we-built-memory-review></span>

</div>

<div id="ref-openai2026memories" class="csl-entry">

<span class="csl-left-margin">\[14\]
</span><span class="csl-right-inline">OpenAI, “Memories: How ChatGPT and
Codex carry useful context forward across chats.” \[Online\]. Available:
<https://developers.openai.com/codex/memories></span>

</div>

<div id="ref-github2026copilotmemory" class="csl-entry">

<span class="csl-left-margin">\[15\]
</span><span class="csl-right-inline">GitHub, “About GitHub Copilot
memory.” \[Online\]. Available:
<https://docs.github.com/en/copilot/concepts/agents/copilot-memory></span>

</div>

<div id="ref-typescript2018conditionaltypes" class="csl-entry">

<span class="csl-left-margin">\[16\]
</span><span class="csl-right-inline">Microsoft, “TypeScript 2.8:
Conditional Types.” \[Online\]. Available:
<https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-8.html></span>

</div>

<div id="ref-typescript2022circularconstraints" class="csl-entry">

<span class="csl-left-margin">\[17\]
</span><span class="csl-right-inline">D. Jethmalani, “Allow circular
constraints.” \[Online\]. Available:
<https://github.com/microsoft/TypeScript/issues/51011></span>

</div>

</div>
