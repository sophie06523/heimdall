# Heimdall

<p align="center">
  <a href="https://www.npmjs.com/package/@arihantdeva/heimdall"><img alt="npm" src="https://img.shields.io/npm/v/%40arihantdeva%2Fheimdall"></a>
  <a href="https://github.com/ArihantDeva/heimdall/actions/workflows/ci.yml"><img alt="CI" src="https://img.shields.io/github/actions/workflow/status/ArihantDeva/heimdall/ci.yml?branch=main"></a>
  <a href="https://github.com/ArihantDeva/heimdall/stargazers"><img alt="GitHub stars" src="https://img.shields.io/github/stars/ArihantDeva/heimdall?style=social"></a>
  <img alt="license" src="https://img.shields.io/badge/license-MIT-blue">
</p>

**Your agent keeps rebuilding work you already did. Heimdall makes it stop.**

<p align="center">
  <a href="https://raw.githubusercontent.com/ArihantDeva/heimdall/main/docs/assets/demo.mp4"><img alt="Heimdall demo" src="https://raw.githubusercontent.com/ArihantDeva/heimdall/main/docs/assets/demo.png" width="720"></a>
</p>

Heimdall gives AI coding agents **persistent memory across every repository and project you work on** so the question *"did I already solve this in another project?"* gets answered by one verified search instead of twenty minutes of grep, `find`, and `ls` loops.

## The problem it solves

**1. Memory that doesn't live in one repository.** Every other memory tool is per-project. But your work isn't: the optimized functions you built in one project could be useful somewhere else. Heimdall indexes *everything you touch* into one semantic graph, so knowledge follows you across repositories, languages, and months.

**2. Orientation time, cut to seconds.** A fresh agent session burns dozens of bash commands just figuring out the lay of the land — `ls`, `grep`, re-reading files it read last week. Heimdall injects the relevant prior work into the session's first prompt and backs a single `kb_search` call: ranked, scoped, verified. Fewer commands, fewer tokens, faster first useful action.

**3. Zero token spend.** Memory maintenance is a local daemon: file watching, tree-sitter AST parsing, sqlite. Indexing a file costs **CPU only — never an LLM call**. Retrieval is hybrid ranked search (lexical + semantic + graph walk) over locally-computed embeddings. Your context window stays for your actual work. You can also use your GPU for up to 3.4x speeds.

**4. Retrieval you can act on.** Semantic memory tools return plausible matches, but Heimdall also verifies it at runtime.

- `STRONG` — the anchor is intact **and** the indexed text answers the query: the file is still a regular file whose size and mtime match its index card, and the query's tokens appear in the content that was indexed. Nothing is read at query time
- `WEAK` — plausible but unverified; the printed reason says which part failed (changed since indexing, no index card, tokens covering too little of the indexed text, or a path that is no longer a regular file)
- `REBUILT` — file moved; Heimdall found it and re-anchored automatically
- `STALE` / `REMOVED` — dead path, logged and pruned so it stops ranking

An agent acting on a dead path is worse than no answer. Ranked retrieval is trustworthy enough to act on. Gives the real picture, instead of the best guess.

## How a session changes

Without Heimdall:

```
$ grep -r "portfolio optimization" .            # wrong repo, 40s
$ find ~/work -name "*.py" | xargs grep -l optimizer   # 2 min
$ ls ~/work/... ; cat notes.md ; ...         # 15 commands later
```

With Heimdall:

```
$ kb_search "portfolio optimization jam optimizer"
   1. [STRONG] portfolio optimizer — ~/work/quant-bot/src — EV-optimizer entry point
   2. [STRONG] excel report builder — ~/work/reports/excel — 276-session tracking table
```

One call. Verified paths. Straight to work.

## How it compares

Architecture-level comparison of shipped defaults — not benchmark claims. "LLM extraction" etc. describe each project's default pipeline as documented; self-hosted or configured-differently deployments vary.

| | **Heimdall** | mem0 | Zep (Graphiti) | Letta (MemGPT) | LangMem / LangChain Memory | Vector-DB RAG | Claude Memory | cAST / grep |
|---|---|---|---|---|---|---|---|---|
| Scope | **All your repos, one graph** | per-app/per-user memories | per-user/session graph | per-agent | per-app/thread | per-corpus/index | per-conversation/account | per-repo |
| Runs on CPU only | **yes** | LLM+embeddings in the loop | LLM extraction for entities/edges | LLM-in-the-loop memory management | LLM extraction + embeddings | embeddings (GPU-friendly) | cloud | yes |
| Token cost of indexing | **zero** (tree-sitter + local embeddings) | LLM extraction per memory op | LLM calls per episode | LLM calls throughout | LLM extraction per write | embedding tokens only | LLM summarization | zero but manual |
| Trust verdicts on results | **STRONG / WEAK / REBUILT / STALE** | none | none | none | none | similarity score only | none | none |
| Self-healing (moved files re-anchored) | **yes** | no | no | no | no | no (stale chunks rank) | no | no |
| Convergent state (idempotent re-index) | **yes** (level-triggered reconciler) | append-oriented | event-sourced episodes | conversation-scoped | append-oriented | re-ingest to update | opaque | n/a |
| Reads private source locally | **yes, never leaves disk** | sent to extraction LLM | sent to extraction LLM | stays local w/ local models | sent to extraction LLM | local if self-hosted | cloud | yes |
| Harness integrations | pi, Claude Code, Codex, Cursor, Windsurf | SDK/API | SDK/API + Graphiti | SDK/API | LangChain-native | DIY per stack | Claude products | editor plugins |

