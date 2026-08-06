# dmp-registry

Registry of SOCIB research projects and their machine-actionable DMPs.

Every DMP here is written by a bot, never by hand. Researchers fill in their DMP
in DSW (Data Stewardship Wizard) and click Submit; the submission webhook commits
the rendered JSON into this repository. Nobody needs a GitHub account to appear
here.

This README is the **shared contract** between the two sides that write here:
[madmp-core](https://github.com/Pierrott64/madmp-core), which lays out a
project's folder, and the submission webhook deployed next to DSW, which drops
DMPs into it. Neither side may drift from what is below.

## Layout

```
projects/
  <id>/
    meta.yaml                   identity + pinned rules versions, written by madmp-core
    template/
      dmp_<id>_template.json    the project DMP, may still contain {snake_case} placeholders
    productions/                deployment DMPs, placeholders resolved
```

`<id>` is the project's one and only machine name. The same string names the
project's config file in madmp-core, this folder, and the DSW packages
generated for it — there is no separate folder name to keep in step. It matches
`^[a-z0-9-]{1,64}$` (`glider`, `canales`, `endurance-line`), which is the
pattern madmp-core's config schema already enforces: no dots and no slashes, so
a submission can never write outside its own folder.

## How a folder comes to exist

A folder is **never** created by a submission. It is laid out ahead of time from
the project's config in madmp-core, by that repository's CI on its default
branch:

```bash
REGISTRY_OWNER=... REGISTRY_REPO=... REGISTRY_TOKEN=... python scripts/sync_registry.py
```

That writes `meta.yaml` and creates `template/` and `productions/` through the
GitHub Contents API — no clone, so the size of this repository never matters to
either side. It is idempotent: nothing is sent when nothing changed, so a push
that touched no config leaves no commit here.

It never deletes, and it never overwrites a folder that carries a different
project `id` — it refuses instead, because that folder is where somebody else's
DMPs land. It refuses the same way when a `meta.yaml` is there and does not
parse: rewriting it would drop the pins below and any key another writer owns,
so that one is fixed by hand.

## `meta.yaml`

Two keys, and they are madmp-core's:

```yaml
id: glider
rules:
  - rda_dcs: 1.0.0
  - ostrails: 1.0.0
```

- **`id`** — the project, and the name of this folder. A folder whose `id` does
  not match the config claiming it is a collision, and is refused rather than
  overwritten.
- **`rules`** — the rules versions this project's DMP was built from, frozen at
  registration. A config's pins move; these do not. Nothing reads them yet, and
  they are written anyway: after the fact, nobody could say what was pinned when
  a given DMP was submitted. The rendered DMP carries no trace of them, so this
  file is the only place they exist.

  **The freeze is per project, and this file is rewritten when a config's pins
  move.** Its current state therefore says what the *next* submission will be
  built from. What a DMP already committed here was built from is the state of
  this file **at that DMP's commit** — both live in this repository, so the two
  are read together:

  ```bash
  git log -1 --format=%H -- projects/<id>/template/dmp_<id>_template.json
  git show <that-commit>:projects/<id>/meta.yaml
  ```

**This file may have more than one writer, and each owns its own keys.**
madmp-core writes `id` and `rules`, compares only those, and carries every other
key across untouched — so anything this repository's own CI records here (a
quality-control verdict, when there is one) is safe from being erased on the
next sync, and does not read as drift. What decides whether the file is rewritten
is the document, not the bytes: key order alone is not a change.

## How a DMP arrives

DSW's submission service is registered per project by madmp-core, scoped to that
project's document template. On Submit, DSW POSTs the rendered document to
`/submissions?project=<id>` of the single shared webhook.

The webhook is stateless and does exactly this:

1. Rejects an `<id>` that does not match the slug pattern.
2. Rejects a folder with **no `meta.yaml`** — an uninitialized folder has no
   rules pins, so a DMP dropped there would be an orphan. The folder has to be
   laid out first.
3. Rewrites the document's `dmp_id` — a DSW project URL placeholder at that
   point — to the file's stable raw URL in this repository.
4. Commits it to `projects/<id>/template/dmp_<id>_template.json`.

It is idempotent: an unchanged DMP commits nothing. **It creates nothing** — no
repositories, no folders, no scaffolding. Laying out a folder belongs to
madmp-core, which knows the config; the webhook only knows the folder name it
was handed.

The folder is the *only* routing input. There is no identity resolution here:
which project a submission belongs to is decided by the `?project=` parameter
baked into the DSW service, never by anything read out of the document.

## `productions/`

The directory exists in every project folder, and nothing writes to it yet. It
is where deployment DMPs — the project template with its placeholders resolved —
will be committed when whatever produces them is built.

## Quality control — not built yet

There is **no quality control running in this repository today**. When it is
built, it will read the `rules` pins of a project's `meta.yaml` and check the
submitted DMP against the very rules it was built with, then record a verdict
under a key of its own here.

It has to read those pins **as of the commit of the DMP it is checking**, not
as the file stands today — see `meta.yaml` above. Reading the current state
would check an older DMP against rules it was never built with, and would do so
silently, since both are valid documents.

Two things are already in place for it, and are the reason it can be added
without changing anything above: the pins are frozen at registration, and
madmp-core carries across every key it does not own.

Nothing in this section is a promise about behaviour that exists. It is written
down so that the first commit that adds it does not have to renegotiate the
contract.

## Where the code lives

Rules, knowledge model, generators and the folder layout are in
[madmp-core](https://github.com/Pierrott64/madmp-core). The submission webhook
is deployed next to DSW and carries its own copy of the GitHub client: the two
sides share this layout, not that code, so a change on one side reaches the
other only if someone carries it over.
