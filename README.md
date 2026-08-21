# ostrails-madmp-registry

The database of SOCIB research projects and their machine-actionable DMPs.

It holds data and nothing else. No code, no workflow, no CI. Everything that
produces what is here, and everything that judges it, is in
[madmp-core](https://github.com/pstcricq/ostrails-madmp-core) and runs before
the data arrives.

**Every file here is written by a machine, never by hand.** Researchers fill in
their DMP in DSW (Data Stewardship Wizard) and click Submit. Nobody needs a
GitHub account to appear here.

**What each file holds, field by field, is in the technical reference:**
<https://pstcricq.github.io/ostrails-madmp-technical-docs/registry/01-layout/>

## Layout

```
projects/
  <id>/
    template/
      .gitkeep                        the mark that this project is registered
      dmp_<id>_template.json          the project DMP, RDA DCS and nothing else
      dmp_<id>_template.meta.json     the versions that DMP was built from
      dmp_<id>_template.check.json    the verdict it got against them
    productions/
      .gitkeep                        deployment DMPs, placeholders resolved
```

`<id>` is the project's one and only machine name. The same string names the
project's config in madmp-core, this folder, and the DSW packages generated for
it. It matches `^[a-z0-9-]{1,64}$` (`glider`, `canales`, `endurance-line`): no
dots and no slashes, so nothing can ever be written outside a project's own
folder.

Git stores no empty directory, so the two `.gitkeep` files are the whole of what
a registered project is before it has a DMP. A folder without them is a project
nobody registered, and a submission to it is refused rather than half written.

## `template/`

The project DMP: one plan per project, with `{snake_case}` placeholders where a
value belongs to a deployment rather than to the project.

Three files, always written together in a single commit. A DMP whose versions or
whose verdict were missing would be a DMP nobody can check, and nothing here can
repair that after the fact.

### `dmp_<id>_template.json`

The DMP itself, RDA DCS and nothing else. Its `dmp_id` is the file's own stable
raw URL in this repository.

### `dmp_<id>_template.meta.json`

The versions the DMP was built from:

```json
{
  "project": "glider",
  "template_version": "1.0.0",
  "rules": [{"rda_dcs": "1.0.0"}, {"ostrails": "1.0.0"}]
}
```

- **`project`** is the folder this document was generated for.
- **`template_version`** is the DSW document template that rendered it, for
  tracing a DMP back to the package it came out of.
- **`rules`** is what the DMP was checked against.

**This file is the only record of what a given DMP was built from.** It sits
beside the document rather than at the root of the folder because a file
describing a *project* would say where that project stands today, and every DMP
already committed would read as checked against rules it was never written
against.

### `dmp_<id>_template.check.json`

The verdict the DMP got against the versions its `meta.json` names:

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
since does not change the answer, and a DMP committed a year ago reads as
checked against the rules it was written against.

The passing results are not kept, they say only that a field is a field. The
warnings are, being the whole of what a document that passed still has to say.
`engine` is the version of madmp-core that ran. There is no timestamp, git dates
the commit, and one here would change the file at every submission.

**What is here has passed.** A DMP that does not hold up is refused at
submission time and never reaches this repository, which is what makes the
default branch readable as "the DMPs that hold up" without anything having to
enforce it here.

To re-check one by hand, with the version its verdict names:

```bash
python -m quality_control.run \
  --pins projects/glider/template/dmp_glider_template.meta.json \
  --dmp  projects/glider/template/dmp_glider_template.json \
  --json /tmp/check.json
```

## `productions/`

The directory exists in every project folder, and nothing writes to it yet. It
is where deployment DMPs, the project template with its placeholders resolved,
will be committed when whatever produces them is built.

## Where the code lives

All of it in [madmp-core](https://github.com/pstcricq/ostrails-madmp-core):
the rules, the knowledge model, the generators, the quality control engine, the
script that lays out a project's folder here, and the submission webhook that
commits into it. It runs inside
[madmp-dsw](https://github.com/pstcricq/ostrails-madmp-dsw), the deployment.

Two programs write here, each with its own token, its own trigger and its own
strictly bounded reach. Which one writes which file, and what neither of them
can do, is in the technical reference:
<https://pstcricq.github.io/ostrails-madmp-technical-docs/registry/03-writers/>
