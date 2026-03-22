# Pipeline Orchestration Guidelines

This document defines how to design multi-step processing pipelines for CLI tools, covering step management, state persistence, caching, and error recovery.

---

## 1. Pipeline Architecture

## 1.1 Core Concepts

A pipeline is a sequence of **steps** that transform input data into output:

```text
Input → [Step 1: Collect] → [Step 2: Process] → [Step 3: Transform] → [Step 4: Output] → Result
```

Each step:

- Has a **name** (identifier) and **label** (display text)
- Receives a **config** and **work directory**
- Reads input from the work directory (previous step's output)
- Writes output to the work directory
- Is independently testable

## 1.2 Step Protocol

```python
from typing import Protocol
from pathlib import Path

class PipelineStep(Protocol):
    name: str
    label: str

    def execute(self, config: Settings, work_dir: Path) -> None: ...
```

Or using a dataclass:

```python
from dataclasses import dataclass
from typing import Callable

@dataclass
class Step:
    name: str
    label: str
    execute: Callable[[Settings, Path], None]
```

## 1.3 Orchestrator Pattern

The orchestrator manages step registration, execution order, state persistence, and error handling:

```python
class PipelineOrchestrator:
    def __init__(self, config: Settings) -> None:
        self.config = config
        self.steps: list[PipelineStep] = []

    def register_steps(self, steps: list[PipelineStep]) -> None:
        self.steps = steps

    def run(self, resume_modules: list[str] | None = None) -> list[StepResult]:
        state = self._load_state() or create_initial_state()
        results = []
        for step in self.steps:
            if not resume_modules and step.name in state.completed_steps:
                continue
            result = self._execute_step(step, state)
            results.append(result)
        return results

    def _load_state(self) -> PipelineState | None: ...
    def _save_state(self, state: PipelineState) -> None: ...
```

---

## 2. Work Directory Structure

Each pipeline run uses a dedicated work directory for intermediate files:

```text
.work/
├─ pipeline-state.json       # Pipeline execution state
├─ cache-meta.json           # Cache metadata
├─ raw/                      # Step 1 output: raw collected data
│  ├─ screenshots/
│  └─ *.json
├─ structured/               # Step 2 output: structured/parsed data
│  └─ *.json
├─ intermediate/             # Step 3 output: enriched/transformed data
│  └─ *.json
└─ reports/                  # Step 4 output: final deliverables
   └─ *.md
```

### Rules

- Each step writes to its own subdirectory or well-known file
- Steps read from previous step's output location
- The work directory is the **single source of truth** for pipeline state
- Initialize all required subdirectories at pipeline start

---

## 3. State Management

## 3.1 Pipeline State

Persist execution state to enable resume after interruption:

```python
from pydantic import BaseModel
from datetime import datetime

class StepResult(BaseModel):
    step: str
    success: bool
    duration: float  # seconds
    error: str | None = None

class PipelineState(BaseModel):
    completed_steps: list[str] = []
    current_step: str | None = None
    results: list[StepResult] = []
    started_at: datetime = datetime.now()
```

## 3.2 State Lifecycle

1. **Before each step**: Set `current_step`, save state
2. **After successful step**: Add to `completed_steps`, clear `current_step`, save state
3. **After failed step**: Record error in `results`, save state
4. **On resume**: Load state, skip completed steps

## 3.3 Resume Strategy

```python
def run(self, resume_modules: list[str] | None = None) -> list[StepResult]:
    state = self._load_state() or create_initial_state()

    for step in self.steps:
        # Skip completed steps (unless explicitly re-running)
        if not resume_modules and step.name in state.completed_steps:
            continue
        # Execute step...
```

---

## 4. Caching

## 4.1 Cache Strategy

Cache expensive operations (network requests, AI API calls) to avoid redundant work:

| Strategy | Use Case |
|----------|----------|
| Input-based hash | Cache by hashing the step's input data |
| TTL-based expiry | Invalidate after configurable duration (e.g., `24h`) |
| Manual invalidation | `--no-cache` flag to force re-execution |

## 4.2 Cache Key Design

```python
import hashlib

def cache_key(step_name: str, input_hash: str) -> str:
    return hashlib.sha256(f"{step_name}:{input_hash}".encode()).hexdigest()
```

- Include step name to avoid collisions
- Hash the input content (not file paths) for portability
- Use SHA-256 for content hashing

## 4.3 TTL Parsing

Support human-readable durations:

```python
import re

def parse_ttl(ttl: str) -> int:
    """Parse TTL string to milliseconds."""
    match = re.match(r"^(\d+)(ms|s|m|h|d)$", ttl)
    if not match:
        raise ValueError(f"Invalid TTL: {ttl}")
    value, unit = int(match.group(1)), match.group(2)
    multipliers = {"ms": 1, "s": 1000, "m": 60_000, "h": 3_600_000, "d": 86_400_000}
    return value * multipliers[unit]
```

---

## 5. Error Handling in Pipelines

## 5.1 Error Wrapping

Wrap raw errors with step-specific error types:

```python
def wrap_step_error(step_name: str, error: Exception) -> AppError:
    if isinstance(error, AppError):
        return error
    msg = str(error)
    error_map = {
        "collect": CollectionError,
        "preprocess": PreprocessError,
    }
    error_class = error_map.get(step_name, AppError)
    return error_class(msg)
```

## 5.2 Graceful Degradation

- Non-critical steps (e.g., report generation) should not abort the entire pipeline
- Log warnings and continue where possible
- Record partial results in state

```python
try:
    step.execute(config, work_dir)
except Exception as e:
    state.results.append(StepResult(step=step.name, success=False, error=str(e), duration=elapsed))
    if step.name != "report":
        logger.warning("Error occurred, continuing with remaining steps")
```

## 5.3 Validation Between Steps

Validate the output of each step before the next step consumes it:

```python
## Before step 2
raw_data = json.loads(collected_path.read_text())
data, warnings = validate_collected_data(raw_data)
for w in warnings:
    logger.warning(w)
```

---

## 6. Manual Review Points

For pipelines that benefit from human review of intermediate results:

```python
if config.pause and sys.stdin.isatty():
    logger.info("Intermediate files generated. Review and edit:")
    logger.info(f"  {work_dir}/intermediate/")
    logger.info("Press Enter to continue, or resume later with:")
    logger.info(f"  my-tool resume --workdir {work_dir}")
    input()  # Wait for Enter
```

- Only prompt in interactive mode
- Allow `--no-pause` to skip for CI
- Provide the resume command for later continuation

---

## 7. Progress Reporting

Display step-based progress during execution:

```python
logger.info(f"[{step_num}/{total_steps}] {step.label}...")
## Output: [2/4] Preprocessing...
```

After completion, show a summary:

```python
for r in state.results:
    status = "OK" if r.success else "FAIL"
    logger.info(f"  [{status}] {r.step:15s} ({r.duration:.1f}s)")
```

---

## 8. Summary

- **Pipeline = ordered sequence of independent steps**
- **Work directory is the single source of truth** for all intermediate data
- **Persist state after every step** to enable resume
- **Cache expensive operations** with input-based hashing and TTL
- **Wrap errors per step** and support graceful degradation
- **Validate output between steps** to catch issues early
- **Support manual review points** for human-in-the-loop workflows
