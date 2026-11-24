# LLM Swarm: Routing Writing Work Across Many AI Providers

**Defensive publication (prior art).** First published November 24, 2026.
Author: **HAPPY** ([@happyendings59](https://github.com/happyendings59)).
Companion to [Swarm Render: Distributed AI Video Network](swarm-render-spec.md).

> **Notice.** This document is published openly so that the ideas in it are
> public, dated and free for anyone to use. It is intended as prior art: a
> public disclosure that prevents any person or company from later patenting,
> or claiming exclusive ownership of, the system and methods described here,
> including the variations listed in the final section. Nothing in this
> document is offered for patent protection by the author.
>
> The date of first publication is recorded by this repository's commit
> history and by any archived copies (see the README).

**Origin.** The concept is the author's own: "torrent computing" for
language models. Big writing jobs (storyboards, scene plans, blog drafts)
are cut into pieces, and each piece goes to whichever AI service has free
or cheap capacity right now. When one service runs out of its daily free
use, the next piece goes to the next service. A Claude connector lets
Claude act as the editor that hands out the pieces and checks what comes
back. The wording of this write-up was drafted with an AI assistant
(Claude, by Anthropic) from the author's idea.

**Author's view.** Same as Swarm Render: this design is meant to stay
open, anyone may build and run it, and nobody should hold exclusive rights
over it.

**License.** This document is licensed under
[Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/).
Code in this repository is licensed separately under the GNU AGPL v3
(see `LICENSE`).

---

## Overview

Swarm Render pools graphics cards to render video shots. LLM Swarm does
the same for text: it pools **language model providers** (cloud APIs with
free or low cost tiers, plus models you run yourself) and hands each one
a piece of a larger writing job.

**Problem.** The author's film slate (The Beetle, then a list of 305
public domain books) needs a scene plan and storyboard for every episode.
That is thousands of small, rule bound writing tasks. Doing them all in
one premium chat burns its usage limit in an hour. Pasting them by hand
into several free chat apps works, but it is slow, nobody records which
model wrote what, and the format rules get broken.

**Goals**

- Cut a writing job into pieces with stable IDs, and track every piece
  from queued to approved.

- Send each piece to the cheapest provider that can do it well and still
  has quota left today, and move to the next provider when one runs out.

- Check every result against the project's written rules before a human
  or Claude ever sees it.

- Record, for every piece, which provider and model wrote it, taken from
  the provider's own API response, not from what the model says about
  itself.

- Resume after a crash, a reboot or a quota reset without redoing
  finished pieces.

- Expose the whole thing to Claude as a **connector** (an MCP server), so
  Claude plans and reviews while the bulk writing runs on free tiers.

**Non-goals**

- Splitting one model's inference across machines (same rule as Swarm
  Render: whole pieces only).

- Training models.

- Getting more free use than a provider offers. See the next section.

## Fair use: official APIs only (hard rule)

This is a scheduler for capacity you are already allowed to use. It is
not a way around anyone's limits.

- **Official APIs only.** Every provider is reached through its published
  developer API with your own API key (for example the Gemini API,
  DeepSeek API, OpenRouter, Groq, Mistral, or an OpenAI compatible
  endpoint). The system never scripts, scrapes or automates a consumer
  chat website or app (such as the Gemini web app, ChatGPT, or duck.ai).
  Those products are meant for a person typing, and automating them is
  usually against their terms.

- **One account per person per provider.** No making extra accounts to
  multiply a free tier, no rotating keys or IP addresses to dodge a rate
  limit.

- **Stop when told to stop.** A rate limit reply (HTTP 429) or a quota
  error takes that provider out of rotation until its stated reset time.
  The router honours `Retry-After` and the provider's rate limit headers.

- **Read the terms per provider.** The provider registry records each
  provider's terms URL, whether its free tier may use your inputs to
  improve its products, and any content or commercial use limits. A
  human approves each provider before it joins the rotation.

