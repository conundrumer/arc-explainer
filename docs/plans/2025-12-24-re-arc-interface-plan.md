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
3. Regenerate dataset deterministically using same seed and n_tasks
4. Compare submission attempts to regenerated ground truth
5. Score using ARC-AGI rules (ANY correct attempt = task solved)
6. Return results with per-task breakdown

### Components Needed

**Frontend:**
- ReArcGeneratorPage: form inputs, download button
- ReArcVerifierPage: file upload, results display

**Backend:**
- reArcController: /generate and /verify endpoints
- reArcService: orchestrate Python re-arc calls
- reArcSeed utils: XOR logic for task IDs
- Python scripts: wrap re-arc library

**Data:**
- arc_task_metadata.json: cached complexity rankings (version controlled)

## Following Existing Patterns

**Controller Pattern (see `server/controllers/`):**
- Export functions wrapped with `asyncHandler` middleware
- Use `formatResponse` utility for consistent API responses
- Import from repositories/services, don't do DB queries directly

**Service Pattern (see `server/services/poetiq/`):**
- Main service file orchestrates Python subprocess calls
- Use `child_process.spawn` for Python scripts
- Handle stdout/stderr streaming for progress updates

**Frontend Patterns:**
- Use TanStack Query hooks for API calls (see existing pages)
- Reuse shadcn/ui components (Card, Button, Table, etc.)
- Follow PageLayout wrapper pattern for consistent styling
- Use Wouter for routing (`<Route path="/rearc/..." component={...} />`)

**Navigation (AppNavigation.tsx):**
- Add dropdown menu item or direct link
- Include icon, title, description
- Follow discriminated union pattern (NavLink | NavDropdown)

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

### Phase 4: Frontend - Generator
- [ ] Create `ReArcGeneratorPage.tsx` using PageLayout
- [ ] Add form controls for generation parameters (use shadcn/ui)
- [ ] Create DatasetDownloadButton component
- [ ] Add route to `App.tsx`
- [ ] Test generation flow end-to-end

### Phase 5: Frontend - Verifier
- [ ] Create `ReArcVerifierPage.tsx`
- [ ] Create SubmissionUploader component (file input)
- [ ] Create VerificationResults component (table/cards)
- [ ] Add visual grid comparison (reuse existing ARC grid rendering)
- [ ] Add route to `App.tsx`
- [ ] Test verification flow end-to-end

### Phase 6: Navigation & Polish
- [ ] Add nav items to AppNavigation.tsx (decide: dropdown or direct links)
- [ ] Error handling for invalid submissions
- [ ] Loading states during generation/verification (use shadcn/ui Skeleton)
- [ ] Validation of submission format
- [ ] User documentation/help text
- [ ] Git commit with detailed message

## Key Technical Details

### XOR Seed Recovery
- Generate (n_tasks - 1) random 8-char hex IDs from seed
- Compute nth ID = XOR of all previous IDs plus seed
- Verification: XOR all n IDs recovers original seed
- Math: `id[0] ^ ... ^ id[n-2] ^ (id[0] ^ ... ^ id[n-2] ^ seed) = seed`

### Task Mapping
- Sort original ARC tasks by complexity (LOC descending)
- Select top n_tasks
- Map: `generated_ids[i] ↔ selected_original_tasks[i]`

### Scoring
- Task solved if ANY of 2 attempts correct (ARC-AGI rules)
- Overall score = solved_tasks / n_tasks

## Integration with Existing Webapp

### Files to Create/Modify

**Backend:**
```
server/
├── controllers/
│   └── reArcController.ts          # NEW - API endpoints
├── services/
│   └── reArc/
│       ├── reArcService.ts         # NEW - Main service
│       ├── reArcGenerator.py       # NEW - Python generation script
│       └── reArcVerifier.py        # NEW - Python verification script
├── utils/
│   └── reArcSeed.ts                # NEW - Seed XOR logic
└── routes.ts                        # MODIFY - Add re-arc routes
```

**Frontend:**
```
client/src/
├── pages/
│   ├── ReArcGeneratorPage.tsx      # NEW - Generation UI
│   └── ReArcVerifierPage.tsx       # NEW - Verification UI
├── components/
│   ├── reArc/
│   │   ├── DatasetDownloadButton.tsx   # NEW
│   │   ├── SubmissionUploader.tsx      # NEW
│   │   └── VerificationResults.tsx     # NEW
│   └── layout/
│       └── AppNavigation.tsx        # MODIFY - Add nav items
└── App.tsx                          # MODIFY - Add routes
```

**Shared:**
```
shared/
└── types.ts                         # MODIFY - Add ReARC types
```

## Parameters

**User-facing:**
- `n_tasks`: 1-400 (how many tasks to generate)
- `diff_lb`, `diff_ub`: 0-1 (re-arc difficulty bounds)
- `seed`: optional (for reproducibility)

**Backend:**
- Metadata cache: `arc_task_metadata.json` (task IDs sorted by LOC)
- Re-arc library: generate examples per task with matching train/test counts

**Navigation:**
- Add to "Misc" dropdown menu

## User Flows

**Generate:** Select n_tasks + difficulty → Download challenges.json
**Verify:** Upload submission.json → See score + per-task results

