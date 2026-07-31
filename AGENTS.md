# AGENTS.md

## Project mission

`video-helper` is a local-first video processing tool. The immediate product goal is P0:

> Import one long recording, detect and remove or compress non-speech gaps, generate subtitles, remap subtitle timing after cuts, and export an auditable rough-cut video.

Read these files before making changes:

1. `README.md`
2. `docs/PRODUCT_SPEC.md`
3. `docs/ARCHITECTURE.md`
4. `ROADMAP.md`

## Current priority

P0 is the only implementation priority until its end-to-end acceptance criteria pass.

Do not begin P1, P2, or P3 implementation merely because those features are more interesting. Later-stage work may only add interface placeholders, research notes, or narrowly scoped experiments that do not delay P0.

## Product boundary for P0

P0 includes:

- importing a long recording;
- media validation and proxy generation;
- audio extraction;
- voice activity and silence analysis;
- configurable silence preservation, compression, or removal;
- ASR with segment and preferably word timestamps;
- an explicit edit decision list;
- source-to-output timeline mapping;
- subtitle remapping and SRT/ASS generation;
- preview and final rendering;
- QC reports;
- resumable CLI execution.

P0 does not include:

- semantic narrative restructuring;
- automatic removal of digressions or weak takes;
- B-roll selection;
- knowledge graph animation;
- script-to-video generation;
- autonomous multimodal directing.

## Architectural invariants

### 1. The source media is immutable

Never modify, rewrite, rename, or delete the user's source video as part of normal processing.

All outputs must be derived artifacts inside a project directory.

### 2. Decisions and execution are separate

Detection modules produce evidence.

The planner produces a structured `cut_plan.json`.

The renderer executes the plan.

No detector or language model may directly rewrite media files.

### 3. The original timeline is canonical

VAD, ASR, and raw annotations use source-media time.

A dedicated timeline mapper converts source time to output time after edits.

Subtitles, chapters, and future visual tracks must use that mapping rather than independently recomputing offsets.

### 4. Internal time is integer-based

Use a single integer time unit internally, preferably microseconds or nanoseconds.

Do not use floating-point seconds as the sole representation for edit boundaries or accumulated timeline calculations.

### 5. Every automatic decision is auditable

A decision must contain, where applicable:

- source start and end;
- action;
- reason;
- confidence;
- detector or rule source;
- configuration snapshot or version;
- lock state.

### 6. Every destructive-looking operation is reversible

A dropped segment is represented in the edit plan. It remains recoverable from the immutable source.

Automatic reruns must preserve user-locked decisions.

### 7. Modules communicate through versioned data contracts

Prefer explicit typed models and JSON schemas over implicit in-memory coupling.

Schema changes must be versioned and documented.

### 8. Long jobs are resumable

Each step records:

- dependency hashes;
- configuration hash;
- tool versions;
- state;
- outputs;
- errors.

A failed later step must not discard valid earlier results.

### 9. Local-first is the default

Large media files remain local unless the user explicitly enables a cloud-backed module.

Any future cloud operation must disclose which data will be transmitted.

### 10. CLI is a first-class interface

Core processing logic must not depend on a GUI.

A future UI should call the same application services and read the same project artifacts as the CLI.

## Platform requirements

Windows is the first required desktop environment.

Also preserve compatibility with Ubuntu and macOS.

Pay particular attention to:

- Unicode and Chinese paths;
- paths containing spaces;
- long Windows paths;
- subprocess argument quoting;
- FFmpeg discovery;
- optional GPU backends with CPU fallback.

Do not require Bash for core tasks.

## Implementation style

- Use a `src/` package layout.
- Use type hints for public and internal interfaces.
- Use validated configuration and data models.
- Keep subprocess calls in a small adapter layer.
- Pass subprocess arguments as arrays, not interpolated shell strings.
- Use structured logs in addition to human-readable CLI output.
- Return meaningful exit codes.
- Keep modules small and single-purpose.
- Add tests with every non-trivial behavior change.
- Document non-obvious timeline math and FFmpeg filter construction.
- Prefer deterministic rules for media execution.
- Treat AI or probabilistic model output as evidence or suggestions with confidence.

## Error handling

Errors must identify:

- the failed stage;
- the project path;
- the relevant input or output;
- the underlying tool or model;
- whether the task can be resumed;
- the next corrective action when known.

Never silently fall back in a way that changes output semantics. Record any backend or encoder fallback in project metadata and logs.

## Testing requirements

At minimum, maintain tests for:

- interval merging;
- padding and clipping at media boundaries;
- full timeline coverage by the cut plan;
- overlap rejection;
- keep/compress/drop duration calculations;
- source-to-output mapping;
- output-to-source mapping;
- subtitle remapping;
- subtitle split and merge rules;
- cache invalidation;
- configuration precedence;
- Unicode and spaced paths.

Use synthetic media fixtures for deterministic integration tests. Keep larger real-media samples out of Git unless their license and size are appropriate.

## Definition of done for a task

A task is complete when:

1. the requested behavior is implemented;
2. relevant tests are added or updated;
3. tests pass locally;
4. CLI errors and logs are useful;
5. documentation reflects any behavior or schema change;
6. no unrelated later-stage feature has been introduced;
7. generated artifacts and caches are excluded from source control.

## Roadmap discipline

Work through `ROADMAP.md` in dependency order.

For each completed milestone:

- update its status;
- record significant design decisions;
- add or update acceptance tests;
- leave the repository in a runnable state.

The first recommended implementation scope is **P0.0 only**:

- Python project skeleton;
- CLI entry point;
- validated configuration;
- project initialization;
- structured logging;
- FFmpeg diagnostics;
- test and CI skeleton.

Do not combine P0.0 with VAD, ASR, rendering, UI, or semantic editing in the same initial change unless the user explicitly requests a broader scope.

## Commit guidance

Prefer small, coherent commits.

Suggested prefixes:

- `feat:` user-visible capability
- `fix:` defect correction
- `refactor:` behavior-preserving internal change
- `test:` tests and fixtures
- `docs:` documentation
- `build:` packaging or dependency management
- `ci:` automation

## Documentation update rule

When implementation changes any of the following, update the corresponding document in the same change:

- product scope or acceptance criteria → `docs/PRODUCT_SPEC.md`
- module boundaries or data contracts → `docs/ARCHITECTURE.md`
- milestone state or execution order → `ROADMAP.md`
- developer constraints → `AGENTS.md`
- user-facing usage → `README.md`
