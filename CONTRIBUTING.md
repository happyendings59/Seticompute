# Contributing to Seticompute (Swarm Render)

**Help wanted.** The design is done and published. The software is not
written yet. If you want a free, open network where anyone can lend a
graphics card to render AI film shots, help build it.

The goal is simple: get a working version built, open source, and
deployed, so the author can start running real render jobs on it and
others can join.

Read the design first: [Swarm Render spec](docs/swarm-render-spec.md).
It has the architecture, the data model, the worker API, the tech stack,
the repo layout and the build plan (milestones M1 to M7).

The text side is in the [LLM Swarm spec](docs/llm-swarm-spec.md), with its
own build plan (L1 to L6). Its hard rule: official provider APIs only,
never scripting consumer chat websites or dodging rate limits.

## First milestone: M1, one worker and one job

Everything starts here. M1 is small on purpose.

- A coordinator (Python, FastAPI, SQLite, a local results folder).
- A worker agent that claims a job and drives a local ComfyUI through its
  `/prompt` API.
- A command that submits one test clip.

**Done when:** one command submits a test job, one machine renders it,
and the finished `.mp4` lands in the coordinator's results folder.

After M1, the spec lists M2 (workflow templates and CSV shot lists), M3
(many workers, leases, retries), M4 (locked worker container), M5
(dashboard and review), M6 (accounts, credits, public jobs) and M7
(hardening).

## Who we need

You do not need to do everything. Pick one.

| Role | What you would do |
|------|-------------------|
| Python backend | Coordinator API, database tables, job scheduler, leases and retries (FastAPI, SQLAlchemy) |
| Worker and ComfyUI | The worker agent loop, the ComfyUI API client, model download and hash checks |
| Packaging | Docker image for workers (NVIDIA Container Toolkit on Linux, WSL2 on Windows), docker compose for the coordinator |
| Dashboard | Server rendered pages with htmx: queue, workers, gallery, approve or reject |
| Testers with GPUs | Run the worker on your own card (8 GB VRAM and up) and report what breaks, especially on Windows |
| Security review | Locked down workers, node allow lists, tokens, abuse handling (spec sections NFR and M7) |
| Docs | Install guides for contributors and for people lending a GPU |

## How to get involved

All contact goes through this repository. Please do not look for personal
emails or phone numbers.

1. **Say hello.** Open an issue with the **"I want to help"** template.
   Say which role you want and what hardware you have.
2. **Pick a task.** Issues labeled `good first issue` or `M1` are the
   place to start. Comment on one before you begin so two people do not
   build the same thing.
3. **Send a pull request.** Keep it small and focused on one task. Match
   the layout in the spec (`coordinator/`, `worker/`, `shared/`,
   `workflows/`, `tools/`, `tests/`). Include a short note on how you
   tested it.
4. **Questions about the design** go in an issue too. If the spec is
   wrong or unclear, say so. It can change.

## Licenses for what you contribute

- **Code** is licensed under the GNU AGPL v3 (see [`LICENSE`](LICENSE)).
  Anyone who runs a modified version as a public service must share
  their changes. That keeps the network open.
- **Design documents** in `docs/` are CC BY 4.0.

By opening a pull request, you agree that your contribution is licensed
under the same license as the files it adds or changes, and that you
have the right to contribute it.

## Ground rules

- The design is a defensive publication. Nobody, including contributors,
  should try to patent or claim exclusive rights to it or its variations.
- Only use AI models whose licenses allow the use (the spec prefers
  Apache 2.0 or similar for public jobs). Do not commit model weights to
  the repo.
- Do not commit secrets, tokens or anyone's private render jobs.
- Be kind and stay on the work.