### Where Heimdall beats mem0 and common RAG

By design, not by benchmark — these follow from the architecture:

1. **Zero-LLM indexing instead of extraction pipelines.** mem0, Zep, Letta, and LangMem all use LLMs to write facts: every remembered fact costs extraction tokens, adds latency, and means your code/notes are processed by a cloud provider unless you wire your own. Heimdall's ingest is tree-sitter plus local CPU embeddings. It cannot leak data, nor does it cost anything.

2. **Verified hits vs plausible hits.** RAG returns nearest neighbors with a similarity score; nothing checks that the chunk still exists, let alone that it answers the question. Heimdall re-verifies every result against the live filesystem at query time and labels it STRONG/WEAK/REBUILT/STALE. STRONG requires an index card that still agrees with the file on disk — its recorded size and mtime match what `stat` reports now, compared exactly (the value round-trips through SQLite unchanged, so there is no tolerance and no window) — and query coverage in the hit's own text, with the path excluded so a filename cannot stand in for content. The whole check is two stats per hit: it never opens, let alone reads, the file it judges. A hit with no card to check against is WEAK and says so; it does not get to claim verification it never performed. Agents can act on STRONG without a confirmation round-trip. One boundary is real and documented rather than claimed away: a same-size rewrite that also restores mtime is invisible to any stat, and closing that requires reading content — which is what `heimdall verify --deep` is for.

3. **Self-healing vs stale corpora.** In vector-RAG, a moved file leaves orphaned chunks ranking forever until someone re-runs ingestion. Heimdall's level-triggered reconciler converges: moved files re-anchor automatically (REBUILT), deletions retract exactly their own nodes, and re-indexing twice is identical to once.

4. **Cross-repo scope without a corpus pipeline.** Classic RAG needs you to define, chunk, and refresh a corpus per app. Heimdall watches working trees continuously — new repos join the graph on their own, and personal context (prompt logs, notes, now even email via `heimdall ingest-email`) lands in the same graph your code lives in.

5. **What they win back.** Fair's fair: mem0/Zep/Letta excel at conversational fact curation across chat products, multi-user serving, and hosted APIs; LLM extraction summarizes messy prose better than regexes. Heimdall is making the opposite bet, that a single developer's machine-wide workspace where the unit of memory is verified file-level knowledge, not chat utterances. 

Heimdall is the only one built for the real indie developer workflow: many repos, many months, one agent session at a time, on hardware you already own. Best for people with tons of side projects.

## FAQ

**Does my code leave my machine?** No. Indexing is tree-sitter parsing + local embeddings on CPU. Search runs against your local daemon. Nothing phones home.

**Do I need a GPU?** No. The embedding model (bge-m3) runs on Apple Silicon / any modern CPU.

**How is this different from grep?** Grep finds strings you already know exist. Heimdall answers "have I solved anything like this before?" across every project you've touched, ranked and verified against what's actually on disk right now.

**What if a file moves or gets deleted?** The reconciler notices on its next pass. Moved files are re-homed automatically (REBUILT verdict); deletions retract exactly their own nodes. A stale path never ranks again.

**Does it work with my agent?** One command wires it into pi, Claude Code, Codex, Cursor, or Windsurf. Anything that can run a CLI can use `search`/`insert` directly.

**Is it production-ready?** It runs daily on this author's machine across ~12,800 live nodes with a 166-test suite guarding the concurrency invariants. v0.2.0. LongMemEval benchmark harness is in `bench/` (in progress).

## Roadmap

- [x] v0.1 — packaged CLI, five harness adapters, trust verdicts
- [x] v0.2 — single-writer reconciler, depth-ladder indexing (symbols + call edges)
- [ ] LongMemEval-S score ≥ 0.90 published from `bench/` (baseline S 0.740 reproduced)
- [ ] LongMemEval-M full reader/judge run published (S subset is token-free recall only — not end-to-end)
- [x] 0.9.0 — graft backend auto-builds on `npm i -g` (postinstall) and `heimdall setup`; no separate `init --backend` flag
- [ ] Linux daemon packaging
- [ ] MCP server mode

