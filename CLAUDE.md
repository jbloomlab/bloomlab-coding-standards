# Lab coding standards (master rules)

These rules apply to all lab repositories. Project-level CLAUDE.md files may extend or override them. Rationale and examples: [`bloomlab-coding-standards/BEST_PRACTICES.md`](BEST_PRACTICES.md) (consult only when needed). This file is the normative rule set; if the two documents ever disagree, this file wins.

When the user asks to run the lab review (`/review`, "structural review", "lab review", or similar), follow the instructions in [`bloomlab-coding-standards/commands/review.md`](commands/review.md).

## Repository organization

- Standard layout, at the repository root: `config.yml` (configuration), `Snakefile` (workflow), `rules/` (`.smk` files), `scripts/`, `notebooks/`, `data/` (all input data), `results/` (all generated output), `environment.yml` (unless the project uses a lab pipeline's environment), `pyproject.toml` (wherever there is Python; holds the linter and formatter configuration), `.gitignore`, `README.md`, `CLAUDE.md`. As needed: `auspice/`, `non-pipeline_analyses/`, `envs/` (per-rule conda environments), `LICENSE`, a cluster submission script, and lab pipeline submodules.
- A simple project may be a few scripts with no `Snakefile`; anything multi-step is a Snakemake workflow.
- Code reads from `config.yml` and `data/`; it writes only to `results/` (or `auspice/`, or a declared temp dir). The one exception is a documented script kept beside the data file it generates, run by hand, whose output is committed as data.
- Paths in code, config, and data files are relative — to the repository root, or to a root configured in `config.yml` for files in an external store. The one exception is large raw sequencing data (FASTQ) held on the cluster, which may be referenced by absolute path, and then only from `config.yml` or a data manifest, never from a script. `snakemake --lint` flags absolute paths.
- Nextstrain Auspice JSONs may optionally be written to `auspice/` rather than `results/`, since Nextstrain Community Builds display trees from that directory in the repository.
- Analyses run outside the main pipeline go in `non-pipeline_analyses/`, each with an explanatory README. Its own configuration and conda environment are often a good idea there but are not required. Prefer that name; where a project uses another (`manual_analyses`, `scratch_notebooks`), it is acceptable, but suggest renaming to `non-pipeline_analyses`. Do not let this directory bloat: an analysis there must not re-implement what the pipeline already does, and one that becomes load-bearing is migrated into the pipeline as a rule plus a script — porting only the part the pipeline does not already provide, rather than moving a notebook across whole.
- Every `.gitignore` starts with the lab default, in this order: `.*` and `!.git*` (ignore dotfiles but keep `.gitignore`, `.gitmodules`, and `.github/`), `_*` (scratch files), `*.out` (cluster job output). Project-specific rules go below it, chiefly the `results/` block. A dotfile that must be tracked needs `git add -f`, since git will not add an ignored path by name.
- Throwaway files — an exploratory script, an intermediate table, a scratch note — are named with a leading `_` and ignored via `_*` in `.gitignore`. Use that prefix rather than cluttering `scripts/` or writing outside the repository; nothing `_`-prefixed is part of the project, and none of it should ever be committed.
- Never modify files in `data/` or values in `config.yml` to make code run. If code and config/data disagree, report the mismatch and stop.

## Input data

A file in `data/` records what was done at the bench, and for some of that it is the only record there will ever be. Write it to be read literally rather than reconstructed.

How input data is best organized depends on what it is: a table with one row per unit, a directory per unit whose filenames are the schema, a YAML description, sequences with a metadata sidecar. Choose the shape that expresses the experiment most directly, and choose it deliberately. The rules below apply whatever the shape — read *column* as whatever that shape's equivalent is: a field, a key, a file within a unit's directory.

- **Record it; do not infer it later.** A fact known when the experiment was set up belongs in the input file. Never recover it downstream from a heuristic — a nearest match, a name prefix, a hardcoded list of positions. Doing so discards information that was free to capture and silently caps what the analysis can detect.
- **One column, one job.** A column must not at once name an identity, mark a control kind, index a replicate, and carry a free-text note. Give each its own column, even where that column is constant for most rows.
- **An identifier names one thing.** Do not pack attributes into a name — a year, a passage, a subtype, a batch. A packed name cannot be joined on, grouped, or validated; a missing component is indistinguishable from an omission; and correcting an attribute changes the identifier itself, breaking every join and every reference to it. Give each attribute its own column and let the identifier stay a stable name.
- **No sentinel values.** Do not overload a column with a value meaning "not applicable" or "none". Add an explicit kind/type column and leave the inapplicable field empty.
- **If code or config has to be told what a value in a data file means, the data file is wrong.** A configuration key whose only job is to translate a value, or to exclude the rows a mislabeled value breaks, is a signal to fix the data file and delete the key.
- **Recording a value is not duplicating it.** Single source of truth forbids stating the same *fact* twice; it does not forbid recording a different fact that happens to be derivable. Where a file deliberately carries a derivable column as a cross-check, code must assert the two agree rather than trust either.
- **Never reconstruct a recordable fact from results.** A fact that can be recorded — which barcode went into a well, which sample is on which plate — is recorded in the input data, never recovered by analyzing the very results it will then be used to interpret. Otherwise a sample mix-up is laundered into a confirmed label by the analysis meant to catch it.
- **One current version of a file, not a history of them.** Fix a data file in place and explain the fix in the commit message; git holds the previous version. Do not accumulate `_old`, `_fixed`, or date-suffixed copies of the same file. A deliberately kept pair — a *designed* and an *actual* library — is rare and explained in the README.
- Where an input is tabular it is plain UTF-8 CSV/TSV, one header row, descriptive column names, and no meaning carried by formatting. Where it is not, the equivalents hold: descriptive names, and nothing meant by position or layout. Missing values are empty fields, not a mixture of `NA`/`-`/`?`/`999`.
- Read identifiers as strings (`dtype=str` when reading tables). Never let IDs be coerced to numbers.
- **Downloaded data records where it came from and when.** For a database download, a file taken from another repository, or an external release, record the source URL or query in a README beside it, and the download date in the filename, the directory name, or a small tracked date file. Re-running the same query later returns different data, so the date is what keeps the analysis interpretable. Where the source allows it, fetching the download in a rule is better than downloading by hand: the URL then lives in `config.yml` and the rule writes a tracked date file. Where the download itself is too large to track, track the date file even though the data is ignored. This applies only to data obtained from an external source: a file the experimenters themselves produced — a plate layout, a bench record, a sample manifest — has no external provenance to cite and needs none.
- Ask before changing what a data file records.

## Configuration

- No experimental parameters in code. Sample lists, inclusion/exclusion, thresholds, reference/database versions, statistical choices, hyperparameters, seeds, and paths live in `config.yml` — or in a data manifest where the value is per-unit — under descriptive keys. Name keys clearly enough that no comment is needed, and comment where the choice is not obvious.
- A constant that is not an experimental parameter — how many top barcodes a report prints, a plot dimension — may be hardcoded, declared once at the top of the script with a comment saying what it controls. Experimental parameters, thresholds, and sample lists never are.
- Do not enumerate in `config.yml` anything derivable from the data. Configuration names the inputs and the choices and lets the pipeline derive the rest, so that adding a sample does not require a configuration change.
- Configuration too complex or too bulky for `config.yml` — detailed plot specifications, long exclusion lists, site numbering maps — goes in its own file in `data/`, referenced by a `config.yml` key. It is still configuration, and all the rules above still apply to it.
- Access required keys directly (`config["key"]`), never `.get(key, default)`. Use `.get` only for genuinely optional keys, with a comment at the point of use saying why it is optional.
- Where fail-fast is deliberately relaxed, apply the default in exactly one place and comment why. Everything nested inside an optional section is still required.
- Keep configuration as simple as it can be while still conveying what is needed. Do not add a key, a nesting level, or an option that nothing requires; remove keys nothing reads.
- Where the config genuinely repeats a value or a block across entries, prefer factoring it with a YAML anchor over copy-pasting it. Weigh this against readability: a set of per-experiment entries — one block per plate, run, or batch — that share most fields but differ in a few is legible as written, and each entry reading as a standalone record of one experiment is worth something on its own. Prefer an anchor there, but do not require one. Factor the repetition where it goes past that: blocks identical in every field, a block repeated across unrelated sections of the config, or so many entries that a reader can no longer see which fields actually vary. The lab also uses YTE templating (`__use_yte__: true`) to generate a set of entries from one template; reach for that only when an anchor will not do, since a templated config is harder to read. Deriving the repetition from the data is better than either.
- YAML is indented two spaces per level, everywhere, including nested mappings.
- Validate config and input data at the start of a run: required keys/columns present, unique IDs, values in allowed sets, referenced files exist — and, where a unit is a directory of files, that every unit has every file its contract requires. Fail immediately, naming the file, column, and offending values.
- Validation covers structure and references, not the scientific content of an experimental parameter. Check that a key is present, that an identifier is unique, that a referenced file exists. Do not ask code to check that a recorded bench value is the *right* value — that a set of primer indices is the one used, that a dilution series matches the bench, that a plate date is correct. Those are confirmed against the bench record by a human, and a check with nothing to compare against is code that only looks like a safeguard.

## Results and what is tracked in git

- All generated output goes to `results/`, and can be deleted and regenerated.
- Ignore `results/` by default in `.gitignore`, then re-include the key scientific outputs — compact, human-readable CSV/TSV/YAML/TXT — so someone can read the main results without running the pipeline. Do not track large or binary regenerable output.
- `seqneut-pipeline` and `dms-vep-pipeline-3` each ship a reference `.gitignore` at `test_example/.gitignore`, which is the tracking policy for that pipeline's outputs. The `.gitignore` of a project that includes one of them must contain every ignore and re-include pattern from that reference, with the project's own patterns added below. Comments, blank lines, and ordering may differ; the set of patterns may not. A project including both carries the union. Re-check this after updating either submodule, since the reference gains patterns as the pipeline gains outputs.
- A path tracked because one of those two reference files re-includes it is settled: which of a pipeline's outputs are worth tracking is decided once, in the pipeline, and not re-argued per project.
- Beyond that, whether an intermediate is tracked is the project's call, made once in `.gitignore` and applied consistently. Some are worth keeping — per-sample counts that record a sequencing run nobody wants to repeat to read. Tracking one deliberately is not a finding; review may note what the choice costs as the project grows, but should not reopen it.
- The same tracking discipline applies to output of analyses outside the main pipeline.
- Nothing in `results/` is read as primary input by a later run of the same pipeline.
- Do not write a meaningless row index into an output CSV (pass `index=False` to pandas `to_csv`). Where the index carries real information — a sample ID, a site number — write it, and make sure it has a name.
- Do not write unneeded columns to an output CSV — no scratch columns, and no set of columns trivially derived from one another.
- Write numbers in output CSVs at a sensible precision, defaulting to 5 significant digits (`float_format="%.5g"` in pandas). More or fewer digits, or a fixed number of decimals (`"%.3f"`), are fine when chosen deliberately; full float repr is not. When writing code that emits numbers, ask what precision is wanted, and use 5 significant digits if the answer is not specified.

## Snakemake

- Rules may live in `Snakefile` itself or in `.smk` files in `rules/` that it includes; either is fine. Once `Snakefile` grows beyond a few hundred lines, break the rules out into `rules/*.smk`, one file per coherent analysis, each ending by defining the list of final outputs it contributes for `rule all` to collect.
- A rule is a thin wrapper: explicit `input`, `output`, `params`, and `log`; params sourced from config; the analysis itself delegated to a script. Do not put analysis logic in a `.smk` file.
- Everything a script needs arrives through `input`, `params`, or `wildcards` — never through an implicit default, a global, or a path the script reconstructs for itself.
- Every rule declares `log:` (`snakemake --lint` checks for it), and everything the rule prints goes there: a `script:` rule's script starts with `sys.stdout = sys.stderr = open(snakemake.log[0], "w")` and prints progress to stdout, and a `shell:` rule redirects into `{log}`. A failed run then leaves the log holding everything up to the error.
- Run Python through the `script:` directive rather than `shell:`.
- Consume an existing rule's output rather than recomputing the same quantity, and add a wildcard to an existing rule rather than copying it into a near-duplicate rule.
- Do not add a rule or a config key whose only purpose is to switch an analysis off; let the script report that there is nothing to show.
- Pin environments per rule where used, and dry-run with the same software deployment as a real run.

## Code style

- Code must pass `ruff`, `black`, `snakefmt`, and `snakemake --lint` before it is committed. Tool configuration lives in `pyproject.toml`. This covers the repository's own files; a submodule's are checked in its repository, not here.
- Scripts are small and single-purpose. Any logic used more than once becomes a named, documented function that is imported — never copy-pasted between scripts or notebooks.
- The default shape for "add an analysis": new config entry + new Snakemake rule + new small script (or reuse an existing function). Do not grow an existing script with unrelated logic, and do not duplicate a script with small edits.
- Keep code concise. Prefer simple, readable code over clever code, and do not add abstractions, generality, or dependencies beyond what the task requires.
- Comments explain why, not what the code already says, and are as short as they can be.
- **Write the current state, not its history.** When you improve something, just improve it: no comment, docstring, or README text explaining how it differs from a previous version, what it "now" does, or what it used to do. Git holds the history. Words like *now*, *new*, *updated*, *previously*, *legacy*, *no longer*, or *instead of* in a comment are a sign the text is narrating a change rather than describing the code.
- Where generated content must reach two places, write it once and copy or redirect it rather than writing it out twice.
- Watch for things nothing uses any more — a data file no config key or script reads, a script no rule runs, a rule no target needs, a config key nothing looks up, superseded results. Point them out and suggest pruning them; ask before deleting anything in `data/`, since it may be the only record of an experiment.
- Fail fast and loud: raise on unexpected values rather than substituting a default; no bare `except`; never silently drop, impute, or coerce data.
- Analysis projects do not require tests; validating inputs at the boundary is what stands in for them. Code that other projects import — a lab package, or a pipeline included as a submodule — does require tests covering new or changed shared functionality, in that repository.
- Prefer a Python script to a notebook, and say so when a notebook is proposed — especially for new work. Notebooks (Jupyter or marimo) are acceptable for exploration, for a deliverable interactive report, and where they already exist. A notebook the pipeline depends on must be run by a rule with explicit inputs and outputs.

## Altair plots

- A Vega-Altair chart embeds its entire dataframe in the spec, so a chart file is as large as the data handed to it. Give the chart only the columns and rows it draws; subset and select before constructing it, not in the encoding.
- Where a column repeats the same value across every row of a key — per-entity metadata carried on every row of a long measurement table — keep it in a small lookup table and pull it in with `.transform_lookup` rather than repeating it.
- `.transform_fold` can likewise shrink a chart that would otherwise carry several wide columns it plots as one key/value pair.

## Submodules, environments, and lab software

- Lab pipelines are included as git submodules. Read the submodule's own `CLAUDE.md` and follow its conventions; a project's `CLAUDE.md` should point at it.
- Never add project-specific rules or data to a shared submodule.
- A submodule's own code is linted and reviewed in its own repository, so in a project that includes one, raise nothing about how the submodule's files are written — no style, formatting, lint, length, or documentation findings. What is in scope is a genuine bug, and a change the upstream repository needs in order to function correctly; both belong there as an issue or a pull request, not as a local edit. Adding project-specific rules or data to the submodule, and working around an upstream limitation locally, remain findings — neither is a question of style.
- Do not recompute or re-plot what an included pipeline already produces; read its outputs.
- If a lab-maintained package (`seqneut-pipeline`, `dms-vep-pipeline-3`, `neutcurve`, `polyclonal`, `nextstrain-prot-titers-tree`, and similar) does not do exactly what is wanted, propose opening an issue in that repository. Tell the user; do not silently hack around it locally.
- Keep submodules and conda environments reasonably current with released software versions. Where a version is deliberately held back, say why in a comment.
- Conda environments list `conda-forge` first, then `bioconda`, and include `nodefaults`. Never list `defaults`. Omitting it is not enough: without `nodefaults`, conda appends whatever channels the local user has configured, so the file stops being an authoritative specification of the environment.
- An environment must contain the tools the project's code is checked with: `ruff` and `black` wherever there is Python, and additionally `snakemake` and `snakefmt` where there is a Snakemake workflow. A project that uses a lab pipeline's environment (`seqneut-pipeline`, `dms-vep-pipeline-3`) already has these; a project that defines its own `environment.yml` has to list them.

## Documentation

- `README.md` describes the project for a human: what it is, what the workflow does, and where the key results are. Keep it succinct and its sections evenly weighted; prefer cutting text to adding it.
- Say where something is configured, never what the current configuration is. Do not restate config values, thresholds, counts, or file lists in Markdown.
- Column meanings, types, and allowed values belong by default in the code that validates them and in the comments in `config.yml`, not in prose that goes stale. A README — the top-level one or one in `data/` — may describe columns in detail, but do this judiciously and only where the file is not self-documenting: a hand-authored input someone must create correctly before the pipeline will run, or a convention a reader could not infer from the file itself (that a blank field is a deliberate choice rather than an omission, say). Do not document columns by default, and do not let a README become a configuration spec.
- `CLAUDE.md` documents conventions only. It does not describe the project, individual rules, or configuration keys. Keep it short — no more than a few hundred lines.
- Describe a thing in exactly one place and cross-reference it from anywhere else that needs it.
- Keep `README.md` current: any change that adds, removes, or redirects an analysis or a result file updates it in the same change.

## Commits and pull requests

- Aim for one conceptual change per commit and per pull request. Some combining is unavoidable in practice; where a change starts accumulating conceptually unrelated edits, say so and propose a split rather than treating the combination as forbidden.
- A commit message says what changed and why, and names any workaround the change removes.
- All work reaches `main` through a pull request. Call out changes to `config.yml` or `data/` explicitly in the description, since those are the scientifically meaningful diffs.
- Configure the repository so the default branch cannot be pushed to directly: require a pull request to merge, and block force pushes.

## Rules for AI assistants (highest priority)

- **Never guess about experimental design or data semantics.** If a decision requires knowing what a column means, which samples to include or exclude, how to treat missing/duplicate/outlier values, which threshold or statistical method is intended, or anything else a scientist would decide — STOP and ask the human. Do not choose a "reasonable default", even a well-justified one. Ask one clear question stating the options.
- If an input fails validation, report it and stop. Do not "fix" data, relax a check, or special-case the code to proceed.
- **Watch the scope of the work as you go, not only at review.** Keep changes minimal and scoped to the request: do not refactor unrelated code, rename things, or reorganize files unless asked, and do not fold an unrelated fix or cleanup into the change in flight. If a change is accumulating conceptually unrelated edits, stop, say so, and propose how to split it across commits or pull requests.
- When a requested change alters scientific behavior (different samples, thresholds, methods), say so explicitly in your summary and in the commit/PR description, and ensure the change is expressed in `config.yml`, not buried in code.
