# Seticompute

Seticompute for AI Video: a SETI@home-style network where many people pool
their graphics cards to render the shots of an AI film from a script or
storyboard, then gather the finished shots into one movie. Torrents in
reverse: instead of many peers sharing pieces of a finished file, many peers
*make* the pieces of a work that does not exist yet.

## Defensive publication (prior art)

The full design is published here as a **defensive publication**:
[Swarm Render: Distributed AI Video Network](docs/swarm-render-spec.md),
first published **November 22, 2026** by **HAPPY**
([@happyendings59](https://github.com/happyendings59)).

It is published so the idea stays free for everyone. It is intended as prior
art that prevents anyone from later patenting or claiming exclusive
ownership of this system, its methods, or the variations it lists.

**Companion: [LLM Swarm](docs/llm-swarm-spec.md)** (first published
November 24, 2026). The same "torrent computing" idea for text: writing
jobs are cut into pieces and routed across many AI providers' official
APIs, free tiers first, with automatic format checks, provenance from the
API itself, a Claude connector (MCP server) for the editor, and idle time
jobs that keep an offline mirror of open knowledge such as Wikipedia
dumps.

## Want to help? Help wanted

The design is published. The software is not built yet. If you want to
help get it written (open source), deployed and running, start with
[CONTRIBUTING.md](CONTRIBUTING.md). It lists the roles needed, the first
milestone (one worker renders one job end to end), and how to get in
touch: open an issue using the **"I want to help"** template.

## Licenses

- **Design documents** (`docs/`): [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
  Use and build on them freely, including commercially, with credit.
- **Code** (when added): GNU AGPL v3, see [`LICENSE`](LICENSE).
