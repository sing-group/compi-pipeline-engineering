---
name: compi-pipeline-engineering
description: >
  Diagnose and improve Compi bioinformatics pipelines against SOLID design
  principles and reproducibility best practices. Covers task decomposition,
  interface contracts, inline-vs-src task code, runner design, robustness
  checks, project layout conventions (both legacy and new-style), and
  test-based verification of changes.
compatibility: opencode
---

## When to use this skill

Load this skill when you are:

- Reviewing an existing Compi pipeline for design quality
- Refactoring tasks (extracting inline code, splitting responsibilities, adding glue tasks)
- Adding environment, input, or monitoring checks to a pipeline
- Setting up or extending a test environment for a Compi pipeline
- Verifying that changes to a pipeline did not break its behaviour
- Running pipeline validations or tests

---

## Compi pipeline fundamentals

### What Compi is

Compi is a language-agnostic, XML-driven workflow management system for portable computational pipelines. Pipelines are defined in XML; each task runs an arbitrary piece of code in any language (Bash, Python, R, AWK, etc.). Compi handles dependency resolution, parallel execution, partial execution, and resumption. Pipelines can be packaged as Docker images using `compi-dk`.

The Compi ecosystem comprises:
- `compi` — the workflow engine and CLI
- `compi-dk` — build and packaging tool (embeds `compi` inside a Docker image)
- Compi Hub — public pipeline repository at `https://www.sing-group.org/compihub`

### Pipeline XML structure

A pipeline XML file has four sections: version, params, tasks, and optional metadata.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<pipeline xmlns="http://www.sing-group.org/compi/pipeline-1.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

    <version>1.0.0</version>

    <params>
        <!-- Global params: automatically injected as env vars into every task -->
        <param name="working_dir" shortName="w" global="true"/>
        <param name="samtools_image" defaultValue="pegi3s/samtools_bcftools:1.22" global="true"/>

        <!-- Non-global params: must be listed in the task's params= attribute -->
        <param name="min_mapping_quality" defaultValue="20"/>

        <!-- Flags: boolean, activated by presence (no value) -->
        <flag name="keep_intermediate_files" global="true"/>
    </params>

    <tasks>
        <!-- Task with external script (preferred) -->
        <task id="align-reads"
              after="prepare-reference"
              params="min_mapping_quality"
              src="tasks/align_reads.sh"/>

        <!-- Task with inline code (common in legacy pipelines, discouraged for complex logic) -->
        <task id="prepare-reference" after="initialization">
            mkdir -p ${working_dir}/index
            hisat2-build ${working_dir}/${genome_fasta} ${working_dir}/index/genome
        </task>

        <!-- Conditional task: skipped if the if= command exits non-zero -->
        <task id="batch-correction"
              after="pca"
              src="tasks/batch_correction.sh"
              if="[[ ${batch_correction} != 'none' ]]"/>

        <!-- Foreach task: one iteration per item in a collection -->
        <foreach id="align-samples"
                 after="prepare-reference"
                 of="command"
                 in="${scripts_dir}/list_samples.sh ${working_dir}"
                 as="sample_name"
                 src="tasks/align_sample.sh"/>

        <!-- Iteration-level dependency: nth iteration of sort-bam waits only
             for the nth iteration of align-samples (not all of them) -->
        <foreach id="sort-bam"
                 after="*align-samples"
                 of="command"
                 in="${scripts_dir}/list_samples.sh ${working_dir}"
                 as="sample_name"
                 src="tasks/sort_bam.sh"/>
    </tasks>

    <metadata>
        <task-description id="align-reads">Aligns reads to the reference genome.</task-description>
        <task-description id="prepare-reference">Builds the genome index.</task-description>
    </metadata>

