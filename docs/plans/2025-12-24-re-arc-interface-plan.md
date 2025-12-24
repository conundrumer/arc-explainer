# Re-ARC Dataset Generation & Verification Interface

**Author:** Claude Code using Sonnet 4.5
**Date:** 2025-12-24
**Purpose:** Plan for implementing a stateless re-arc dataset generation and verification interface

## Overview

Create a web interface that allows users to:
1. Generate re-arc datasets on demand
2. Submit solutions for verification
3. Verification works via XOR of task IDs to regenerate the seed (stateless, no DB needed)

## Data Structures

```typescript
// Core types
type IOGrid = {
  input: number[][];
  output: number[][];
};

type Challenge = {
  train: IOGrid[];
  test: IOGrid[];
};

type ChallengeDataset = {
  [taskId: string]: Challenge;
};

type Submission = {
  [taskId: string]: {
    attempt_1: number[][];
    attempt_2: number[][];
  }[];
};

// Metadata for complexity ranking (cached)
type TaskMetadata = {
  taskId: string;           // Original ARC-AGI task ID (e.g., "007bbfb7")
  numTrain: number;         // Number of training examples
  numTest: number;          // Number of test examples
  verifierLOC: number;      // Lines of code in re-arc verifier (complexity proxy)
};

type MetadataCache = {
  version: string;          // Re-arc version used
  tasks: TaskMetadata[];    // All 400 tasks, sorted by LOC descending
};
```

## High-Level Approach

### Generation Flow
1. User selects parameters: n_tasks (1-400), diff_lb, diff_ub, optional seed
2. Backend selects top n_tasks by complexity from metadata cache
3. Generate (n_tasks - 1) random IDs from seed, compute nth ID via XOR
4. Map generated IDs to selected original tasks 1-to-1
5. Call re-arc to generate examples for each task (matching train/test counts)
6. Return dataset JSON for download

### Verification Flow
1. User uploads submission JSON
2. Extract task IDs, XOR to recover seed
3. Decode optional message from sorted task IDs (if present)
4. Regenerate dataset deterministically using same seed and n_tasks
5. Compare submission attempts to regenerated ground truth
6. Score using ARC-AGI rules (ANY correct attempt = task solved)
7. Return results with per-task breakdown + decoded message

### Components

**Backend (Priority):**
- reArcController: /generate and /verify endpoints
- reArcService: orchestrate Python re-arc calls
- reArcSeed utils: XOR logic for task IDs
- Python scripts: wrap re-arc library
- arc_task_metadata.json: cached complexity rankings (version controlled)

**Frontend (Later):**
- TBD - design after backend works

## Backend Patterns (from existing codebase)

**Controllers:**
- Export functions wrapped with `asyncHandler` middleware
- Use `formatResponse` utility for consistent responses
- Import from services, don't do DB queries directly

**Services:**
- Orchestrate Python subprocess calls via `child_process.spawn`
- Handle stdout/stderr streaming for progress updates

## Implementation Steps

### Phase 1: Setup & Install
- [x] Review re-arc library documentation (https://github.com/michaelhodel/re-arc)
- [x] Understand generation parameters and API
- [x] Get user approval on key decisions (APPROVED: variable tasks 1-400, complexity filtering, LOC ranking)
- [ ] Install re-arc library (clone from GitHub)
- [ ] Understand how to extract verifier LOC from re-arc codebase
- [ ] Test re-arc generation with sample seed
- [ ] Verify XOR seed approach produces deterministic results

### Phase 1.5: Metadata Pre-processing
- [ ] Create script to extract metadata from all 400 ARC tasks
- [ ] For each task in `data/training/`:
  - [ ] Count train/test examples
  - [ ] Extract verifier function from re-arc `verifiers.py`
  - [ ] Count lines of code in verifier function
- [ ] Generate `arc_task_metadata.json` with complexity rankings
- [ ] Cache this file (one-time generation, version control it)

### Phase 2: Core Infrastructure
- [ ] Create type definitions in `shared/types.ts`
- [ ] Implement XOR seed logic in `server/utils/reArcSeed.ts`
- [ ] Create Python wrapper scripts for re-arc library
- [ ] Implement `reArcService.ts` with generation and verification methods
- [ ] Test Python integration with simple script

### Phase 3: Backend API
- [ ] Create `reArcController.ts` with generation endpoint
- [ ] Create verification endpoint
- [ ] Add routes to `server/routes.ts` (follow existing pattern)
- [ ] Test with curl/Postman
- [ ] Add error handling and validation

### Phase 4: Frontend (DEFERRED)
Design and implement UI once backend is working

## Key Technical Details

### Task ID Structure & Steganographic String Encoding
**Format:** 8 hex chars (32 bits) = upper 16 bits + lower 16 bits
- **Upper 16 bits:** Random from PRNG(seed), must be unique (collision detection)
- **Lower 16 bits:** Random from PRNG, optionally XOR'd with hidden UTF-8 message

**Steganographic encoding (optional):**
1. Generate (n-1) IDs from PRNG(seed), ensure upper bytes unique
2. Compute nth ID so `XOR(all_ids) = seed`
3. Sort all IDs by full 32-bit value
4. Save original lower bytes: `original_lower[i] = sorted_ids[i].lower`
5. XOR string into lower bytes (looks like random noise):
   - `sorted[0].lower ^= string_length`
   - `sorted[1..n-2].lower ^= string_bytes` (2 bytes per ID)
   - `sorted[n-1].lower ^= XOR(all_string_bytes)` ← compensates to preserve seed
6. Max string length: `(n_tasks - 1) * 2` bytes

**Seed recovery & decoding:**
- `XOR(all_ids) = seed` (always works - compensation maintains this)
- Decode: Regenerate PRNG sequence → get original_lower[i] → XOR back
  - `string_bytes[i] = sorted[i].lower ^ original_lower[i]`
- String is invisible without knowing the seed!

### Task Mapping
- Sort original ARC tasks by complexity (LOC descending)
- Select top n_tasks
- Map: `generated_ids[i] ↔ selected_original_tasks[i]`

### Scoring
- Task solved if ANY of 2 attempts correct (ARC-AGI rules)
- Overall score = solved_tasks / n_tasks

## Files to Create

**Backend:**
```
server/
├── controllers/reArcController.ts
├── services/reArc/
│   ├── reArcService.ts
│   ├── reArcGenerator.py
│   └── reArcVerifier.py
├── utils/reArcSeed.ts
└── routes.ts (add routes)
```

**Data:**
```
data/arc_task_metadata.json
```

**Types:**
```
shared/types.ts (add ReARC types)
```

## API Parameters

**Generation endpoint:**
- `n_tasks`: 1-400 (how many tasks)
- `diff_lb`, `diff_ub`: 0-1 (re-arc difficulty bounds)
- `seed`: optional (for reproducibility)
- `message`: optional UTF-8 string to encode in task IDs (max `(n_tasks-2)*2` bytes)

**Verification endpoint:**
- `submission`: JSON object with task IDs and attempts

**Backend dependencies:**
- `arc_task_metadata.json`: task IDs sorted by LOC
- re-arc library: generate examples matching train/test counts