- **Private stays private.** Pieces marked private never go to a
  provider whose free tier may keep or train on inputs. They go to a paid
  tier with a no training term, or to a model you run yourself.

## How it fits Swarm Render

LLM Swarm is a new **job kind** on the same coordinator, not a second
system. The coordinator, queue, leases, retries, registry, dashboard and
review flow from the Swarm Render spec are reused.

| Swarm Render                       | LLM Swarm                                                        |
|------------------------------------|------------------------------------------------------------------|
| Job = one shot                     | Piece = one block of text work (a scene, a shot list, a section) |
| Worker = a GPU running ComfyUI     | Worker = a provider adapter, a local model, or a volunteer GPU   |
| Approved workflow registry         | Approved prompt template registry                                |
| Approved model registry (SHA-256)  | Approved provider registry (terms, quotas, data use flag)        |
| Match by VRAM and cached models    | Match by context length, features, quota left, privacy class     |
| Review takes in the gallery        | QA checks, then Claude or human review                           |
| Export approved takes              | Assemble approved pieces into one document                       |

Three kinds of LLM worker:

1. **API workers.** A small adapter per provider that runs inside the
   coordinator (or on any machine you own). It holds that provider's API
   key, calls the official API, and reports usage. It "claims" pieces
   exactly like a GPU worker claims jobs.

2. **Local workers.** A model you run yourself through a local server
   (for example Ollama or llama.cpp). No quota, no data leaves the
   machine, slower and smaller. See "Keeping local models separate".

3. **Volunteer GPU workers.** Swarm Render workers that also accept text
   pieces with an approved open weight model (the "non video AI
   workloads" variation of the Swarm Render spec). Public pieces only.

### Keeping local models separate

A person may already run a local model server for another purpose, such
as a private legal assistant. LLM Swarm must never touch it.

- The local worker uses **its own** model server instance: its own
  install folder, its own model folder, its own port (not the default
  Ollama port 11434), and ideally its own operating system user.

- The provider registry stores the exact base URL of the swarm's local
  server, and the router refuses any local URL not in the registry.

- No shared prompts, logs, caches or vector stores between the two.

## Pieces

A piece is the smallest unit of work that one model call can finish and
one check can judge.

**Example.** "Write the stage 1 scene plan for The Beetle E03" becomes
one piece per scene block. Each piece carries the master prompt, the
book text for that block, and the output schema.

**Stable piece IDs.** `<project>/<document>/<block>@v<n>`, for example
`beetle/E03-scene-plan/B07@v1`. The ID never changes when the piece is
retried or moved to another provider. A new version (`@v2`) is made only
when the input or template changes. Each piece also stores a SHA-256 hash
of its fully rendered prompt, so the same prompt is never paid for twice.

**A piece as an API worker receives it**

```json
{
  "piece_id": "beetle/E03-scene-plan/B07@v1",
  "batch_id": "beetle-E03-stage1",
  "template": {"id": "storyboard_stage1_scene_plan", "version": 3},
  "privacy": "public",
  "needs": {"min_context_tokens": 32000, "json_output": true},
  "max_output_tokens": 4000,
  "prompt_sha256": "9f2c...",
  "inputs": {"block_text_ref": "beetle/source/ch03-part2.txt"},
  "lease_seconds": 300
}
```

## The router

For every queued piece the router picks a provider in four steps.

1. **Eligible.** The provider is approved, enabled and not cooling down
   after a limit; it supports the piece's needs (context length, JSON or
   structured output, language); its data use flag allows the piece's
   privacy class.

2. **Has room.** Its quota ledger shows enough requests and tokens left
   today, this minute, and in any other window the provider publishes.

3. **Ranked.** Among eligible providers with room, sort by: price for
   this piece (free first), then quality score for this template (from
   past QA pass rates), then speed. A batch may pin a preferred order,
   for example "DeepSeek first", matching how the author already works.

4. **Spread.** No single provider gets more than a set share of one
   batch, so one model's habits do not shape the whole document, and so a
   comparison across models comes for free.