## Contributing

PRs welcome — see [CONTRIBUTING.md](CONTRIBUTING.md). The concurrency invariants are tested; break them and the suite goes red before you do.

## Quickstart

```bash
npm i -g @arihantdeva/heimdall
heimdall init --harness claude-code   # or pi | codex | cursor | opencode | all
```

That's it for install + harness wiring (`init`, `insert` work immediately).

`npm i -g` auto-builds the graft backend during install when prerequisites
are present
(macOS — Xcode Command Line Tools + `brew install cmake pkg-config libyaml sqlite`;
Debian/Ubuntu — `apt install cmake pkg-config build-essential git libsqlite3-dev libyaml-dev`;
Fedora/Amazon Linux — `dnf install gcc gcc-c++ make cmake git pkgconf-pkg-config sqlite-devel libyaml-devel`;
`git` required — llama.cpp is cloned at first build);
first build takes a few minutes (log: `~/.heimdall/bootstrap.log`). If
prerequisites are missing, SETUP NEEDED prints and the install still
succeeds — run `heimdall setup` to build later. Opt out:
`HEIMDALL_NO_BUILD=1 npm i -g @arihantdeva/heimdall`. Defaults: model
`bge-m3`, accel auto (Metal on Apple Silicon, CUDA when `nvidia-smi`
present, else CPU), threads = physical cores, 2 instances.

`heimdall setup` also builds/installs `graftd` when none is working,
downloads the model if needed, generates `~/.graft/config.yaml`, and
installs the launchd daemon:

```bash
heimdall setup                         # detect + config + model + daemon
heimdall setup --model bge-small-en-v1.5   # smaller model (~36MB, low RAM)
heimdall setup --detect-only           # just print the hardware profile
heimdall doctor                        # should print HEALTHY
heimdall search "excel tracker portfolio optimization"
```

Full flags + hardware matrix + model catalog: [docs/setup.md](docs/setup.md).
Prefer hand-rolling? Copy
`config/heimdall.yaml.example` to `~/.graft/config.yaml` and edit
(`/PATH/TO/` placeholders). On macOS the backend runs as the launchd job
`com.graft.daemon` (setup writes the plist; template:
`launchd/com.heimdall.backend.plist.example`).

### Alternative backend: Mnemosyne