</pipeline>
```

### `<param>` and `<flag>` attributes

| Attribute      | Applies to    | Description                                                              |
|:---------------|:--------------|:-------------------------------------------------------------------------|
| `name`         | both          | Unique identifier; letters, digits, underscores; no leading digit        |
| `shortName`    | both          | Shorter alternative name passed on the CLI                               |
| `defaultValue` | `<param>` only | Default value; omit to make the param mandatory                         |
| `global`       | both          | If `"true"`, automatically injected into every task as an env var        |

### `<task>` and `<foreach>` attributes

| Attribute     | Mandatory | Description                                                                   |
|:--------------|:----------|:------------------------------------------------------------------------------|
| `id`          | yes       | Unique task identifier                                                        |
| `after`       | no        | Comma/space-separated prerequisite task IDs; prefix with `*` for iteration-level dependency in `foreach` |
| `params`      | no        | Space-separated non-global param names to inject as env vars into this task   |
| `src`         | no        | Path (relative to the pipeline XML file) to the external script for this task |
| `if`          | no        | Bash command; task is skipped if exit code is non-zero                        |
| `interpreter` | no        | Bash expression to wrap task code; receives `$task_code`                      |
| `of`          | foreach   | Collection type: `list`, `range`, `file`, `param`, `command`                  |
| `in`          | foreach   | The list literal, directory, parameter name, or shell command to enumerate    |
| `as`          | foreach   | Variable name for the current iteration value                                 |

### Runners

Runners redirect task execution to a custom script instead of running the task directly as a local process. They are defined in a separate XML file and passed to `compi run` via `-r runner.xml`. Runners are the extension point for Docker, Slurm, SSH, and cloud execution.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<runners xmlns="http://sing-group.org/compi/runners-1.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

    <!-- Run a specific task inside a Docker container -->
    <runner tasks="align-reads">
        envs=$(for param in $task_params; do echo -n "-e $param "; done)
        docker run --rm $envs \
            -v ${working_dir}:${working_dir} \
            --entrypoint /bin/bash pegi3s/hisat2:2.0.5 \
            -c "${task_code}"
    </runner>

</runners>
```

Variables available inside runner code: `task_id`, `task_code`, `task_params`, `i` (foreach iteration value), and all pipeline params as env vars.

### Parameter file format

```bash
# Mandatory params
working_dir=/path/to/analysis/working-dir
genome_fasta=genome/GRCh38.fa
reference_annotation=genes/GRCh38.gtf

# Optional params (uncomment to activate)
#counts_method=htseq
#min_mapping_quality=30
#batch_correction=batch_column

# Flags: activated by presence alone (no value needed)
keep_intermediate_files
```

Passed to the pipeline via `compi run -pa params` or `-pa compi.parameters`.

### Key CLI commands

| Command | Purpose |
|:--------|:--------|
| `compi run -p pipeline.xml -pa params` | Run the full pipeline |
| `compi run -pa params` | Run (when file is named `pipeline.xml`) |
| `compi run ... -r runner.xml` | Use custom runners |
| `compi run ... -o` | Show task stdout in console |
| `compi run ... -l /dir` | Save per-task logs to directory |
| `compi run ... -num-tasks N` | Maximum parallel tasks |
| `compi run ... -st task-id` | Run a single task only |
| `compi run ... -ut task-id` | Run up to and including a task (with all its dependencies) |
| `compi run ... -bt task-id` | Run all dependencies of a task, excluding the task itself |
| `compi run ... -f task-id` | Start from a task |
| `compi run ... -a task-id` | Start after a task (the named task is excluded) |
| `compi resume -p pipeline.xml` | Resume an interrupted execution |
| `compi validate -p pipeline.xml` | Validate pipeline XML |
| `compi export-graph -p pipeline.xml -o pipeline.png -f png` | Export DAG as image |

Always run `compi validate -p pipeline.xml` after editing the pipeline XML to confirm structural correctness.

### Two project layout conventions

Both conventions are actively used. Detect which one is in use before diagnosing or modifying a pipeline.

| Convention | When encountered | Pipeline file | Task scripts |
|:-----------|:-----------------|:--------------|:-------------|
| **Legacy / standard** | Most existing Compi projects | `pipeline.xml` at repo root | `tasks/` and/or `scripts/` at repo root |
| **New-style** | New developments (recommended in paper) | `private/pipelines/main.xml` | `private/scripts/`, `private/env/shared/`, etc. |

**Detection rule:** if `pipeline.xml` exists at the repository root, assume the legacy layout. If `private/pipelines/main.xml` exists, assume the new-style layout.

---

## SOLID principles for Compi pipelines

The SOLID principles were originally formulated for object-oriented software design. In Compi pipelines, they are used as a practical design vocabulary for reasoning about task interfaces, coupling, extension points, substitutability, and separation of concerns. The mapping is interpretive and heuristic, not a claim of strict object-oriented equivalence.

### Task interfaces

Before applying SOLID reasoning to a Compi pipeline, define each task as a black box with an explicit interface. In practice, a task interface has three parts:

- **Inputs:** the files and formats the task consumes
- **Outputs:** the files and formats the task produces
- **Parameters:** the settings that control its behaviour

Parameter design should distinguish between task-level parameters, which control the task's own logic, and tool-level parameters, which are forwarded directly to the underlying software.

This interface should be clear before implementation. Ask: if this task were reimplemented with a different tool, language, or script behind `src=`, could it preserve the same inputs, outputs, and parameter semantics? If yes, the interface is well-defined, and the pipeline can evolve without changing its structure.

### Single Responsibility Principle (SRP)

**Pipeline-level interpretation:** A task should have one and only one reason to change. Its scope must be bounded by a single, clearly named responsibility.

**Compi-specific manifestation:**
- Each `<task>` or `<foreach>` element should correspond to exactly one analytical or operational step.
- Glue logic (output renaming, format conversion, file moves, metadata transforms) must not be embedded inside compute tasks. It must be extracted into dedicated tasks with their own `src=` scripts.
- A task that is hard to name with a single verb and a single object (e.g., `align-reads`, `sort-bam`, `count-features`) likely violates SRP.

**Anti-pattern to look for:**
```xml
<!-- BAD: one task mixes alignment, SAM-to-BAM conversion, sorting, and output renaming -->
<task id="align-and-process" after="prepare-reference">
    hisat2 ... -S ${working_dir}/tmp/out.sam
    samtools view -bS ${working_dir}/tmp/out.sam > ${working_dir}/tmp/out.bam
    samtools sort ${working_dir}/tmp/out.bam -o ${working_dir}/alignments/${sample}.sorted.bam
    mv ${working_dir}/alignments/${sample}.sorted.bam ${working_dir}/results/${sample}_final.bam
</task>
```

**Recommended design move:**
```xml
<!-- GOOD: each step is a separate task with its own src= script -->
<foreach id="align-reads"    after="prepare-reference" ... src="tasks/align_reads.sh"/>
<foreach id="convert-to-bam" after="*align-reads"      ... src="tasks/convert_to_bam.sh"/>
<foreach id="sort-bam"       after="*convert-to-bam"   ... src="tasks/sort_bam.sh"/>
```

**Compute tasks vs. glue tasks:**
- *Compute tasks* directly invoke third-party bioinformatics tools (`align-reads`, `count-features`, `run-fastqc`). They are candidates for reuse and publication as community modules.
- *Glue tasks* perform format conversions, renamings, merges, or pre/post-processing (`convert-sam-to-bam`, `join-counts`, `prepare-dea-dirs`). They are inherently pipeline-specific. Making them explicit and first-class is preferable to hiding their logic inside compute tasks.

### Open/Closed Principle (OCP)

**Pipeline-level interpretation:** A pipeline should be extendable without requiring structural modifications to its core. Growth happens through addition, not modification.

**Compi-specific manifestation:**
- Task implementation must live in external `src=` script files, not as inline XML body text. The pipeline XML describes *what* must be done and *in what order*; the scripts describe *how*.
- To support an alternative tool, create a new script file with the same I/O contract and update the `src=` attribute. The pipeline XML structure and all downstream tasks remain untouched.
- Runners follow the same pattern: each execution environment (Docker, Slurm, SSH) is a self-contained `<runner>` entry in `runner.xml`. Adding a new execution environment never requires touching `pipeline.xml`.

**Anti-pattern to look for:**
```xml
<!-- BAD: inline code — any change to the aligner requires editing pipeline.xml -->
<foreach id="align-reads" after="genome-index" of="command" in="..." as="sample_name">
    hisat2 --dta -x ${working_dir}/${genome_index_dir}/${genome_index} \
        -1 ${working_dir}/${samples_dir}/${sample_name}_1.fastq.gz \
        -2 ${working_dir}/${samples_dir}/${sample_name}_2.fastq.gz \
        -S ${working_dir}/${samples_alignment_dir}/${sample_name}.sam
</foreach>
```

**Recommended design move:**
```xml
<!-- GOOD: implementation is external; swapping the aligner = new script + update src= -->
<foreach id="align-reads"
         after="genome-index"
         of="command" in="${scripts_dir}/list_samples.sh ${working_dir}" as="sample_name"
         src="tasks/align_reads.sh"/>
```

