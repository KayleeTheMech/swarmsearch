# Swarmsearch: Full-Auto Implementation Plan

Plan for building everything in `README.md` with autonomous coding agents and no token budget. Token spend is not the constraint. The constraints are verification, merge conflicts, and design decisions that cannot be verified by tests.

## Principles

- Contracts first. Interfaces (protobuf/OpenAPI, Rust traits, on-disk formats) are frozen before parallel work starts.
- Every agent task ends with a machine-checkable gate: `cargo test`, `cargo clippy -D warnings`, `cargo deny`, property tests, benchmarks, or an e2e run.
- One agent, one worktree, one crate or directory. Merges go through an integration agent, never agent-to-agent.
- Independent verification: the agent that writes code never grades it. A separate reviewer agent gets the diff only, not the author's reasoning.
- Open design questions (below) are resolved up front by a decision log, not mid-implementation.

## Phase -1: Manual README rework (human + interactive consultation)

Yvonne rewrites `README.md` by hand, with Claude as a sounding board (questions, critique, counter-arguments; no autonomous drafting). Output: a README that answers the questions below, or states explicitly that a question is deferred. Full-auto starts only after this phase; every unanswered question becomes an agent guess.

### Questions to answer in the README

**Purpose and scope**
1. Who is the user: oneself, a community, the general public? Target number of nodes and users in year one?
2. What is the primary value: privacy, anti-slop quality, bias labelling, independence from big search, or personal knowledge search? Rank them; they conflict (e.g. privacy vs. federated queries).
3. What is explicitly out of scope (images, video, news, maps, accounts, ads, LLM answers)?
4. Which languages must work first?
5. Success criteria: what does "it works" look like for M1 and for the full system?

**Index and crawling**
6. What gets crawled: the whole web, a seed list, pages the user visits, bookmarks, or a mix? Is browsing history ever an input?
7. How big may a local index get (disk) and how much bandwidth per day is the default cap?
8. Is the crawler allowed to ignore `robots.txt`? Which identification (user agent) does it use?
9. What is stored per page: full text, extracted text only, snippet only, or just terms? This decides legal exposure and what peers can share.
10. How fresh must results be, and how are re-crawls scheduled?
11. Are JS-rendered pages crawled (headless browser) or skipped?

**Federation**
12. What exactly is federated: raw queries sent to peers, index shards, term lookups, or only filters? "Federated with other indices" has several incompatible meanings.
13. Who are the peers: anyone (open network), invited friends, or a curated list? How are peers found?
14. Is leaking the query text to peers acceptable? If not, which cost (latency, recall, complexity) is acceptable to avoid it?
15. What happens when peers return junk or lie? Is a bad peer dropped, down-weighted, or ignored?
16. Must the system work offline or with zero peers (local-only mode)?