Ranked search also supports [mnemosyne-oss](https://github.com/mnemosyne-oss/mnemosyne) (SQLite-backed agent memory) as the retrieval backend. Graft remains the zero-config default; select mnemosyne per-invocation or persistently:

```bash
pip install mnemosyne-memory            # or: uv pip install mnemosyne-memory
HEIMDALL_BACKEND=mnemosyne heimdall search "preferences"        # one-off
# persistent:
python3 -c 'import json,pathlib; p=pathlib.Path.home()/".heimdall/config.json"; c=json.loads(p.read_text()) if p.exists() else {}; c["backend"]="mnemosyne"; p.write_text(json.dumps(c,indent=2))'
heimdall search "preferences"
```

Resolution order: `$HEIMDALL_BACKEND` > `~/.heimdall/config.json` `backend` key > `graft`. Pin a non-PATH binary with `MNEMOSYNE=/path/to/mnemosyne`. Mnemosyne results flow through the same ranked/verified output as graft hits.

## Demo

![Heimdall demo](assets/demo.gif)

Real session output (abridged):

```
$ kb_search "portfolio optimization jam optimizer"
== retrieve (hybrid ranked): portfolio optimization jam optimizer
   1. [STRONG] cov83%  portfolio optimizer — ~/work/quant-bot/src
      EV-optimizer entry point, edited 2026-08-14
   2. [WEAK]   excel report builder — ~/work/reports/excel
```

Every hit carries a trust verdict computed against the live filesystem — not a cached embedding score. Nothing is STRONG unless an index card says the file is unchanged (`~/.heimdall/global.db`: recorded size and mtime still match what `stat` reports, compared exactly — the value round-trips bit-for-bit through SQLite, so this is deterministic, not a tolerance), and either the query tokens overlap the hit's text or the semantic layer matched it on top of that verified identity. A similarity score alone never makes a hit STRONG. A hit that cannot meet the bar says which part failed, on its own line: `file changed since it was indexed`, `no index card for this path — content not verified`, `indexed content intact, but query tokens cover only 20% of it`, `anchor is gone from disk`, `path is no longer a regular file`. The ceiling is honest and specific: a same-size rewrite that also restores mtime leaves metadata *byte-identical* while content differs, so no stat-based check can see it — and the card's `sha1` cannot close it for free, because it hashes `read_preview(path)`, i.e. content that must be read. That boundary is owned by `heimdall verify --deep`, which re-hashes the file and reports the drift, so the exposure lasts until that audit runs.

## Design history

v0.1.0 hooks inferred graph mutations by regex-parsing bash commands and writing the graph from every hook process. It collapsed: writes it didn't recognize were invisible, concurrent hook processes raced delete+insert, and a misparse wrote wrong data as fact. v0.2.0 replaced all of it with a **single-writer, level-triggered reconciler**: nothing ever tells the graph *what* changed, only *that a path might have*. The reconciler reads the file from disk and makes the graph match.

The trust verdict layer came from the same lesson one level up: even a perfect graph lies if its anchors rot (directories get reorganized aggressively). So search results are re-verified against the filesystem at query time, with self-healing (rehome) for moved files.

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=ArihantDeva/heimdall&type=Date)](https://star-history.com/#ArihantDeva/heimdall&Date)

If Heimdall saved you a rebuild, a star helps other agents' humans find it.


## Architecture / how it works

```
                 ┌─────────────────────────── harness integrations ───────────────────────────┐
                 │  Pi: extensions/kb-*.ts          Claude Code: PostToolUse hook             │
                 │  (kb-tools, kb-autosync, kb-orient, kb-search-guard)   Codex/Cursor/Windsurf │
                 └───────────────┬──────────────────────────────┬─────────────────────────────┘
                                 │ only ever appends hints      │
                                 ▼                              ▼
                     ~/.heimdall/hints.jsonl        heimdall CLI (bin/heimdall.js)
                                 │                     init / insert / hint / verify / depth
                                 ▼                                    │
   ┌───────────────────────── THE SINGLE WRITER (holds O_EXCL lock) ───┘
   │  bin/heimdall-reconciler.mjs  (daemon: fs watch + hint ingest + drain loop + audit timer)
   │        │ drain()  reads hint queue → reconcilePath() per path
   │        ▼
   │  bin/lib/reconcile.mjs  — level-triggered convergence
   │        reads file from disk → desiredState() → compares hash/depth
   │        → sink projection (graft insert/delete) BEFORE journal commit
   │        → journal.commit() in one transaction, generation-guarded (ABA)
   │        ▼
   │  bin/lib/journal.mjs  — AUTHORITATIVE index (~/.heimdall/journal.db, node:sqlite, WAL)
   │        paths (hash/size/mtime/depth/generation/state) · owned_nodes · owned_edges
   │        pending_edges (order-independent cross-file edges) · queue (dedup)
   │        ▼
   └──────────►  bin/lib/sink.mjs  — projection target (pluggable backend)
                 GraftSink (graft CLI) | MemorySink (tests, --dry-run)
                 ▼
        ~/.graft daemon (vendor/graft, built from source) — sqlite + bge-m3 embeddings + graph edges

   SEARCH PATH (read side, no lock):
   kb-search.sh → graft ask per repo (per-repo code graphs, --json) + embed-index.py query (global semantic)
                → merged, deduped → verdict pass in-process (STRONG / WEAK / NOPATH)
                → optional graft explore (graph walk of related work) → printed, ranked
```

**Core invariants** (each has a test):

- **Level-triggered, not event-driven.** Hooks/watchers/scripts can only say "look at this path". They are never believed about *what* happened. A missed, duplicated, or flatly wrong hint costs one `stat` — nothing more. This is what makes 40 concurrent agents editing one file harmless: they produce N cheap appends that collapse to one queue row (primary-key dedup).
- **Single writer by construction.** Every graph/journal mutation passes through `Lock` — an `O_EXCL` create with stale-PID liveness reclamation (Node has no portable flock). Two writers cannot exist, so the old delete-then-insert race (file briefly vanished from the graph) is unrepresentable. The daemon holds the lock for its lifetime; one-shot `heimdall reconcile` takes it too and serializes against the daemon.
- **Exact ownership.** Every node and edge belongs to exactly one path. A commit deletes and re-inserts all of that path's rows in one transaction — a deleted symbol cannot survive as a ghost, and one path's retraction never touches another's rows.
- **Content hash is the oracle.** Desired state = pure function of (bytes, depth). Reconciling twice is byte-identical to reconciling once. The `generation` counter is the ABA guard: a commit whose generation moved since the reconcile started is rejected (action `stale`) and the path re-queued, so a late reconcile can never resurrect a node a newer one deleted.
- **Order independence.** A cross-file edge to a not-yet-indexed symbol parks as a pending edge, owned by the source file, and resolves in both directions once the target file reconciles. The final graph does not depend on which file was reconciled first.
- **Audit as backstop.** `heimdall verify` compares the journal against the filesystem (stat-only: size/mtime, plus a depth-upgrade check) and exits 1 on drift; `--deep` re-hashes everything and catches even a same-size/same-mtime rewrite. This is the accuracy claim as a CI-able command. The daemon runs the same audit on a timer (default 15 min) and re-queues drift.
- **Journa