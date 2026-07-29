# dmp-registry

Registry of SOCIB research projects and their machine-actionable DMPs.

Every DMP here is written by a bot, never by hand. Researchers fill in their DMP
in DSW (Data Stewardship Wizard) and click Submit; the submission webhook commits
the rendered JSON into this repository. Nobody needs a GitHub account to appear
here.

This README is the **shared contract** between the two sides that write here:
[madmp-core](https://github.com/Pierrott64/madmp-core) (`dsw.publish registry`,
which creates a project's folder) and the submission webhook deployed next to
DSW (which drops DMPs into it). Neither side may drift from what is below.

## Layout

```
projects/
  <folder>/
    meta.yaml                       identity + pinned rules versions, written by `publish registry`
    template/
      dmp_<folder>_template.json    the project DMP, may still contain {snake_case} placeholders
    productions/                    deployment DMPs, placeholders resolved
```

`<folder>` is the project acronym, lowercase, matching `^[a-z0-9-]{1,64}$`
(`glider`, `canales`, `endurance-line`). No dots and no slashes, so a submission
can never write outside its own folder.

## How a folder comes to exist

A folder is **never** created by a submission. It is created ahead of time from
the project's config in madmp-core:

```bash
python -m dsw.publish registry configs/projects/<id>.yaml
```

That writes `meta.yaml` and the empty `template/` and `productions/` dirs via the
GitHub Contents API (no clone, so the size of this repo never matters). It never
deletes: republishing only updates `meta.yaml`. If the folder already carries a
different project `id`, it refuses rather than clobber it.

`meta.yaml` holds the project's identity and the **pinned rules versions** the
quality control reads, plus a QC status slot CI fills in:

```yaml
folder: glider
id: socib-glider
name: SOCIB Glider
rules:
  - rda_dcs: 1.0.0
  - ostrails: 1.0.0
  - socib: 1.0.0
qc:
  status: null
  checked_at: null
```

## How a DMP arrives

DSW's submission service is registered per project by
`dsw.publish submission`, scoped to that project's document template. On Submit,
DSW POSTs the rendered document to `/submissions?project=<folder>` of the single
shared webhook.

The webhook is stateless and does exactly this:

1. Rejects a `<folder>` that does not match the slug pattern.
2. Rejects a folder with **no `meta.yaml`** — an uninitialized folder means no
   rules pins, so a DMP dropped there would be an orphan. Run
   `publish registry` first.
3. Rewrites the document's `dmp_id` — a DSW project URL placeholder at that
   point — to the file's stable raw URL in this repository.
4. Commits it to `projects/<folder>/template/dmp_<folder>_template.json`.

It is idempotent: an unchanged DMP commits nothing. It creates no repositories
and no scaffolding.

The folder is the *only* routing input. There is no identity resolution here:
which project a submission belongs to is decided by the `?project=` parameter
that `publish submission` baked into the DSW service, not by anything read out
of the document.

## Quality control

CI runs the maDMP quality control on every DMP it receives, using the rules
versions pinned in that project's `meta.yaml`, and records the outcome in
`meta.yaml`:

- `pass` — no failing check
- `pass-with-warnings` — optional fields missing, suggested vocabularies not
  followed
- `fail` — invalid JSON, missing required field, cardinality or cross-field rule
  broken

A failing DMP is still committed: the artifact is kept so it can be inspected and
fixed. What a failure blocks is downstream use — a project whose template is
`fail` cannot be used to generate deployment DMPs in `productions/`. It never
blocks data acquisition.

Rules, knowledge model and generators live in
[madmp-core](https://github.com/Pierrott64/madmp-core).