To switch to a different aligner: create `tasks/align_reads_bwamem2.sh` (same I/O contract), change `src="tasks/align_reads_bwamem2.sh"`. No other file changes required.

### Liskov Substitution Principle (LSP)

**Pipeline-level interpretation:** When two tasks fulfil the same analytical role, they must be interchangeable without altering the correctness of the rest of the pipeline. This requires that their output formats, file naming conventions, and parameter contracts be mutually consistent.

**Compi-specific manifestation:**
- Pipelines that offer alternative implementations of a step (e.g., `htseq-count` vs. `feature-counts`, `bowtie2` vs. `bwa-mem2`) must ensure that both produce outputs with the same schema: same file names, same column structure, same format conventions.
- Downstream tasks must depend on the interface contract, not on which implementation was used.
- Because Compi does not enforce task interfaces at the XML level, this is a documentation and testing discipline: write down the expected output schema for every task that has alternatives, and verify both implementations against it.

**Anti-pattern to look for:**
```
htseq-count  produces: ${sample}.tsv  (columns: gene_id, count)
feature-counts produces: ${sample}.txt (columns: Geneid, Chr, Start, End, Strand, Length, count)
→ downstream join-all task breaks when counts_method=featurecounts
```

**Recommended design move:** both implementations must emit the same output schema. A glue task (`feature-counts-postprocessing`) normalises the `featureCounts` output to match the `htseq-count` schema before any downstream task consumes it. The downstream task sees only the normalised format regardless of which counter was used.

### Interface Segregation Principle (ISP)

**Pipeline-level interpretation:** Tasks should depend only on the inputs, outputs, and parameters they actually require.

**Compi-specific manifestation:**
- `global="true"` should be reserved for parameters that are genuinely needed by every (or almost every) task: `working_dir`, container image names that are used widely, shared infrastructure paths.
- Task-specific parameters must be declared without `global="true"` and listed explicitly in the `params=` attribute of the tasks that use them. This makes each task's dependencies explicit and minimal.
- A task that reads `${genome_fasta}` has no business seeing `${rcpa_database}`. Forcing it to receive both via a global blob violates ISP.

**Anti-pattern to look for:**
```xml
<!-- BAD: every parameter is global, so every task sees the entire config blob -->
<param name="counts_method"               defaultValue="featurecounts" global="true"/>
<param name="rcpa_database"               defaultValue="KEGG"          global="true"/>
<param name="trimmomatic_parameters"      defaultValue="..."            global="true"/>
<param name="pathfindr_delite_file_prefix" defaultValue="..."           global="true"/>
```

**Recommended design move:**
```xml
<!-- GOOD: only infrastructure params are global; task-specific params are scoped -->
<param name="working_dir"      global="true"/>
<param name="samtools_image"   defaultValue="pegi3s/samtools_bcftools:1.22" global="true"/>

<param name="counts_method"    defaultValue="featurecounts"/>  <!-- not global -->
<param name="rcpa_database"    defaultValue="KEGG"/>            <!-- not global -->

<!-- Each task explicitly declares the params it uses -->
<task id="count-features" params="counts_method" src="tasks/count_features.sh"/>
<task id="run-rcpa"        params="rcpa_database" src="tasks/run_rcpa.sh"/>
```

### Dependency Inversion Principle (DIP)

**Pipeline-level interpretation:** Mismatches between tasks must be resolved through dedicated adapter (glue) tasks rather than by modifying compute tasks to accommodate each consumer. Compute tasks define the interface they expose; glue tasks absorb the coupling.

**Compi-specific manifestation:**
- When task A produces a format that task B cannot directly consume, insert a glue task between them. Do not change task A's script.
- Execution environment concerns (container images, Conda environments, resource directives) must be decoupled from analytical logic and managed via runner XML and parameter declarations, so that compute task scripts remain portable.
- A compute task script that contains `docker run ...` inline is mixing execution environment with analytical logic — this coupling belongs in a runner.

**Anti-pattern to look for:**
```xml
<!-- BAD: compute task modified to satisfy downstream format expectation -->
<task id="count-features" src="tasks/count_features.sh"/>
<!-- tasks/count_features.sh internally runs awk to reformat output
     because join-all.sh expects a specific column order — this is
     glue logic embedded inside a compute task -->
```

