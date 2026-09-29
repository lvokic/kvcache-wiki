# Wiki operating guide

This repository is a persistent knowledge base maintained with an LLM. Its current scope is KV-cache management and LLM serving systems. When selecting additional papers, prioritize systems research from venues such as OSDI, SOSP, NSDI, FAST, SIGCOMM, ASPLOS, and MLSys. Include algorithmic or model-level work when it explains an important system design or trade-off. Keep source material separate from the evolving wiki, and agree with the owner before broadening the scope.

## Structure

- `raw/` contains owner-curated source files. Treat these as immutable: read them, but never edit, rename, or delete them.
- `wiki/index.md` is the content-oriented catalog. Keep it organized by page type and give each page a short description.
- `wiki/log.md` is the chronological, append-only record of ingests, substantial saved analyses, and lint passes.
- `wiki/sources/` contains one concise note per ingested source.
- `wiki/topics/` and `wiki/entities/` contain synthesis and entity pages as the subject requires.
- `wiki/analysis/` contains substantial answers that the owner asks to preserve.

Create only the page types that are useful for the chosen subject. Link pages with relative Markdown links so the wiki works in Obsidian and on disk.

## Evidence and writing

- Make clear which statements come from a source and which are synthesis or inference.
- Support substantive claims with links to the source note and, where practical, a locator in the raw source (page, section, timestamp, or heading).
- Preserve qualifications, dates, and relevant counterevidence. When sources disagree, record the disagreement with citations instead of silently choosing one account.
- Do not present an uncited claim as established fact. Do not add web research unless the owner asks for it.
- Prefer updating an existing page over creating a duplicate. Keep pages focused and cross-link related concepts, entities, and sources.
- Treat the wiki as derived knowledge, not a replacement for the raw sources.

## Workflows

### Ingest a source

When asked to ingest a source:

1. Read the source without modifying it. If the material cannot be read or its identity is unclear, report that before making claims about it.
2. Create a source note with bibliographic details when available, a concise summary, key claims, limitations, and links to relevant pages.
3. Find and update affected topic, entity, and overview pages. Add evidence and cross-links; flag contradictions or changes in confidence.
4. Update `wiki/index.md` for every page created or substantially changed.
5. Append one dated ingest entry to `wiki/log.md` with the source and pages touched.
6. Report the main takeaways, files changed, and unresolved questions.

### Answer a question

Start with `wiki/index.md`, then read the relevant pages and follow their citations when the answer depends on source detail. Cite the wiki pages and underlying sources in the response. Distinguish established evidence from interpretation and call out gaps. Save a durable analysis under `wiki/analysis/` only when the owner asks to preserve it; if saved, update the index and append a log entry.

### Lint the wiki

When asked to lint, check for broken links, uncited or stale claims, contradictions, duplicate or orphan pages, missing index entries, and useful missing cross-links. Report findings with file paths. Make straightforward repairs, then update the log with the date, scope, and fixes; do not invent evidence to fill gaps.

## Page conventions

Use descriptive filenames and a clear title at the top. Add lightweight YAML frontmatter only when it improves navigation (for example, `type`, `created`, or `updated`). Avoid metadata that has no defined use. Keep source notes traceable to a stable path under `raw/`.