**Quota ledger.** Limits are never hard coded in the code. Each provider
entry lists its windows (per minute, per day, tokens per day) as the
admin copies them from the provider's published limits page, with the
date checked. The ledger counts every call and its token usage from the
API response, and where a provider returns rate limit headers, those
override the local count. Daily windows reset at the provider's stated
reset time and time zone.

**Failover.** A piece that hits a limit, times out or fails QA goes back
to the queue with that provider excluded for that piece, and the next
eligible provider takes it. After 3 failed attempts on 3 different
providers the piece is marked failed and shown to the editor.

**Cost cap.** Each batch has a spending cap (zero for "free tiers only").
Paid providers are skipped once the cap would be passed.

## QA pass

Every result goes through two checks before anyone reads it.

**1. Automatic checks** (cheap, no model needed), defined per template
from the project's own rule files (for the film slate, the rules in the
private `aistoryboardvideorules` repository; for the blog, the master
prompt rules):

- The output parses (JSON, or the exact labeled sections asked for).
- Required fields are present and non empty; counts and durations are in
  range (for example shot lengths add up to the scene length).
- Banned characters or phrases are absent (for example the blog's no
  dash rule).
- Every citation marker points to a listed source, and every URL is a
  plain publisher URL, not a search or redirect link.
- Output is not a near copy of another piece (catches a model repeating
  itself).

**2. Editor review.** Results that pass go to the editor, either Claude
through the connector or a human on the dashboard: approve, fix, or send
back with a note. A send back requeues the piece with the note added and
the last provider excluded.

QA results feed the provider's quality score for that template, so the
router learns which free model is good at which job.

## Provenance

Every call writes one line to an append only log, and each line includes
the hash of the line before it, so edits to history are detectable.

| Field                | Source                                                        |
|----------------------|---------------------------------------------------------------|
| piece_id, attempt    | Coordinator                                                   |
| provider             | Provider registry entry used for the call                     |
| model                | The `model` field in the provider's API response              |
| request_id           | The provider's response or request ID, where it gives one     |
| started, finished    | Coordinator clock (UTC, to the second)                        |
| tokens in, out       | Usage block in the API response                               |
| prompt_sha256        | Hash of the exact prompt sent                                 |
| output_sha256        | Hash of the exact text received                               |
| qa_result, reviewer  | QA pass and editor decision                                   |

The model name comes from the API, so a credit line such as "Prepared by
DeepSeek (deepseek-chat via the DeepSeek API)" is backed by a record, not
by the model's own guess about itself. The assembled document can carry a
generated credits table listing every provider and model that wrote an
approved piece.

## Resume

All state lives in the coordinator database. On restart:

- Pieces that were leased and whose lease has run out go back to the
  queue.
- Approved pieces are never rerun; a matching `prompt_sha256` with an
  approved result is reused.
- Providers cooling down stay cooling until their stored reset time.

## The Claude connector (MCP server)

The coordinator ships an MCP server so Claude can drive the swarm as a
tool. Claude does the judgment work (splitting a job, reviewing,
fixing, assembling); the swarm does the bulk drafting on other providers'
free or cheap tiers. That keeps premium Claude usage for editing, which
is where it is worth the most.

**What it does not do.** The connector does not move Claude's own usage
anywhere, and it never logs into any chat product. It only calls the
official APIs that the admin configured with their own keys. API keys
stay inside the connector and are never shown to Claude.

**Transport.** Local stdio for Claude Code and the Claude desktop app
(the connector runs on your own machine). Remote HTTP with OAuth for use
as a custom connector in Claude on the web, Phase 2.

**Tools**