**Recommended design move:**
```xml
<!-- GOOD: glue task handles the adaptation; compute task is unchanged -->
<foreach id="count-features"
         after="*sort-bam"
         of="command" in="..." as="sample_name"
         src="tasks/count_features.sh"/>

<foreach id="normalise-counts"
         after="*count-features"
         of="command" in="..." as="sample_name"
         src="tasks/normalise_counts.sh"/>   <!-- glue: reformats for join-all -->

<task id="join-all" after="normalise-counts" src="tasks/join_all.sh"/>
```

---

## Robustness recommendations

### Environment checks

Environment checks must run before any analytical task begins. Place them in a dedicated `<task id="check-environment">` that is a prerequisite of all other tasks, or integrate them into the `initialization` task via shared helper functions.

What to check:

- **Docker / container manager availability:** most Compi pipelines use Docker runners; verify the daemon is accessible (`docker info` or `singularity --version`) before any runner-dependent task runs.
- **Resource availability:** disk space in `working_dir`, memory, GPU presence when required. Estimate requirements from the pipeline's most demanding tasks.
- **Permissions:** read access on all input files and directories; read-write access on `working_dir` and its subdirectories.
- **File system compatibility:** some tools fail on FAT32/exFAT (file size limits, symlink support). Warn or abort if the working directory resides on an incompatible filesystem.
- **Binary and tool dependencies:** verify that any tool invoked outside a container (e.g., `samtools`, `python`) is present and meets the minimum version requirement.

In new-style pipelines, reusable guard functions live in `private/env/shared/` and are sourced by `checks.sh`. In legacy pipelines, shared functions typically live in `tasks/functions.sh` and are sourced by `initialization.sh`.

### Input parameter checks

Input parameter checks must run early — either in the `check-environment` / `check-inputs` task or within `initialization`. When ISP is applied (per-task param scopes), each parameter slice is smaller and cheaper to validate in isolation.

What to check:

- **Type and existence:** numeric params are within valid ranges; file/directory params point to existing paths; file content matches the declared format (do not rely on extensions alone — check file headers or run a format validator).
- **IN constraints (fixed allowed values):** params that must take a value from a closed set. Example: `counts_method` must be `htseq` or `featurecounts`; anything else should abort with a clear message.
- **OR constraints (mutually exclusive params):** when two params represent alternative ways to specify the same input (e.g., `input_fasta` vs. `input_fasta_dir`), their simultaneous presence must trigger an error or explicit warning.
- **NOT constraints (resource request vs. available hardware):** a request for more parallel tasks than available CPUs, or for GPU mode on a CPU-only node, must be rejected before execution starts.

### Monitoring checks

Monitoring checks run during execution, within or immediately after tasks that have non-trivial failure modes.

What to check:

- **Expected output file presence:** before a downstream task begins, assert that its required inputs actually exist. Use `if=` guards in Compi or explicit `test -f` / `test -s` assertions at the start of each script.
- **File count tracking in `foreach` tasks:** log the number of input items and the number of output files produced. A mismatch (e.g., 10 samples in but 9 output BAMs) indicates a silent partial failure.
- **Biological expectation guards:** if a task produces an empty result that should not normally be empty (e.g., no alignments after HISAT2, no DEGs after DElite), emit a warning and optionally skip downstream tasks using `if=`.

---

## Standard project layout

### Legacy layout (most existing Compi projects)

```
repo-root/
├── pipeline.xml                  # main pipeline definition
├── runner.xml                    # optional custom runners
├── hub.metadata                  # optional Compi Hub metadata
├── README.md
├── image-files/                  # files downloaded by compi-dk
│   └── compi                     # compi binary
├── tasks/                        # external task scripts (src= targets)
│   ├── functions.sh              # shared Bash helper functions
│   ├── initialization.sh
│   └── <task-name>.sh
├── scripts/                      # helper scripts called from task code
│   ├── list_samples.sh
│   └── *.py / *.R
└── resources/
    └── init-working-dir/
        ├── parameters            # example parameter file for users
        ├── run.sh                # main launch script
        └── run-<variant>.sh      # partial-execution launch scripts
```

### New-style layout (recommended for new developments)

```
repo-root/
├── pipeline-cli                  # user-facing entry point
├── README.md
├── params-examples/              # example parameter files
└── private/
    ├── pipelines/
    │   └── main.xml              # main pipeline definition
    ├── scripts/                  # task scripts + initialization.sh + checks.sh
    ├── runners/                  # runner scripts
    ├── env/
    │   ├── config/               # environment.json, containers.json
    │   └── shared/               # reusable check libraries (sourced by checks.sh)
    └── vendors/
        └── containers/
            └── containers.json   # container image registry with pinned tags
```