**Filters and labels**
17. Concrete label taxonomy: fixed set or user-defined? Who may create a filter list?
18. The five behaviours (label, block from index, disfavor, favor, informative): are they properties of the filter list, of the subscriber's config, or both? Can a subscriber override a publisher's intent?
19. Granularity: whole domain, path prefix, single URL, or content hash? (Shinigami Eyes works on accounts/domains.)
20. Bloom filters cannot enumerate, delete, or carry values. Is that acceptable? How are false positives handled (a false positive on a "block from index" filter silently hides a legitimate site)? What false-positive rate is tolerable, and is an appeal/override path required?
21. Update model: how are removals and corrections propagated (a domain that was labelled wrongly, or changed owner)?
22. Trust: who signs filters? Is there a default list shipped with the software, and who controls it? How are malicious or targeted lists (labelling a person's site to harass them) handled?
23. Is the tagging UI public or private by default? Does a user's tagging reveal an identity, and is that wanted?
24. Which labels are the project willing to ship or host, and which not (e.g. politically contested labels such as "propaganda")? This is a policy decision, not a technical one.
25. How is "AI-generated" or "SEO slop" decided: human tags only, or automated classifiers feeding the filters? If classifiers: which, and who audits them?

**Ranking and results**
26. Baseline ranking algorithm: BM25 only, link-based signals, or something else? How strongly may filters move a result (hard rerank vs. soft boost)?
27. Does the result page always show why a result was shown or demoted?
28. Is there any personalisation, and where does that data live?

**Frontend and extension**
29. "Minimal JavaScript": hard rule (no-JS must be fully functional) or preference? Which features may require JS?
30. Is the web UI local-only (localhost), or can it be hosted for others?
31. Firefox only, or later Chromium and Safari? What does the extension send to the local engine and when?

**Architecture and operations**
32. One binary or separate services? Does "service" mean separate processes with network APIs, or crates in a single daemon?
33. Target platforms (macOS, Linux, Windows, ARM, Raspberry Pi)? Minimum hardware?
34. Storage and runtime constraints: embedded engine such as tantivy acceptable, or must the index format be custom/open?
35. Install story: single binary, Docker, package manager? Auto-update?
36. Configuration: file, CLI flags, UI? Defaults for a non-technical user?

**Security, privacy, legal**
37. Threat model: who is the adversary (SEO spammers, state censors, harassers, curious peers, malware)?
38. Which personal data may exist anywhere in the system (IPs, queries, tags)? Retention rules?
39. Jurisdiction and licensing: repo license is in place (see `LICENSE`); what about licensing of shared filter lists and of indexed content that is redistributed?
40. Is content cached or shown to others a copyright concern the project accepts, avoids (links and snippets only), or defers?

**Governance and delivery**
41. Who decides contested design questions and filter policy: Yvonne alone, a maintainer group, a community vote?
42. Existing projects to reuse or compare against: YaCy, Marginalia, Mojeek, SearXNG, Shinigami Eyes, Brave Search goggles, Mastodon blocklists, Common Crawl. What does Swarmsearch do that these do not?
43. Order of delivery: is M1 (local-only) acceptable as the first usable release, or does the project only make sense with federation?
44. Budget or deadline constraints for the autonomous build (wall-clock, compute, money)? Who reviews outputs, and at which points?

### How to use the questions

- Minimum viable answers: 2, 6, 9, 12, 14, 17-20, 22, 24, 32, 37. These decide architecture, so rework starts there.
- Questions 1-5 and 41-44 can be one short paragraph each.
- Unknown answers are acceptable if marked `DECIDE LATER: <condition>`; Phase 0 then turns them into decision-log entries rather than agent guesses.
- Claude's role in this phase: point out contradictions between answers (e.g. strong privacy + open federated queries), name missing questions, and challenge vague terms ("decentralized", "federated", "bias"). Claude does not fill in answers.
- Exit criterion: README contains no sentence that two readers could interpret as different systems; Yvonne marks it frozen.

## Phase 0: Spec hardening (sequential)

1. Spec agent expands the README into `docs/spec/`: threat model, data model, wire protocol, filter semantics, glossary.
2. Three adversarial reviewer agents attack the spec independently: security/abuse, privacy, distributed-systems correctness. Findings are merged into the spec.
3. Decision log (`docs/adr/`) for each open question:
   - Peer discovery: libp2p Kademlia DHT vs. gossip vs. static seed lists + peer exchange.
   - Federated query model: fan-out to N peers with timeout, vs. DHT-routed term lookups.
   - Filter distribution: signed filter lists over HTTP/ActivityPub-style subscription vs. libp2p pubsub.
   - Bloom filter variant: standard vs. counting vs. cuckoo/xor filters. Bloom filters cannot delete and cannot be enumerated, which conflicts with "label a source" (a label needs a value, not just membership). Likely answer: one filter per label, or a keyed filter map.
   - Trust and Sybil resistance: signed filters, publisher keys, web-of-trust weighting, reputation.
   - Index engine: build on `tantivy` vs. custom. Default: tantivy.
4. Human gate: the decision log is the one place a human review is worth blocking on. Everything after runs unattended.

## Phase 1: Contracts and scaffolding (sequential, one agent)

- Cargo workspace:
  - `crates/core`: shared types (`Url`, `Document`, `Label`, `FilterRef`, `Query`, `Result`), error types, config.
  - `crates/proto`: wire protocol definitions + generated code.
  - `crates/index`: local index service (tantivy-backed).
  - `crates/crawler`: crawler service.
  - `crates/filters`: filter formats, subscription client/server, signing.
  - `crates/engine`: query planner, ranking, local + federated search.
  - `crates/net`: peer transport, discovery, rate limiting.
  - `crates/web`: HTML+CSS frontend server (server-rendered, Axum + Askama/Maud).
  - `crates/cli`: `swarmsearch` binary tying services together.
  - `extensions/firefox`: WebExtension.
  - `tests/e2e`, `tests/sim`: multi-node harness.
- CI: fmt, clippy, test, deny, audit, coverage floor, fuzz smoke, benches.
- Trait definitions with mock implementations so every downstream crate compiles and tests against mocks on day one.

## Phase 2: Parallel build (one agent per track, worktree-isolated)

| Track | Scope | Gate |
|-------|-------|------|
| A. Index | Schema, ingest API, segment management, query API, per-filter-label fields, snapshot/export | Property tests on ingest/query round-trip; bench on 10M docs |
| B. Crawler | Fetcher, robots.txt, politeness, token-bucket bandwidth limit, URL frontier, dedupe (SimHash), content extraction (readability), language detection | Local test web server with traps (infinite pages, redirects, slow loris); bandwidth cap asserted within 5% |
| C. Filters | Filter formats, label types (`label`, `block-from-index`, `disfavor`, `favor`, `info`), signing, versioning, delta updates, subscription protocol | Fuzz parsers; false-positive-rate tests against configured targets; signature tamper tests |
| D. Engine | Ranking with filter weights, user filter config, result annotation, federated fan-out, result merging/dedupe, timeout handling | Golden-query relevance tests; chaos tests with dead/slow/malicious peers |
| E. Net | Transport, peer discovery, auth, per-peer rate limits, abuse handling | Multi-process sim with 50+ nodes |
| F. Web UI | Search page, results with filter badges, tag/untag actions, favor/disfavor config, no-JS baseline | Playwright e2e with JS disabled; accessibility checks (axe); page weight budget |
| G. Firefox ext | Context menu + toolbar tagging, inline page badges (Shinigami-Eyes style), local-engine API client | web-ext lint; Playwright-driven Firefox e2e |
| H. Docs/Ops | User docs, operator docs, config reference, packaging (Docker, Homebrew, deb) | Doc-test of every command; clean-machine install test |

Each track agent loops: implement → run gate → fix → request review → address review, until gate and reviewer both pass, then opens a PR against the integration branch.

## Phase 3: Integration (sequential, integration agent)

- Merge tracks in dependency order: core → proto → index → filters → crawler → net → engine → web → extension.
- Replace mocks with real implementations one seam at a time; run the full suite after each swap.
- Fix cross-track contract drift. Drift is resolved by amending the contract and re-running all consumers, not by patching one side.

## Phase 4: Adversarial verification (parallel, fresh agents)

- Red team: spam the index with SEO/AI slop to test filter-based blocking; poison filters via malicious publishers; Sybil peers returning crafted results; query-response amplification; crawler SSRF (internal IPs, `file://`, DNS rebinding); XSS via indexed content rendered in results; ReDoS in extraction.
- Privacy audit: what a peer learns from a federated query (query text, IP, interest profile); mitigation options (query batching, k-anonymity via cover queries, onion-style relay, or restricting federation to hashed term lookups).
- Resilience: network partitions, clock skew, corrupted index segments, crash-during-write recovery.
- Scale simulation: 1k–10k simulated nodes in a deterministic simulator (e.g. `turmoil` or custom) measuring query latency, recall, and filter propagation time.
- Relevance evaluation: seed a corpus, build a judged query set, compare against a BM25 baseline. LLM judges are acceptable for relevance grading provided at least two independent judges agree; flag disagreements.

## Phase 5: Hardening loop (repeat until convergence)

1. Reviewer agents run `/code-review` at max effort per crate.
2. Fixer agents address confirmed findings; each fix needs a regression test.
3. Re-run Phase 4 suites.
4. Stop when two consecutive rounds yield no confirmed findings above low severity.

## Orchestration

- Implement as a Workflow script: `pipeline(tracks, build, review)` for Phase 2, `parallel(attacks)` for Phase 4, a `while` loop with a convergence check for Phase 5.
- Worktree isolation per agent. Branch naming: `auto/<track>/<task>`.
- Shared memory: `docs/adr/` (decisions), `docs/spec/` (contracts), `STATUS.md` (per-track state). Agents read these, and only the integration agent writes `docs/spec/`.
- Checkpointing: workflow resume IDs stored in `STATUS.md` so a crashed run restarts from cached agent results.
- Budget guards are still worth having even with unlimited tokens: wall-clock timeout per task, max retry count per gate, and a stuck-detector (same failing test three iterations running) that escalates to a fresh agent with a different approach.

## What full-auto does not solve

- **Cold start.** An index of one user's crawl is nearly empty. Federation is useless without peers, and peers do not exist without users. Agents can build the software, not the network. A bootstrap plan (public seed node, pre-built index of a curated seed list) is needed.
- **Filter governance.** Labels like "propaganda" or "transphobia" are contested. Software can provide signed, subscribable lists. It cannot decide who is trusted. Abuse of the filter system (targeted labelling of individuals' sites) needs a policy decision.
- **Ranking quality.** Relevance can be measured against baselines but not guaranteed. Tuning needs real usage data.
- **Legal.** Crawling, caching, and redistributing content snippets across peers carries copyright and GDPR exposure. Needs human review.
- **Verification ceiling.** Passing tests means the code matches the spec, not that the spec is right. Phase 0 and the decision log carry the highest risk.

## Suggested milestones

1. **M1, local-only:** crawler → index → search page. No federation, no filters.
2. **M2, filters:** filter formats, local subscription, annotated results, tagging UI, Firefox extension against local engine.
3. **M3, federation:** peer discovery, fan-out queries, filter distribution between nodes.
4. **M4, hardening:** Phases 4–5, privacy mitigations, packaging.

Milestones ship in order. M1 is the smallest slice that exercises every Phase 1 contract except the network and can be validated end-to-end without other users.