| Tool              | Input                                                             | Returns                                                       |
|-------------------|-------------------------------------------------------------------|---------------------------------------------------------------|
| `list_templates`  | none                                                              | Approved prompt templates and their rule sets                 |
| `submit_pieces`   | batch name, template id, privacy, list of pieces (id + inputs), cost cap, optional provider order | Batch id, piece ids, queued count                 |
| `check_quota`     | optional provider                                                 | Per provider: enabled, cooling until, requests and tokens left in each window, data use flag |
| `batch_status`    | batch id                                                          | Counts by state; failed pieces with last error                |
| `get_result`      | piece id                                                          | Latest output, QA report, provider and model, attempt number  |
| `review_result`   | piece id, approve or reject, note, optional fixed text            | New state; a rejected piece is requeued with the note         |
| `get_provenance`  | piece id or batch id                                              | Provenance lines for the pieces                               |
| `assemble`        | batch id, output format                                           | The approved pieces joined in order, plus a credits table     |
| `cancel`          | piece id or batch id                                              | New state                                                     |

**Resources.** Each template's rules are exposed as MCP resources
(`rules://<template>`), so Claude reads the same rules the QA pass
enforces.

**A typical session.** Claude reads the rules, splits an episode into 12
scene blocks, calls `submit_pieces`, checks `batch_status` later, pulls
each result with `get_result`, fixes small problems itself and approves
(`review_result` with fixed text), rejects anything badly wrong, then
calls `assemble` and saves the document to the project.

## Idle jobs: keeping knowledge online

When no foreground work is queued, workers may run **idle jobs** at the
lowest priority. They stop within 10 s when real work arrives (same rule
as FR-16 in the Swarm Render spec).

**Knowledge mirror.** The first idle job class keeps an offline copy of
open reference works, so the knowledge stays reachable if a site goes
down or is blocked.

- **Source: official dumps only.** Wikipedia is mirrored from the
  Wikimedia dumps at `dumps.wikimedia.org` (or the Kiwix ZIM files built
  from them), never by crawling the live site, which Wikimedia asks
  people not to do. Each download is checked against the checksums the
  dump site publishes.

- **License kept.** Wikipedia text is CC BY-SA. The mirror keeps the
  license, the attribution and the link back to each article's history,
  and any page served from the mirror says so.

- **Shared, not duplicated.** Workers that opt in hold pieces of the
  mirror (by dump file), torrent style, with a manifest listing which
  worker holds which file and its hash. Any one full copy can rebuild the
  rest.

- **Optional index.** A second idle job builds a search index or
  embeddings over the mirror on a local model, so it can be searched
  offline. That index is also public and carries the same license notes.

Other idle job ideas: re-running QA on old approved pieces when rules
change, and translating approved public text into other languages for
subtitle files.

## Data model additions

| Table         | Key columns                                                                                                      |
|---------------|------------------------------------------------------------------------------------------------------------------|
| providers     | id, kind (api, local, volunteer), base URL, model ids allowed, terms URL, data use flag, limits JSON with date checked, price per token, enabled, cooling_until |
| quota_ledger  | provider id, window, window start, requests used, tokens used, last header snapshot                              |
| templates     | id, version, prompt text, output schema, QA rule set, approved                                                   |
| pieces        | id, batch id, template id+version, inputs, privacy, needs, status, attempts, excluded providers, prompt_sha256, lease_until |
| piece_results | piece id, attempt, provider id, model, output, output_sha256, tokens, QA report, review, reviewer note           |
| provenance    | sequence, prev_hash, line JSON, line_hash                                                                        |
| idle_jobs     | id, class (mirror, index), source URL, checksum, holder worker ids, status                                       |

## Tech stack and repo layout

Same stack as Swarm Render (Python, FastAPI, SQLite then Postgres). Each
provider adapter uses the provider's official SDK or its documented
OpenAI compatible HTTP endpoint. The connector uses the official MCP
Python SDK.

```text
swarm-render/
  coordinator/            # shared with Swarm Render
  llm/
    router.py             # eligibility, ranking, spread, cost cap
    quota.py              # ledger, rate limit headers, reset times
    adapters/             # one file per provider, official APIs only
    qa/                   # automatic checks, one rule set per template
    provenance.py         # hash chained log
    assemble.py
  connector/
    server.py             # MCP server: tools and rules resources
  templates/              # approved prompt templates + rule sets
  idle/
    mirror.py             # dump download, checksum, manifest
  tests/
```