### Working directory output layout (both conventions)

```
working-dir/
├── compi.parameters              # exact parameter file used (reproducibility pack)
├── run.sh                        # invocation script used (reproducibility pack)
└── compi/
    ├── logs/
    │   └── <timestamp>/
    │       └── tasks/            # per-task stdout and stderr logs
    └── <task-intermediate-dirs>/ # one subdirectory per task for intermediate outputs
```

Output results are written to `working-dir/output/<task-name>/` (new-style) or to task-specific subdirectories under `working-dir/compi/` (legacy).

---

## Reproducibility practices

- **Version control:** record the pipeline version in the `<version>` tag; tag Git releases; the working directory must capture the exact Git tag or commit SHA used for each analysis.
- **Pinned container images:** every `<param ... defaultValue="image:tag"/>` that references a container image must use an explicit, non-moving tag (never `latest`). In new-style pipelines, all images are registered in `containers.json` with pinned tags and verified at initialisation.
- **Reproducibility pack:** the working directory must always contain: the exact `compi.parameters` file used, the `run.sh` invocation script, and a record of the pipeline version. Input data must be identified by dataset release version, accession identifier, or checksum — not by a mutable file path alone.
- **End-to-end tests:** the full pipeline must be runnable on a small real or synthetic dataset with known expected outputs. These tests should be executable in CI to detect regressions whenever pipeline code, parameter defaults, or container images change.

### Versioning

- Pipeline versions are defined in `<version>` inside the `pipeline.xml`. These follow semantic versioning so make sure the correct numbers are changed when making changes in a pipeline.

---

## Test environment

### Detection

When working with a Compi pipeline, search for test infrastructure in this order:

1. `tests/` at the repository root
2. `private/tests/`
3. `private/e2e-tests/`
4. Any directory whose name contains `test` or `e2e`
5. Partial-execution runner scripts (e.g., `resources/init-working-dir/run-<task>-1.sh`) — these are informal test surrogates that indicate which pipeline segments have been run in isolation

If no dedicated test directory exists, document it as a diagnosis finding and create a minimal one as part of the improvement workflow.

### Recommended test layout

```
tests/
├── tests_config.json         # metadata: name, description, estimated runtime, last-run timestamp
└── <test-name>/              # one subdirectory per test case; each is a self-contained working dir
    ├── compi.parameters      # parameter file for this test
    ├── run.sh                # invocation script for this test
    ├── input/                # minimal input data (real subset or synthetic)
    └── expected/             # expected output files or a checksums manifest
```

### Synthetic data tools

| Tool | Purpose |
|:-----|:--------|
| ART | Simulate Illumina sequencing reads (FASTQ) from a FASTA reference |
| SeqModeller | Simulate mutations, insertions, and repetitions in FASTA genomes |
| Seq-Gen | Generate sequences from evolutionary models and phylogenies |

Synthetic data libraries should be versioned and maintained with the same modularity and reuse discipline as the pipeline code itself.

### Partial execution for development testing

Use Compi's partial execution flags to test individual pipeline segments without running the full workflow:

```bash
# Test only up to and including sort-bam (with all its dependencies)
compi run -pa compi.parameters -ut sort-bam -o

# Re-run only a single task after fixing its script
compi run -pa compi.parameters -st count-features -o

# Test a pipeline subpath
compi run -pa compi.parameters -f align-reads -ut join-all -o
```

If the project uses `compi-dk`, then build the Docker image with `compi-dk build --tag-version -drd` and then run Compi using this Docker image.

---

## Diagnosis workflow

Follow these steps in order to produce a structured assessment of an existing Compi pipeline.

### Step 1 — Detect layout convention

Check for the presence of `pipeline.xml` at the repository root (legacy layout) vs. `private/pipelines/main.xml` (new-style layout). Note which convention is in use, as it determines where scripts, runners, check libraries, and test directories are expected to be found.

### Step 2 — Parse the pipeline XML

Read `pipeline.xml` (or `private/pipelines/main.xml`). List all `<task>` and `<foreach>` elements with their `id`, `after`, `src`, `params`, and `if` attributes. Build a mental model of the task DAG.

### Step 3 — Check each task against SOLID principles

For every task:

