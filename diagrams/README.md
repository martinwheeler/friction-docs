---
title: friction diagrams
---

# Diagrams

Interactive diagrams of [friction](https://github.com/martinwheeler/friction), made with
[Archify](https://github.com/tt-a1i/archify). They are explained in [How friction works](../).

| Page | Source | Type |
|---|---|---|
| [System overview](architecture.html) | [architecture.json](https://github.com/martinwheeler/friction-docs/blob/main/diagrams/src/architecture.json) | architecture |
| [One occurrence, end to end](occurrence-sequence.html) | [occurrence.sequence.json](https://github.com/martinwheeler/friction-docs/blob/main/diagrams/src/occurrence.sequence.json) | sequence |
| [Item lifecycle](item-lifecycle.html) | [item.lifecycle.json](https://github.com/martinwheeler/friction-docs/blob/main/diagrams/src/item.lifecycle.json) | lifecycle |
| [Nightly backfill](backfill-workflow.html) | [backfill.workflow.json](https://github.com/martinwheeler/friction-docs/blob/main/diagrams/src/backfill.workflow.json) | workflow |
| [Jira sync, one run](jira-sync-sequence.html) | [jira-sync.sequence.json](https://github.com/martinwheeler/friction-docs/blob/main/diagrams/src/jira-sync.sequence.json) | sequence |
| [Database schema](schema-erd.html) | [schema.erd.json](https://github.com/martinwheeler/friction-docs/blob/main/diagrams/src/schema.erd.json) | erd |

## Rebuilding

Each source pins `meta.repository.revision` to the friction commit it describes, and every box
cites the lines it was drawn from. After changing friction, update the cited lines and the
revision, then rebuild from the root of this repository with a friction checkout beside it
(`../friction`):

```sh
ARCHIFY=~/.agents/skills/archify/bin/archify.mjs
build() { node $ARCHIFY finalize "$1" "diagrams/src/$2" "diagrams/$3.html" --repo-root ../friction --quality "${4:-showcase}"; }
build architecture architecture.json          architecture
build sequence     occurrence.sequence.json   occurrence-sequence
build lifecycle    item.lifecycle.json        item-lifecycle
build workflow     backfill.workflow.json     backfill-workflow
build sequence     jira-sync.sequence.json    jira-sync-sequence
build erd          schema.erd.json            schema-erd standard
```

`finalize` validates the source, checks every citation against the pinned commit, renders the
page and checks it in a headless browser. Its receipts (`*.finalize*.json`, `*.delivery.json`,
`*.browser-check.json`) are ignored by git.
