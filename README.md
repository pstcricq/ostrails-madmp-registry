# ostrails-madmp-registry

Registry of SOCIB research projects and their machine-actionable DMPs.

Every DMP here is written by a bot, never by hand. Researchers fill in their DMP
in DSW (Data Stewardship Wizard) and click Submit, the submission webhook
commits the rendered JSON into this repository. Nobody needs a GitHub account to
appear here.

This README is the **shared contract** between the two sides that write here,
and both are [madmp-core](https://github.com/pstcricq/ostrails-madmp-core): its
CI lays out a project's folder, and the submission webhook it ships, deployed
next to DSW, commits the DMPs. Neither may drift from what is below.

**This repository runs no CI.** It holds data, and everything that judges that
data runs before the data arrives.

## Layout

```
projects/
  <id>/
    template/
      .gitkeep                        laid out by madmp-core, and the mark that it was
      dmp_<id>_template.json          the project DMP, RDA DCS and nothing else
      dmp_<id>_template.meta.json     the versions that DMP was built from
      dmp_<id>_template.check.json    the verdict it got against them
    productions/
      .gitkeep                        deployment DMPs, placeholders resolved
```

`<id>` is the project's one and only machine name. The same string names the
project's config file in madmp-core, this folder, and the DSW packages
generated for it, there is no separate folder name to keep in step. It matches
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

That writes the two `.gitkeep` files through the GitHub Contents API, no clone,
so the size of this repository never matters to either side. Git stores no empty
directory, so those two files are the whole of what a laid-out folder is, and
their presence is what the webhook reads before it writes anything.

It is idempotent: nothing is sent when nothing changed, so a push that touched
no config leaves no commit here. It never deletes, and it never writes outside a
project's own folder.

## `dmp_<id>_template.meta.json`

The versions a DMP was built from, committed beside it, by the same commit:

```json
{
  "project": "glider",
  "template_version": "1.0.0",
  "rules": [{"rda_dcs": "1.0.0"}, {"ostrails": "1.0.0"}]
}
```

- **`project`** is the folder this document was generated for. The webhook
  refuses a document whose `project` is not the folder it was routed to.
- **`template_version`** is the DSW document template that rendered it, for
  tracing a DMP back to the package it came out of. Nothing reads it.
- **`rules`** is what quality control checks the DMP against.

**This file is the only record of what a given DMP was built from, and it is
written by the same commit as the DMP.** That is the whole reason it exists
here rather than at the root of the folder: a file describing a project rather
than a document would say where the project stands *today*, and every DMP
already committed would be checked against rules it was never written against.

The values come from the document itself. madmp-core's document template stamps
them into a `metadata` object beside `dmp` when it renders, so a researcher left
on an older template goes on submitting that template's versions. The webhook
takes that object out of the document, which is why what lands in
`dmp_<id>_template.json` is RDA DCS and nothing else.

## How a DMP arrives

DSW's submission service is registered per project by madmp-core, scoped to that
project's document template. On Submit, DSW POSTs the rendered document to
`/submissions?project=<id>` of the single shared webhook.

The webhook is stateless and does exactly this:

1. Rejects an `<id>` that does not match the slug pattern.
2. Takes the `metadata` object out of the document, and rejects a document that
   carries none, one naming another project, or one whose `rules` are not a
   list of `{standard: version}` mappings. A DMP whose versions are unknown
   cannot be checked against them.
3. Rejects a folder with **no `template/.gitkeep`**, an uninitialized folder
   being one nobody registered. The folder has to be laid out first.
4. Rewrites the document's `dmp_id`, a DSW project URL placeholder at that
   point, to the file's stable raw URL in this repository.
5. **Checks the document** against the rules those versions name, and refuses
   it with a `422` when it does not hold up, naming the first few violations.
   Nothing is read or written before this: it needs no network, and it is the
   answer the researcher is waiting for.
6. Writes **the three files in one commit**, through the Git Data API: a tree,
   a commit, and one move of the branch reference. Separate writes would leave
   a DMP whose versions or verdict are missing whenever the second failed, and
   the webhook holds no state to repair that with.

**What is here has passed.** A DMP that does not hold up is refused at
submission time and never reaches this repository, which is what makes the
default branch readable as "the DMPs that hold up" without anything having to
enforce it here.

**It creates nothing**, no repositories, no folders, no scaffolding. Laying out
a folder belongs to madmp-core's CI, which knows the config, the webhook only
knows the folder name it was handed.

The folder is the *only* routing input. Which project a submission belongs to is
decided by the `?project=` parameter baked into the DSW service, and the
document's own `metadata.project` is checked against it rather than trusted in
its place.

## `productions/`

The directory exists in every project folder, and nothing writes to it yet. It
is where deployment DMPs, the project template with its placeholders resolved,
will be committed when whatever produces them is built.

## Quality control

It does not run here. A document is checked **before** it is committed, by the
webhook, against the rules its own `metadata` names, and the verdict is
committed beside it:

```json
{
  "verdict": "pass",
  "summary": {"total": 100, "pass": 66, "fail": 0, "warning": 1, "missing": 33},
  "rules": [{"rda_dcs": "1.0.0"}, {"ostrails": "1.0.0"}],
  "engine": "1.0.0",
  "warnings": [{"instance_path": "...", "message": "..."}]
}
```

A document is judged by the versions **it** names, so a project whose pins moved
since does not change the answer, and a DMP committed a year ago was checked
against the rules it was written against.

The passing results are not kept, they say only that a field is a field. The
warnings are, being the whole of what a document that passed still has to say.
There is no timestamp, git dates the commit, and one here would change the file
at every submission.

`engine` is the version of madmp-core that ran. To re-check a DMP by hand, that
is the version to use:

```bash
python -m quality_control.run \
  --pins projects/glider/template/dmp_glider_template.meta.json \
  --dmp  projects/glider/template/dmp_glider_template.json \
  --json /tmp/check.json
```

## Where the code lives

Rules, knowledge model, generators, the folder layout and the quality control
engine are in
[madmp-core](https://github.com/pstcricq/ostrails-madmp-core). The submission
webhook is deployed next to DSW, in
[ostrails-madmp-dsw](https://github.com/pstcricq/ostrails-madmp-dsw), and
carries its own copy of the GitHub client: the two sides share this layout, not
that code, so a change on one side reaches the other only if someone carries it
over.