- **SRP:** Does the task combine multiple analytical responsibilities? Does its script (or inline body) contain glue logic (renaming, format conversion, file moves) mixed with compute logic?
- **OCP:** Does the task use `src=` pointing to an external script, or does it have inline code? Inline code means any change to the task requires editing `pipeline.xml`.
- **LSP:** If the task is one of several alternatives for the same role (e.g., `htseq-count` and `feature-counts`), do all alternatives produce the same output schema?
- **ISP:** Are any params declared `global="true"` that are only used by a subset of tasks? Does the task list its non-global params explicitly in `params=`?
- **DIP:** Does the task contain format conversion or adaptation logic that exists solely to satisfy a downstream consumer? This logic should be in a glue task.

### Step 4 — Check robustness

- Is there a `check-environment` or `initialization` task that verifies Docker availability, disk space, permissions, and key dependencies before analysis begins?
- Is there a `check-inputs` task or equivalent guards that validate parameter types, formats, IN/OR/NOT constraints?
- Do high-risk tasks assert expected output file presence before downstream tasks begin? Do `foreach` tasks track input/output file counts?

### Step 5 — Check reproducibility

- Are all container image params pinned to explicit version tags (no `latest`)?
- Does the working directory layout capture `compi.parameters` and the run script?
- Is the pipeline version recorded in `<version>`?

### Step 6 — Check test environment

- Is a test directory present? If so, does it contain at least one complete test case with input data, a parameter file, a run script, and expected outputs?
- If no test directory exists, are there partial-execution runner scripts that serve as informal surrogates?

### Step 7 — Produce the diagnosis table

Present findings in the following format:

| File | Issue | Principle / Recommendation | Suggestion |
|:-----|:------|:---------------------------|:-----------|
| `pipeline.xml` | Task `align-reads` has inline code mixing alignment and SAM output renaming | SRP, OCP | Extract inline body to `tasks/align_reads.sh`; extract renaming logic to a new `rename-alignment` task with its own script |
| `pipeline.xml` | `counts_method`, `rcpa_database`, and `trimmomatic_parameters` are all `global="true"` | ISP | Remove `global="true"` from task-specific params; list them in the `params=` attribute of the tasks that use them |
| `pipeline.xml` | No `check-environment` task | Robustness — environment checks | Add `<task id="check-environment">` before `initialization`; verify Docker availability, disk space, and input permissions |
| `pipeline.xml` | `feature-counts` and `htseq-count` produce different output file formats; `join-all` only handles one | LSP | Normalise both outputs to the same schema via a `normalise-counts` glue task |
| `pipeline.xml` | Task `align-and-count` combines genome indexing, alignment, SAM sorting, and count generation | SRP | Split into `index-genome`, `align-reads`, `sort-bam`, `count-features` |
| `tasks/count_features.sh` | Script runs `awk` reformat internally to satisfy `join-all` column expectations | DIP | Move the `awk` reformat into a dedicated `normalise-counts` task; compute task should not adapt its output for consumers |
| `resources/init-working-dir/parameters` | Container image param `hisat2_image` uses `latest` tag | Reproducibility | Pin to a specific version, e.g. `pegi3s/hisat2:2.0.5` |
| *(none found)* | No test directory | Test environment | Create `tests/<test-name>/` with `compi.parameters`, `run.sh`, input data, and an `expected/` manifest |

---

## Improvement workflow

Apply changes in the following order. Run `compi validate -p pipeline.xml` after every structural change to the XML and end-to-end tests if they exist.

### 1. SRP violation — extract inline code to external script

1. Create `tasks/<task-id>.sh` (legacy) or `private/scripts/<task-id>.sh` (new-style).
2. Move the inline task body verbatim into the new script.
3. Add `src="tasks/<task-id>.sh"` to the `<task>` element and remove the inline body.
4. Run `compi validate`.

### 2. SRP violation — split mixed responsibilities

1. Identify the distinct responsibilities in the task.
2. For each new responsibility, create a new `<task id="...">` element in the XML and a corresponding script.
3. Update `after=` dependencies to form the correct sequence.
4. Remove the original monolithic task and its script (or archive it).
5. Run `compi validate`.

### 3. OCP violation — support an alternative tool

1. Create `tasks/<task-id>-<variant>.sh` implementing the same I/O contract as the original.
2. Update `src=` on the `<task>` element to point to the new file.
3. Do not delete or modify the original script — it can coexist as an alternative.
4. Document the I/O contract so future variants know what they must honour.

