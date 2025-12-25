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
type Task = {
  train: { input: number[][]; output: number[][] }[];
  test: { input: number[][]; output?: number[][] }[];
};

type Dataset = {
  [taskId: string]: Task;
};

type Submission = {
  [taskId: string]: {
    attempt_1: number[][];
    attempt_2: number[][];
  }[];
};

type TaskMetadata = {
  numTrain: number;
  numTest: number;
  verifierLOC: number;
};

type ReArcMetadata = {
  [taskId: string]: TaskMetadata;  // Keyed by original ARC-AGI task ID
};
```

## High-Level Approach

### Generation Flow
1. User requests dataset generation (minimal API, no parameters)
2. Backend uses preset: n_tasks=400, diff_lb/diff_ub from config
3. Generate task IDs from random seed, compute last ID via XOR
4. Select top n_tasks by complexity from metadata cache
5. Map generated IDs to selected original tasks 1-to-1
6. Call re-arc to generate examples for each task (matching train/test counts)
7. Return dataset JSON for download

### Verification Flow
1. User uploads submission JSON
2. Extract task IDs, XOR to recover seed
3. Decode optional message from sorted task IDs (if present)
4. Regenerate dataset deterministically using same seed and n_tasks
5. Compare submission attempts to regenerated ground truth
6. Score using ARC-AGI rules (ANY correct attempt = task solved)
7. Return score + decoded message

### Components

**Backend (Priority):**
- reArcController: /generate and /verify endpoints
- reArcService: orchestrate Python re-arc calls
- reArcSeed utils: XOR logic and message encoding/decoding
- Python scripts: wrap re-arc library
- rearc_metadata.json: cached metadata (version controlled)

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
- [ ] Clone re-arc library from GitHub
- [ ] Review re-arc codebase structure (generators.py, verifiers.py, main.py)
- [ ] Understand how to extract verifier LOC
- [ ] Test re-arc generation with sample seed
- [ ] Verify deterministic generation

### Phase 1.5: Metadata Pre-processing
- [ ] Create script to extract metadata from all 400 ARC tasks
- [ ] For each task in `data/training/`:
  - [ ] Count train/test examples
  - [ ] Extract verifier function from re-arc `verifiers.py`
  - [ ] Count lines of code in verifier function
- [ ] Generate `data/rearc_metadata.json`
- [ ] Version control this file

### Phase 2: Core Infrastructure
- [ ] Create type definitions in `shared/types.ts`
- [ ] Implement XOR seed logic in `server/utils/reArcSeed.ts`
- [ ] Write tests for XOR seed logic (`tests/reArcSeed.test.ts`)
- [ ] Create Python wrapper scripts for re-arc library
- [ ] Implement `reArcService.ts` with generation and verification methods
- [ ] Write integration tests (`tests/reArcService.test.ts`)

### Phase 3: Backend API
- [ ] Create `reArcController.ts` with generation endpoint
- [ ] Create verification endpoint
- [ ] Add routes to `server/routes.ts` (follow existing pattern)
- [ ] Write endpoint tests (`tests/reArcController.test.ts`)
- [ ] Add error handling and validation

### Phase 4: Frontend (DEFERRED)
Design and implement UI once backend is working

## Key Technical Details

### XOR Seed Recovery & Steganographic Encoding
**Task IDs:** 8 hex chars (32 bits)
- Upper 16 bits: unique random from PRNG(seed)
- Lower 16 bits: random from PRNG, optionally XOR'd with message bytes

**Seed recovery:** `XOR(all_task_ids) = seed`

**Optional message encoding:**
- XOR raw bytes into lower 16 bits of sorted IDs
- Decode by regenerating PRNG sequence and XOR'ing back
- Looks like random noise without seed
- Max message: `(n_tasks - 1) * 2` bytes

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
data/rearc_metadata.json
```

**Tests:**
```
tests/
├── reArcSeed.test.ts          # XOR logic, message encoding
├── reArcService.test.ts       # Python integration, generation
└── reArcController.test.ts    # API endpoints
```

**Types:**
```
shared/types.ts (add ReARC types)
```

## API (Minimal)

**Generation endpoint:** `POST /api/rearc/generate`
- No parameters (uses preset configuration)
- Returns: `{ dataset: Dataset, seed: number, message?: string }`

**Verification endpoint:** `POST /api/rearc/verify`
- Body: `{ submission: Submission }`
- Returns: `{ score: number, message?: string }`

**Internal configuration:**
- `n_tasks`: 400 (hardcoded, all tasks)
- `diff_lb`, `diff_ub`: from config file
- `rearc_metadata.json`: task metadata sorted by LOC