## Build plan

1. **L1 · One provider, one piece.** Coordinator stores a piece, one
   adapter calls one API, the result and its provenance line are saved.
   - Done when: one command submits a scene block and the output plus the
     API reported model name appear in the database.

2. **L2 · Quota ledger and failover.** Two or more adapters, limits from
   the registry, 429 handling, cooling, reset times.
   - Done when: with a provider's daily limit set to 3 in the registry,
     the 4th piece goes to the next provider on its own.

3. **L3 · QA pass.** Automatic checks from one real rule set (the
   storyboard stage 1 rules).
   - Done when: a result with a missing field is rejected and requeued
     without a human.

4. **L4 · Claude connector.** MCP server over stdio with the tools above.
   - Done when: Claude in Claude Code submits an episode's scene blocks,
     reviews them and assembles one document.

5. **L5 · Local worker and privacy classes.** A separate local model
   server, private pieces routed only there.
   - Done when: a private piece never appears in any cloud adapter's log.

6. **L6 · Idle knowledge mirror.** Dump download, checksum, manifest,
   opt in holders.
   - Done when: a worker with no queued work downloads and verifies one
     dump file and the manifest lists it.

## Risks and open questions

| Risk                                                      | Effect                                    | Plan                                                                  |
|-----------------------------------------------------------|-------------------------------------------|-----------------------------------------------------------------------|
| A provider changes or ends its free tier                  | Pieces stall or start costing money       | Limits and prices live in the registry with a date checked; cost cap  |
| Free tiers that may train on inputs                       | Unreleased work leaks into a model        | Data use flag; private pieces never go there                          |
| Different models write in different styles                | Patchy document                           | Strict templates, QA, spread rule, editor pass at assembly           |
| A model invents facts or sources                          | Wrong content in a published piece        | Citation checks in QA; human or Claude review before approval         |
| Someone points the router at a scraped chat site          | Terms violation, account bans             | Adapters are reviewed and approved; no browser automation in the code |
| Local worker touches another local system                 | Private data mixed in                     | Own instance, own port, URL allow list (see "Keeping local models separate") |

**Open questions**

- Which providers to approve first, and in what default order.
- Should QA rule sets be written as code, or as a small rules file that
  both the QA pass and the prompt template read?
- Phase 2: let other people share a batch's work across their own API
  keys (each paying or using their own free tier), credited on the
  leaderboard?

## Additional variations disclosed

The system above is one embodiment. The following variations are also
disclosed and are equally part of this publication:

- **Any text or code work.** The same piece queue, router and QA pass
  split and route any language work: articles, scripts, translations,
  subtitles, summaries, legal document review on private workers, code
  generation with tests as the automatic check, data extraction, and
  labeling.
- **Any provider mix.** Providers may be free tiers, paid APIs, local
  models, volunteer GPUs, rented GPUs or a marketplace, and the ranking
  may weigh price, speed, quality score, energy use, or a fixed order.
- **Model as editor.** Any capable model (not only Claude), or a human,
  or a chain of both, may act as the editor that splits jobs, reviews
  results and assembles the final work, through MCP or any other tool
  calling interface.
- **Cross checking.** The same piece may be sent to two or more
  providers on purpose, and the results compared, merged or voted on,
  with disagreements sent to the editor.
- **Learned routing.** QA pass rates, editor decisions and costs may
  train a routing model that predicts the best provider per template.
- **Signed provenance.** The provenance chain may be signed with a key,
  timestamped by a third party, or anchored to a public ledger, and
  published next to the finished work as a verifiable credits list.
- **Idle work of any kind.** Idle time may keep mirrors of any openly
  licensed collection (encyclopedias, public domain books, open
  courseware, public law), build indexes over them, or re-check old
  results, with license and attribution kept.
- **Combined swarm.** One coordinator runs text pieces and render jobs
  together, so a storyboard written by the LLM swarm feeds the shot
  queue of Swarm Render directly, from book to finished film.