### 4. LSP violation — align alternative implementations

1. Define the canonical output schema for the task role (file name pattern, column names, format).
2. Audit each alternative implementation against this schema.
3. For any implementation that diverges, add a normalisation glue task immediately after it.
4. Ensure downstream tasks depend only on the normalised output, not on implementation-specific outputs.

### 5. ISP violation — scope parameter visibility

1. Identify every `global="true"` param that is not genuinely needed by all tasks.
2. Remove `global="true"` from those params.
3. Add the param names to the `params=` attribute of each task that uses them.
4. Run `compi validate` and execute a test run to confirm env vars are correctly injected.

### 6. DIP violation — introduce a glue task

1. Identify the adaptation logic embedded in a compute task (format conversion, column reorder, naming fix).
2. Create a new `<task id="adapt-...">` in the XML and a corresponding script containing only the adaptation logic.
3. Insert the new task between the compute task and its downstream consumer in the `after=` chain.
4. Remove the adaptation logic from the compute task script.

### 7. Missing environment checks

1. Locate the earliest task in the pipeline (typically `initialization` or a dedicated `check-environment`).
2. Add guard assertions for Docker availability, disk space, permissions, and required binary dependencies.
3. In new-style pipelines, place these in `private/scripts/checks.sh` using shared functions from `private/env/shared/`. In legacy pipelines, source `tasks/functions.sh` and add checks to `initialization.sh`.

### 8. Missing input parameter checks

1. Add a `check-inputs` task (or expand `initialization`) with explicit guards for each user-facing parameter.
2. For IN constraints, use a case statement or `grep -wq` check against the allowed value list; exit with a clear error message on violation.
3. For OR constraints, check that mutually exclusive params are not both set.
4. For NOT constraints, compare the requested resource count against detected hardware capacity.

### 9. Reproducibility gaps

1. Pin all container image params to explicit version tags.
2. Ensure that every analysis run records `compi.parameters` and the invocation script in the working directory.
3. Add input data provenance (accession IDs, checksums, or release versions) to the working directory or to a `data-manifest.txt`.
4. If using new-style layout, register all images in `containers.json` and add download-and-verify logic to `initialization`.

---

## Verification workflow

After making changes, verify that the pipeline still produces correct outputs.

If the project uses `compi-dk`, then build the Docker image with `compi-dk build --tag-version -drd` and then run Compi using this Docker image.

### Step 1 — Validate XML

```bash
compi validate -p pipeline.xml
```

Fix any reported errors before proceeding.

### Step 2 — Locate or create a test environment

Search for test directories in this order: `tests/`, `private/tests/`, `private/e2e-tests/`, any `*test*` or `*e2e*` directory. If none exists, create a minimal test case:

```
tests/smoke/
├── compi.parameters    # minimal parameter file
├── run.sh              # compi run -pa compi.parameters -o -l logs
└── input/              # smallest possible real or synthetic input data
```

### Step 3 — Run the test

Execute the test's `run.sh`, or invoke Compi directly:

```bash
compi run -pa tests/smoke/compi.parameters -o -l tests/smoke/logs -num-tasks 4
```

To test only the tasks affected by your change, use partial execution:

```bash
# Re-run only the modified task and everything downstream from it
compi run -pa tests/smoke/compi.parameters -f <modified-task-id> -o
```

### Step 4 — Verify outputs

Compare actual outputs against the `expected/` manifest:

```bash
# File presence check
diff <(ls tests/smoke/expected/) <(ls tests/smoke/output/)

# Checksum check
md5sum --check tests/smoke/expected/checksums.md5
```

### Step 5 — Inspect logs on failure

Per-task logs are written to the directory specified with `-l`. For each failing task:

1. Read `<log-dir>/<task-id>.out` and `<task-id>.err`.
2. Map the failure to the responsible task.
3. Cross-reference with the diagnosis table to determine whether the failure is related to an unresolved design issue.
4. Fix the script or XML, re-validate, and re-run.

### Step 6 — Document the test case

Update `tests/tests_config.json` with the new or modified test case:

```json
{
  "tests": [
    {
      "name": "smoke",
      "description": "End-to-end run on minimal synthetic data",
      "estimated_runtime_minutes": 5,
      "last_run": "2026-06-01"
    }
  ]
}
```
